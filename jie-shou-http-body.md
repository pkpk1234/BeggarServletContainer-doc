# 接收HTTP Body

解析请求中的Body需要注意Transfer-Encoding头。

当Transfer-Encoding的值为chunked时，不应该使用Content-Length去读取Body。

chunked的Body格式为：

> 数据以一系列分块的形式进行发送。
>
> [`Content-Length`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Length)
>
> 首部在这种情况下不被发送。。在每一个分块的开头需要添加当前分块的长度，以十六进制的形式表示，后面紧跟着 '
>
> `\r\n`
>
> ' ，之后是分块本身，后面也是'
>
> `\r\n`
>
> ' 。终止块是一个常规的分块，不同之处在于其长度为0。终止块后面是一个挂载（trailer），由一系列（或者为空）的实体消息首部构成

[https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Transfer-Encoding](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Transfer-Encoding)

所以chunked的Body使用 0\r\n\r\n 作为Body的结束标志。

由此可以构造延迟读取Body内容的流：

```java
public class HttpBodyInputStream extends InputStream {
    //chunked body使用 0\r\n\r\n结尾
    private static final char[] TAILER_CHARS = {'0', '\r', '\n', '\r', '\n'};
    private final InputStream inputStream;
    private long contentLength;
    private boolean isChuncked;
    private long readed = 0;
    private boolean isFinished = false;
    private int idx = 0;

    public HttpBodyInputStream(InputStream inputStream, boolean ifChuncked) {
        this.inputStream = inputStream;
        this.isChuncked = ifChuncked;
    }

    @Override
    public int read() throws IOException {
        if (isChuncked) {
            return readChunked();
        } else {
            return readByte();
        }
    }

    /**
     * 读取chunked body
     *
     * @return
     * @throws IOException
     */
    private int readChunked() throws IOException {
        if (isFinished) {
            return -1;
        }
        int i = this.inputStream.read();
        if (i == -1) {
            return i;
        } else {
            if (idx == TAILER_CHARS.length - 1) {
                isFinished = true;
            } else if (TAILER_CHARS[idx++] != (char) i) {
                idx = 0;
            }
        }
        return i;
    }

    /**
     * 读取非chunked body
     *
     * @return
     * @throws IOException
     */
    private int readByte() throws IOException {
        readed++;
        if (readed > contentLength) {
            return -1;
        } else {
            return this.inputStream.read();
        }
    }

    public long getContentLength() {
        return contentLength;
    }

    public void setContentLength(long contentLength) {
        this.contentLength = contentLength;
    }
}
```

此处BodyParser并不处理压缩和编码，只是直接返回raw body，使用处再根据实际情况进行处理。

单元测试：

```java
public class TestHttpBodyInputStream {
    private static final Logger LOGGER =
            LoggerFactory.getLogger(TestHttpBodyInputStream.class);
    /*
        7
        Mozilla
        9
        Developer
        7
        Network
        0

     */
    private static final String TEST_OTHER_MESSAGE = "testtest";
    private static final String CHUNKED_BODY = "7\r\n" +
            "Mozilla\r\n" +
            "9\r\n" +
            "Developer\r\n" +
            "7\r\n" +
            "Network\r\n" +
            "0\r\n" +
            "\r\n";

    private static final String BODY = CHUNKED_BODY + TEST_OTHER_MESSAGE;

    private static final String NORMAL_BODY = "Mozilla Developer Network";

    @Test
    public void testReadChunkedBody() throws IOException {
        InputStream in = new ByteArrayInputStream(BODY.getBytes());
        HttpBodyInputStream httpBodyInputStream = new HttpBodyInputStream(in, true);
        ByteOutputStream out = new ByteOutputStream();
        IOUtils.copy(httpBodyInputStream, out);
        LOGGER.info(out.toString());
        assertEquals(CHUNKED_BODY, out.toString());
    }

    @Test
    public void testReadBody() throws IOException {
        InputStream in = new ByteArrayInputStream(NORMAL_BODY.getBytes());
        HttpBodyInputStream httpBodyInputStream = new HttpBodyInputStream(in, false);
        httpBodyInputStream.setContentLength(25);
        ByteOutputStream out = new ByteOutputStream();
        IOUtils.copy(httpBodyInputStream, out);
        LOGGER.info(out.toString());
        assertEquals(NORMAL_BODY, out.toString());
    }
}
```

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step10

