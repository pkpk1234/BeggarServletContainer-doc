# Server接口编写

开发环境搭建好了，可以开始写代码了。

但是应该怎么写呢，完全没头绪。还是从核心基本功能入手，Servlet容器，说白了就是一HTTP服务器嘛，能支持Servlet。

别人要用你的服务器，总是需要启动的，用完了，是需要关闭的。

那么先定义一个Server，用来表示服务器，提供启动和停止功能。

大师们总说面向接口编程，那么先定义一个Server接口：

```java
public interface Server {
    /**
     * 启动服务器
     */
    void start();

    /**
     * 关闭服务器
     */
    void stop();
}
```

大师们还说，要多测试，所以再添加一个单元测试类。但是现在只有接口，没实现，没法测，没关系，先写个输出字符串到标准输出的实现再说。

```java
public class SimpleServer implements Server {
    @Override
    public void start() {
        System.out.println("Server start");
    }

    @Override
    public void stop() {
        System.out.println("Server stop");
    }
}
```

有了这个实现，就可以写出单元测试了。

```java
public class TestServer {
    private static final Server SERVER = new SimpleServer();

    @BeforeClass

    @Test
    public void testServerStart() {
        SERVER.start();
    }

    @Test
    public void testServerStop() {
        SERVER.stop();
    }
}
```

先不管这个SimpleServer啥用没有，看看上面的单元测试，里面出现了具体实现SimpleServer，大师很不高兴，如果编写了ComplicatedServer，这里代码岂不是要改。重构一下，添加一个工厂类。

```java
public class ServerFactory {
    /**
     * 返回Server实例
     * @return
     */
    public static Server getServer() {
        return new SimpleServer();
    }
}
```

单元测试重构后如下：这样就将接口和具体实现隔离开了，代码更加灵活。

```java
public class TestServer {
    private static final Server SERVER = ServerFactory.getServer();

    @BeforeClass

    @Test
    public void testServerStart() {
        SERVER.start();
    }

    @Test
    public void testServerStop() {
        SERVER.stop();
    }
}
```

再看单元测试，没法写assert断言啊，难道要用客户端请求下才知道Server的启停状态？Server要自己提供状态查询接口。

重构Server，添加getStatus接口，返回Server状态，状态应是枚举，暂定STARTED、STOPED两种，只有调用了start方法后，状态才会变为STARTED。

Server重构后如下：

```java
public class SimpleServer implements Server {

    private ServerStatus serverStatus = ServerStatus.STOPED;

    @Override
    public void start() {
        this.serverStatus = ServerStatus.STARTED;
        System.out.println("Server start");
    }

    @Override
    public void stop() {
        this.serverStatus = ServerStatus.STOPED;
        System.out.println("Server stop");
    }

    @Override
    public ServerStatus getStatus() {
        return serverStatus;
    }

}
```

再为单元测试添加断言：

```java
public class TestServer {
    private static final Server SERVER = ServerFactory.getServer();

    @Test
    public void testServerStart() {
        SERVER.start();
        assertTrue("服务器启动后，状态是STARTED",SERVER.getStatus().equals(ServerStatus.STARTED));
    }

    @Test
    public void testServerStop() {
        SERVER.stop();
        assertTrue("服务器关闭后，状态是STOPED",SERVER.getStatus().equals(ServerStatus.STOPED));
    }
}
```

再继续看Server接口，要接受客户端的请求，需要监听本地端口，端口应该作为构造参数传入，并且Server应该具有默认的端口。再继续重构。

```java
public class SimpleServer implements Server {

    private ServerStatus serverStatus = ServerStatus.STOPED;
    public final int DEFAULT_PORT = 18080;
    private final int PORT;

    public SimpleServer(int PORT) {
        this.PORT = PORT;
    }

    public SimpleServer() {
        this.PORT = DEFAULT_PORT;
    }

    @Override
    public void start() {
        this.serverStatus = ServerStatus.STARTED;
        System.out.println("Server start");
    }

    @Override
    public void stop() {
        this.serverStatus = ServerStatus.STOPED;
        System.out.println("Server stop");
    }

    @Override
    public ServerStatus getStatus() {
        return serverStatus;
    }

    public int getPORT() {
        return PORT;
    }
}
```

问题又来了，ServerFactory没法传端口，最简单的方法是修改ServerFactory.getServer\(\)方法，增加一个端口参数。但是以后要为Server指定管理端口怎么办，又加参数？大师说NO，用配置类，为配置类加属性就行了。

```java
public class ServerConfig {

    public static final int DEFAULT_PORT = 18080;
    private final int port;

    public ServerConfig(int PORT) {
        this.port = PORT;
    }

    public ServerConfig() {
        this.port = DEFAULT_PORT;
    }

    public int getPort() {
        return port;
    }
}
```

Server重构，修改构造函数

```java
public class SimpleServer implements Server {

    private ServerStatus serverStatus = ServerStatus.STOPED;
    private final int port;

    public SimpleServer(ServerConfig serverConfig) {
        this.port = serverConfig.getPort();
    }

    @Override
    public void start() {
        this.serverStatus = ServerStatus.STARTED;
        System.out.println("Server start");
    }

    @Override
    public void stop() {
        this.serverStatus = ServerStatus.STOPED;
        System.out.println("Server stop");
    }

    @Override
    public ServerStatus getStatus() {
        return serverStatus;
    }

    @Override
    public int getPort() {
        return port;
    }
}
```

ServerFactory重构

```java
public class ServerFactory {
    /**
     * 返回Server实例
     * @return
     */
    public static Server getServer(ServerConfig serverConfig) {
        return new SimpleServer(serverConfig);
    }
}
```

单元测试重构

```java
public class TestServer {
    private static Server server;

    @BeforeClass
    public static void init() {
        ServerConfig serverConfig = new ServerConfig();
        server = ServerFactory.getServer(serverConfig);
    }

    @Test
    public void testServerStart() {
        server.start();
        assertTrue("服务器启动后，状态是STARTED", server.getStatus().equals(ServerStatus.STARTED));
    }

    @Test
    public void testServerStop() {
        server.stop();
        assertTrue("服务器关闭后，状态是STOPED", server.getStatus().equals(ServerStatus.STOPED));
    }

    @Test
    public void testServerPort() {
        int port = server.getPort();
        assertTrue("默认端口号", ServerConfig.DEFAULT_PORT == port);
    }
}
```

跑下测试：

![](/assets/TestServer.jpg)

OK，经过多轮重构，Server接口编写暂时完成。下一步开始实现真正有用的功能。

完整代码：https://github.com/pkpk1234/BeggarServletContainer

