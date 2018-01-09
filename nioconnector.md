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



