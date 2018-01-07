# EventListener接口

让我们继续看SocketConnector中的acceptConnect方法：

```java
@Override
    protected void acceptConnect() throws ConnectorException {
        new Thread(() -> {
            while (true && started) {
                Socket socket = null;
                try {
                    socket = serverSocket.accept();
                    LOGGER.info("新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
                } catch (IOException e) {
                    //单个Socket异常，不要影响整个Connector
                    LOGGER.error(e.getMessage(), e);
                } finally {
                    IoUtils.closeQuietly(socket);
                }
            }
        }).start();
    }
```

注意socket = serverSocket.accept\(\)，这里获取到socket之后只是打印日志，并没获取socket的输入输出进行操作。

操作socket的输入和输出是否应该在SocketConnector中？这时大师又说话了，Connector责任是啥，就是管理connect的啊，connect怎么使用，关它屁事。再看那个无限循环，像不像再等待事件来临啊，成功accept一个socket就是一个事件，对scoket的使用，其实就是事件响应嘛。

OK，让我们按照这个思路来重构一下，目的就是加入事件机制，并将对具体实现的依赖控制在那几个工厂类里面去。

新增接口EventListener接口进行事件监听

```java
public interface EventListener<T> {
    /**
     * 事件发生时的回调方法
     * @param event 事件对象
     * @throws EventException 处理事件时异常都转换为该异常抛出
     */
    void onEvent(T event) throws EventException;
}
```

为Socket事件实现一下，acceptConnect中打印日志的语句可以移动到这来

```java
public class SocketEventListener implements EventListener<Socket> {
    private static final Logger LOGGER = LoggerFactory.getLogger(SocketEventListener.class);

    @Override
    public void onEvent(Socket socket) throws EventException {
        LOGGER.info("新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
    }
```

重构Connector，添加事件机制，注意whenAccept方法调用了eventListener

```java
public class SocketConnector extends Connector<Socket> {
    ... ...
    private final EventListener<Socket> eventListener;

    public SocketConnector(int port, EventListener<Socket> eventListener) {
        this.port = port;
        this.eventListener = eventListener;
    }

    @Override
    protected void acceptConnect() throws ConnectorException {
        new Thread(() -> {
            while (true && started) {
                Socket socket = null;
                try {
                    socket = serverSocket.accept();
                    whenAccept(socket);
                } catch (Exception e) {
                    //单个Socket异常，不要影响整个Connector
                    LOGGER.error(e.getMessage(), e);
                } finally {
                    IoUtils.closeQuietly(socket);
                }
            }
        }).start();
    }

    @Override
    protected void whenAccept(Socket socketConnect) throws ConnectorException {
        eventListener.onEvent(socketConnect);
    }
    ... ...
}
```

重构ServerFactory，添加对具体实现的依赖

```java
public class ServerFactory {

    public static Server getServer(ServerConfig serverConfig) {
        List<Connector> connectorList = new ArrayList<>();
        SocketEventListener socketEventListener = new SocketEventListener();
        ConnectorFactory connectorFactory =
                new SocketConnectorFactory(new SocketConnectorConfig(serverConfig.getPort()), socketEventListener);
        connectorList.add(connectorFactory.getConnector());
        return new SimpleServer(serverConfig, connectorList);
    }
}
```

再运行所有单元测试，一切都OK。

现在让我们来操作socket，实现一个echo功能的server吧。

直接添加到SocketEventListener中

```java
public class SocketEventListener implements EventListener<Socket> {
    private static final Logger LOGGER = LoggerFactory.getLogger(SocketEventListener.class);

    @Override
    public void onEvent(Socket socket) throws EventException {
        LOGGER.info("新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
        try {
            echo(socket);
        } catch (IOException e) {
            throw new EventException(e);
        }
    }

    private void echo(Socket socket) throws IOException {
        InputStream inputstream = null;
        OutputStream outputStream = null;
        try {
            inputstream = socket.getInputStream();
            outputStream = socket.getOutputStream();
            Scanner scanner = new Scanner(inputstream);
            PrintWriter printWriter = new PrintWriter(outputStream);
            printWriter.append("Server connected.Welcome to echo.\n");
            printWriter.flush();
            while (scanner.hasNextLine()) {
                String line = scanner.nextLine();
                if (line.equals("stop")) {
                    printWriter.append("bye bye.\n");
                    printWriter.flush();
                    break;
                } else {
                    printWriter.append(line);
                    printWriter.append("\n");
                    printWriter.flush();
                }
            }
        } finally {
            IoUtils.closeQuietly(inputstream);
            IoUtils.closeQuietly(outputStream);
        }
    }
}
```

之前都是在单元测试里面启动Server的，这次需要启动Server后，用telnet去使用echo功能，所以再为Server编写一个启动类，在其main方法里面启动Server

```java
public class BootStrap {
    public static void main(String[] args) throws IOException {
        ServerConfig serverConfig = new ServerConfig();
        Server server = ServerFactory.getServer(serverConfig);
        server.start();
    }
}
```

服务器启动后，使用telnet进行验证，如下：

![](/assets/test-echo.jpg)

到现在为止，我们的服务器终于有了实际功能，下一步终于可以去实现请求静态资源的功能了。

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step4

分支step4

![](/assets/git-br-step4.jpg)

