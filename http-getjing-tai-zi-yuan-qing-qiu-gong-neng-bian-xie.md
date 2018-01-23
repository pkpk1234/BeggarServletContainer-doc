# HTTP Get静态资源请求功能编写

HTTP协议处理真是麻烦，早知道找个现成HTTP框架，只编写Servlet相关部分……

## 总体流程

先把HTTP处理流程定下来，根据前面定下来的框架，需要继承AbstractHttpEventHandler，在doHandle方法中编写接受请求内容、生成响应内容，并输出到客户端的代码，模板类：

```java
public abstract class AbstractHttpEventHandler extends AbstractEventHandler<Connection> {
    @Override
    protected void doHandle(Connection connection) {
        //1.从输入中构造出HTTP请求对象,Body的内容是延迟读取
        HttpRequestMessage requestMessage = doParserRequestMessage(connection);
        //2.构造HTTP响应对象
        HttpResponseMessage responseMessage = doGenerateResponseMessage(requestMessage);
        try {
            //3.输出响应到客户端
            doTransferToClient(responseMessage, connection);
        } catch (IOException e) {
            throw new HandlerException(e);
        } finally {
            //4.完成响应后，关闭Socket
            if (connection instanceof SocketConnection) {
                IOUtils.closeQuietly(((SocketConnection) connection).getSocket());
            }
        }

    }

    /**
     * 通过输入构造HttpRequestMessage
     *
     * @param connection
     * @return
     */
    protected abstract HttpRequestMessage doParserRequestMessage(Connection connection);

    /**
     * 根据HttpRequestMessage生成HttpResponseMessage
     *
     * @param httpRequestMessage
     * @return
     */
    protected abstract HttpResponseMessage doGenerateResponseMessage(
            HttpRequestMessage httpRequestMessage);

    /**
     * 写入HttpResponseMessage到客户端
     *
     * @param responseMessage
     * @param connection
     * @throws IOException
     */
    protected abstract void doTransferToClient(HttpResponseMessage responseMessage,
                                               Connection connection) throws IOException;
}
```

## 构造出HTTP请求对象

还是先定下处理流程，首先构造RequestLine、然后构造QueryParameter和Headers，如果有Body，则构造Body。

整个过程，需要多个parser参与，并且有parser的输出是另外的parser的输入，所有这里通过ThreadLocal来保存这些在多个parser之间共享的变量，保存在HttpParserContext中。

```java
public abstract class AbstractHttpRequestMessageParser extends AbstractParser implements HttpRequestMessageParser {
    private static final Logger LOGGER = LoggerFactory.getLogger(AbstractHttpRequestMessageParser.class);

    /**
     * 定义parse流程
     *
     * @return
     */
    @Override
    public HttpRequestMessage parse(InputStream inputStream) throws IOException {
        //1.设置上下文:设置是否有body、body之前byte数组，以及body之前byte数组长度到上下文中
        getAndSetBytesBeforeBodyToContext(inputStream);
        //2.解析构造RequestLine
        RequestLine requestLine = parseRequestLine();
        //3.解析构造QueryParameters
        HttpQueryParameters httpQueryParameters = parseHttpQueryParameters();
        //4.解析构造HTTP请求头
        IMessageHeaders messageHeaders = parseRequestHeaders();
        //5.解析构造HTTP Body，如果有个的话
        Optional<HttpBody> httpBody = parseRequestBody();

        HttpRequestMessage httpRequestMessage = new HttpRequestMessage(requestLine, messageHeaders, httpBody, httpQueryParameters);
        return httpRequestMessage;
    }

    /**
     * 读取请求发送的数据，并保存为byte数组设置到解析上下文中
     *
     * @param inputStream
     * @throws IOException
     */
    private void getAndSetBytesBeforeBodyToContext(InputStream inputStream) throws IOException {
        byte[] bytes = copyRequestBytesBeforeBody(inputStream);
        HttpParserContext.setHttpMessageBytes(bytes);
        HttpParserContext.setBytesLengthBeforeBody(bytes.length);
    }

    /**
     * 解析并构建RequestLine
     *
     * @return
     */
    protected abstract RequestLine parseRequestLine();

    /**
     * 解析并构建HTTP请求Headers集合
     *
     * @return
     */
    protected abstract IMessageHeaders parseRequestHeaders();

    /**
     * 解析并构建HTTP 请求Body
     *
     * @return
     */
    protected abstract Optional<HttpBody> parseRequestBody();

    /**
     * 解析并构建QueryParameter集合
     *
     * @return
     */
    protected abstract HttpQueryParameters parseHttpQueryParameters();

    /**
     * 构造body(如果有)之前的字节数组
     * @param inputStream
     * @return
     * @throws IOException
     */
    private byte[] copyRequestBytesBeforeBody(InputStream inputStream) throws IOException {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream(inputStream.available());
        int i = -1;
        byte[] temp = new byte[3];
        while ((i = inputStream.read()) != -1) {
            byteArrayOutputStream.write(i);
            if ((char) i == '\r') {
                int len = inputStream.read(temp, 0, temp.length);
                byteArrayOutputStream.write(temp, 0, len);
                if ("\n\r\n".equals(new String(temp))) {
                    break;
                }
            }
        }
        return byteArrayOutputStream.toByteArray();
    }
}
```



