# HTTP Get静态资源请求功能编写

HTTP协议处理真是麻烦，早知道找个现成HTTP框架，只编写Servlet相关部分……

先把HTTP处理流程定下来，根据前面定下来的框架，需要继承AbstractHttpEventHandler，在doHandle方法中编写接受请求内容、生成响应内容，并输出到客户端的代码，模板类：

```java
public abstract class AbstractHttpEventHandler extends AbstractEventHandler<Connection> {
    @Override
    protected void doHandle(Connection connection) {
        //从输入中构造出HTTP请求对象,Body的内容是延迟读取
        HttpRequestMessage requestMessage = doParserRequestMessage(connection);
        //构造HTTP响应对象
        HttpResponseMessage responseMessage = doGenerateResponseMessage(requestMessage);
        try {
            //输出响应到客户端
            doTransferToClient(responseMessage, connection);
        } catch (IOException e) {
            throw new HandlerException(e);
        } finally {
            //完成响应后，关闭Socket
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



