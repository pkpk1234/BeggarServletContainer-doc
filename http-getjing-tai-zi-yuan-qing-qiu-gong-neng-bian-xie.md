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

这里通过copyRequestBytesBeforeBody方法确定是否具有body，同时将body之前字节都保存起来。

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

RequestLine解析时会设置Http请求方法和queryString到HttpParserContext中，作为QueryParameter解析的输入。

```java
    @Override
    protected RequestLine parseRequestLine() {
        RequestLine requestLine = this.httpRequestLineParser.parse();
        HttpParserContext.setHttpMethod(requestLine.getMethod());
        HttpParserContext.setRequestQueryString(requestLine.getRequestURI().getQuery());
        return requestLine;
    }
```

QueryParameter的解析很简单，直接调用HttpQueryParameterParser即可。

HttpHeader解析流程为从HttpParserContext中取出请求报文，去掉第一行，然后逐行处理，构造成key-value的形式保存起来。

同时，通过Content-Length头或者Transfer-Encoding判断请求是否包含Body。

```java
public class DefaultHttpHeaderParser extends AbstractParser implements HttpHeaderParser {
    private static final String SPLITTER = ":";
    private static final Logger LOGGER = LoggerFactory.getLogger(DefaultHttpHeaderParser.class);

    @Override
    public HttpMessageHeaders parse() {
        try {
            String httpText = getHttpTextFromContext();
            HttpMessageHeaders httpMessageHeaders = doParseHttpMessageHeaders(httpText);
            setHasBody(httpMessageHeaders);
            return httpMessageHeaders;
        } catch (UnsupportedEncodingException e) {
            throw new ParserException("Unsupported Encoding", e);
        }
    }

    /**
     * 从上下文获取bytes并转换为String
     *
     * @return
     * @throws UnsupportedEncodingException
     */
    private String getHttpTextFromContext() throws UnsupportedEncodingException {
        byte[] bytes = HttpParserContext.getHttpMessageBytes();
        return new String(bytes, "utf-8");
    }

    /**
     * 解析Body之前的文本构建HttpHeader，并保存到HttpMessageHeaders中
     *
     * @param httpText
     * @return
     */
    private HttpMessageHeaders doParseHttpMessageHeaders(String httpText) {
        HttpMessageHeaders httpMessageHeaders = new HttpMessageHeaders();
        String[] lines = httpText.split(CRLF);
        //跳过第一行
        for (int i = 1; i < lines.length; i++) {
            String keyValue = lines[i];
            if ("".equals(keyValue)) {
                break;
            }
            String[] temp = keyValue.split(SPLITTER);
            if (temp.length == 2) {
                httpMessageHeaders.addHeader(new HttpHeader(temp[0], temp[1].trim()));
            }
        }
        return httpMessageHeaders;
    }

    /**
     * 设置报文是否包含Body到上下文中
     */
    private void setHasBody(HttpMessageHeaders httpMessageHeaders) {
        if (httpMessageHeaders.hasHeader("Content-Length")
                || (httpMessageHeaders.getFirstHeader("Transfer-Encoding") != null
                    && "chunked".equals(httpMessageHeaders.getFirstHeader("Transfer-Encoding").getValue())))
        {
            HttpParserContext.setHasBody(true);
        }
    }
}
```

如果请求包含了Body，就从InputStream中读取Content-Length长度的内容作为Body内容。Transfer-Encoding的暂时没处理，后续再加。

```java
public class DefaultHttpBodyParser implements HttpBodyParser {
    @Override
    public HttpBody parse() {
        int contentLength = HttpParserContext.getBodyInfo().getContentLength();
        InputStream inputStream = HttpParserContext.getInputStream();
        try {
            byte[] body = IOUtils.readFully(inputStream, contentLength);
            String contentType = HttpParserContext.getContentType();
            String encoding = getEncoding(contentType);
            HttpBody httpBody =
                    new HttpBody(contentType, encoding, body);
            return httpBody;
        } catch (IOException e) {
            throw new ParserException(e);
        }
    }

    /**
     * 获取encoding
     * 例如：Content-type: application/json; charset=utf-8
     *
     * @param contentType
     * @return
     */
    private String getEncoding(String contentType) {
        String encoding = "utf-8";
        if (StringUtils.isNotBlank(contentType) && contentType.contains(";")) {
            encoding = contentType.split(";")[1].trim().replace("charset=", "");
        }
        return encoding;
    }
}
```

到此为止HTTP请求对象构造完毕。

## 编写静态资源Handler

在AbstractHttpEventHandler的基础上，添加静态资源返回功能。

总体流程为：从HTTP请求对象中获取到请求路径，在docBase目录下查找对应路径的资源并返回。

需要注意的是不同类型文件的Content-Type响应头需要设置正确，否则文件不会正确显示。

```java
public class HttpStaticResourceEventHandler extends AbstractHttpEventHandler {

    private final String docBase;
    private final AbstractHttpRequestMessageParser httpRequestMessageParser;

    public HttpStaticResourceEventHandler(String docBase,
                                          AbstractHttpRequestMessageParser httpRequestMessageParser) {
        this.docBase = docBase;
        this.httpRequestMessageParser = httpRequestMessageParser;
    }

    @Override
    protected HttpRequestMessage doParserRequestMessage(Connection connection) {
        try {
            HttpRequestMessage httpRequestMessage = httpRequestMessageParser
                    .parse(connection.getInputStream());
            return httpRequestMessage;
        } catch (IOException e) {
            throw new HandlerException(e);
        }
    }

    @Override
    protected HttpResponseMessage doGenerateResponseMessage(
            HttpRequestMessage httpRequestMessage) {
        String path = httpRequestMessage.getRequestLine().getRequestURI().getPath();
        Path filePath = Paths.get(docBase, path);
        //目录、无法读取的文件都返回404
        if (Files.isDirectory(filePath) || !Files.isReadable(filePath)) {
            return HttpResponseConstants.HTTP_404;
        } else {
            ResponseLine ok = ResponseLineConstants.RES_200;
            HttpMessageHeaders headers = HttpMessageHeaders.newBuilder()
                    .addHeader("status", "200").build();
            HttpBody httpBody = null;
            try {
                setContentType(filePath, headers);
                httpBody = new HttpBody(new FileInputStream(filePath.toFile()));
            } catch (FileNotFoundException e) {
                return HttpResponseConstants.HTTP_404;
            } catch (IOException e) {
                throw new HandlerException(e);
            }
            HttpResponseMessage httpResponseMessage = new HttpResponseMessage(ok, headers,
                    Optional.ofNullable(httpBody));
            return httpResponseMessage;
        }

    }

    /**
     * 根据文件后缀设置文件Content-Type
     *
     * @param filePath
     * @param headers
     * @throws IOException
     */
    private void setContentType(Path filePath, HttpMessageHeaders headers) throws IOException {
        //使用Files.probeContentType在mac上总是返回null
        //String contentType = Files.probeContentType(filePath);
        String contentType = MimetypesFileTypeMap.getDefaultFileTypeMap().getContentType(filePath.toString());
        headers.addHeader(new HttpHeader("Content-Type", contentType));
        if (contentType.indexOf("text") == -1) {
            headers.addHeader(new HttpHeader("Content-Length",
                    String.valueOf(filePath.toFile().length())));
        }
    }

    @Override
    protected void doTransferToClient(HttpResponseMessage responseMessage,
                                      Connection connection) throws IOException {
        HttpResponseMessageWriter httpResponseMessageWriter = new HttpResponseMessageWriter();
        httpResponseMessageWriter.write(responseMessage, connection);
    }

}
```

## 启动服务器测试

修改BootStrap，添加对应功能

```java
EventListener<Connection> socketEventListener3 =
                new ConnectionEventListener(
                    new HttpStaticResourceEventHandler(System.getProperty("user.dir"),
                        new DefaultHttpRequestMessageParser(new DefaultHttpRequestLineParser(),
                                new DefaultHttpQueryParameterParser(),
                                new DefaultHttpHeaderParser(),
                                new DefaultHttpBodyParser())));
SocketConnector connector3 =
                SocketConnectorFactory.build(18083, socketEventListener3);
ServerConfig serverConfig = ServerConfig.builder()
                .addConnector(connector3)
                .build();
        Server server = ServerFactory.getServer(serverConfig);
        server.start();
```

新建web目录，添加html、图片和js

![](/assets/web-dirs.jpg)

index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hello Beggar Servlet Container</title>
    <link type="text/css" rel="stylesheet" href="main.css" />
</head>
<body>
<img src="spring.jpg">
<div id="content"></div>
<script src="index.js"></script>
</body>
</html>
```

index.js

```js
var date = new Date();
var content = "now is " + date;
var div = document.querySelector("#content");
div.innerHTML = content;
```

main.css

```js
body {
    background-color: antiquewhite;
}

#content {
    background-color: cornflowerblue;
}
```

本地测试：

用浏览器访问[http://localhost:18083/web/index.html](http://localhost:18083/web/index.html)

显示如下

![](/assets/web.jpg)

完整代码：[https://github.com/pkpk1234/BeggarServletContainer/tree/step9](https://github.com/pkpk1234/BeggarServletContainer/tree/step9)

//TODO: 整理写死的字符串

