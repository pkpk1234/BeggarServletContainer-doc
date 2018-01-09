# NIOConnector

现在为Server添加NIOConnector，添加之前可以发现我们的代码其实是有问题的。比如现在的代码是无法让服务器支持同时监听多个端口和IP的，如同时监听 127.0.0.1:18080和0.0.0.0:18443现在是无法做到的。因为当期的端口号是Server的属性，并且只有一个，但是端口其实应该是Connector的属性，因为Connector专门负责了Server的IO。

重构一下，将端口号从Server中去掉，取而代之的是Connector列表；将当期的Connector抽象类重命名为AbstractConnector，再新建接口Connector，添加getPort和getHost两个方法，让Connector支持将监听绑定到不同IP的功能。

去掉getPort方法

```java
public interface Server {
    /**
     * 启动服务器
     */
    void start() throws IOException;

    /**
     * 关闭服务器
     */
    void stop();

    /**
     * 获取服务器启停状态
     * @return
     */
    ServerStatus getStatus();

    /**
     * 获取服务器管理的Connector列表
     * @return
     */
    List<Connector> getConnectorList();
}
```

去掉port属性和方法

```java
public class SimpleServer implements Server {
    private static Logger logger = LoggerFactory.getLogger(SimpleServer.class);
    private volatile ServerStatus serverStatus = ServerStatus.STOPED;
    private final List<AbstractConnector> connectorList;

    public SimpleServer(List<AbstractConnector> connectorList) {
        this.connectorList = connectorList;
    }
    ... ...
}
```

添加HOST属性绑定IP，添加backLog属性设置ServerSocket的TCP属性SO\_BACKLOG。修改init方法，支持ServerSocket绑定IP。

```java
public class SocketConnector extends AbstractConnector<Socket> {
    private static final Logger LOGGER = LoggerFactory.getLogger(SocketConnector.class);
    private static final String LOCALHOST = "localhost";
    private static final int DEFAULT_BACKLOG = 50;
    private final int port;
    private final String host;
    private final int backLog;
    private ServerSocket serverSocket;
    private volatile boolean started = false;
    private final EventListener<Socket> eventListener;

    public SocketConnector(int port, EventListener<Socket> eventListener) {
        this(port, LOCALHOST, DEFAULT_BACKLOG, eventListener);
    }

    public SocketConnector(int port, String host, int backLog, EventListener<Socket> eventListener) {
        this.port = port;
        this.host = StringUtils.isBlank(host) ? LOCALHOST : host;
        this.backLog = backLog;
        this.eventListener = eventListener;
    }


    @Override
    protected void init() throws ConnectorException {

        //监听本地端口，如果监听不成功，抛出异常
        try {
            InetAddress inetAddress = InetAddress.getByName(this.host);
            this.serverSocket = new ServerSocket(this.port, backLog, inetAddress);
            this.started = true;
        } catch (IOException e) {
            throw new ConnectorException(e);
        }
    }
```

执行单元测试，一切OK。现在可以开始添加NIO了。

根据前面一步一步搭建的架构，需要添加支持NIO的EventListener和EventHandler两个实现即可。

NIOEventListener中莫名其妙出现了SelectionKey，表面这个类和SelectionKey是强耦合的，说明Event这块的架构设计是很烂的，势必又要重构，今天先不改了，完成功能先。

```java
public class NIOEventListener extends AbstractEventListener<SelectionKey> {
    private final EventHandler<SelectionKey> eventHandler;

    public NIOEventListener(EventHandler<SelectionKey> eventHandler) {
        this.eventHandler = eventHandler;
    }

    @Override
    protected EventHandler<SelectionKey> getEventHandler(SelectionKey event) {
        return this.eventHandler;
    }
}
```

同意的道理，NIOEchoEventHandler也不应该和SelectionKey强耦合，echo功能简单，如果是返回文件内容的功能，那样的话，大段大段的文件读写代码是完全无法复用的。

```java
public class NIOEchoEventHandler extends AbstractEventHandler<SelectionKey> {
    @Override
    protected void doHandle(SelectionKey key) {
        try {
            if (key.isReadable()) {
                SocketChannel client = (SocketChannel) key.channel();
                ByteBuffer output = (ByteBuffer) key.attachment();
                client.read(output);
            } else if (key.isWritable()) {
                SocketChannel client = (SocketChannel) key.channel();
                ByteBuffer output = (ByteBuffer) key.attachment();
                output.flip();
                client.write(output);
                output.compact();
            }
        } catch (IOException e) {
            throw new HandlerException(e);
        }
    }
}
```

修改ServerFactory，添加NIO功能，这里的代码也是有很大设计缺陷的，ServerFactory只应该根据传入的config信息构造Server，而不是每次都去改工厂。

```java
public class ServerFactory {
    /**
     * 返回Server实例
     *
     * @return
     */
    public static Server getServer(ServerConfig serverConfig) {
        List<Connector> connectorList = new ArrayList<>();
        SocketEventListener socketEventListener =
                new SocketEventListener(new FileEventHandler(System.getProperty("user.dir")));
        ConnectorFactory connectorFactory =
                new SocketConnectorFactory(new SocketConnectorConfig(serverConfig.getPort()), socketEventListener);
        //NIO
        NIOEventListener nioEventListener = new NIOEventListener(new NIOEchoEventHandler());
        //监听18081端口
        SocketChannelConnector socketChannelConnector = new SocketChannelConnector(18081,nioEventListener);

        connectorList.add(connectorFactory.getConnector());
        connectorList.add(socketChannelConnector);
        return new SimpleServer(connectorList);
    }
}
```

运行BootStrap，启动Server，telnet访问18081端口，功能是勉强实现了，但是架构设计是有重大缺陷的，进一步添加功能之前，需要重构好架构才行。

![](/assets/nio-echo-server.jpg)

完整代码：[https://github.com/pkpk1234/BeggarServletContainer/tree/step6](https://github.com/pkpk1234/BeggarServletContainer/tree/step6)

分支step6

![](/assets/git-br-step6.jpg)

