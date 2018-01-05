# 监听端口接收请求

上一步中我们已经定义好了Server接口，并进行了多次重构，但是实际上那个Server是没啥毛用的东西。现在要为其添加真正有用的功能。大师说了，饭要一口一口吃，衣服要一件一件脱，那么首先来定个小目标——启动ServerSocket监听请求，不要什么多线程不要什么NIO，先完成最简单的功能。下面还是一步一步来写代码并进行重构优化代码结构。

关于Socket和ServerSocket怎么用，网上很多文章写得比我好，大家自己找找就好。

代码写起来很简单：（下面的代码片段有很多问题哦，大神们请不要急着喷，看完再抽）

```java
public class SimpleServer implements Server {

    ... ...
    @Override
    public void start() {
        Socket socket = null;
        try {
            this.serverSocket = new ServerSocket(this.port);
            this.serverStatus = ServerStatus.STARTED;
            System.out.println("Server start");
            while (true) {
                socket = serverSocket.accept();// 从连接队列中取出一个连接，如果没有则等待
                System.out.println(
                        "新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
            }
        }
        catch (IOException e) {
            e.printStackTrace();
        }
        finally {
            if (socket != null) {
                try {
                    socket.close();
                }
                catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    @Override
    public void stop() {
        try {
            if (this.serverSocket != null) {
                this.serverSocket.close();
            }
        }
        catch (IOException e) {
            e.printStackTrace();
        }
        this.serverStatus = ServerStatus.STOPED;
        System.out.println("Server stop");
    }

    ... ...
}
```

添加单元测试：

```java
public class TestServerAcceptRequest {
    private static Server server;
    // 设置超时时间为500毫秒
    private static final int TIMEOUT = 500;

    @BeforeClass
    public static void init() {
        ServerConfig serverConfig = new ServerConfig();
        server = ServerFactory.getServer(serverConfig);
    }

    @Test
    public void testServerAcceptRequest() {
        // 如果server没有启动，首先启动server
        if (server.getStatus().equals(ServerStatus.STOPED)) {
            //在另外一个线程中启动server
            new Thread(() -> {
                server.start();
            }).run();
            //如果server未启动，就sleep一下
            while (server.getStatus().equals(ServerStatus.STOPED)) {
                System.out.println("等待server启动");
                try {
                    Thread.sleep(500);
                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            Socket socket = new Socket();
            SocketAddress endpoint = new InetSocketAddress("localhost",
                    ServerConfig.DEFAULT_PORT);
            try {
                // 试图发送请求到服务器，超时时间为TIMEOUT
                socket.connect(endpoint, TIMEOUT);
                assertTrue("服务器启动后，能接受请求", socket.isConnected());
            }
            catch (IOException e) {
                e.printStackTrace();
            }
            finally {
                try {
                    socket.close();
                }
                catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @AfterClass
    public static void destroy() {
        server.stop();
    }
}
```

运行单元测试，我檫，怎么偶尔一直输出“等待server启动"，用大师的话说就算”只看见轮子转，不见车跑“。原因其实很简单，因为多线程咯，测试线程一直无法获取到另外一个线程中更新的值。大师又说了，早看不惯满天的System.out.println和到处重复的

```java
try {
    socket.close();
} catch (IOException e) {
    e.printStackTrace();
}
```

了。

大师还说了，代码太垃圾了，问题很多：如果Server.start\(\)时端口被占用、权限不足，start方法根本没有抛出异常嘛，调用者难道像SB一样一直等下去，还有，Socket如果异常了，while\(true\)就退出了，难道一个Socket异常，整个服务器就都挂了，这代码就是一坨屎嘛，滚去重构。

首先为ServerStatus属性添加volatile，保证其可见性。

代码片段：

```java
public class SimpleServer implements Server {
    private volatile ServerStatus serverStatus = ServerStatus.STOPED;
... ...
}
```

然后引入sl4j+log4j2，替换掉漫天的System.out.println。

然后编写closeQuietly方法，专门处理socket的关闭。

```java
public class IoUtils {

    private static Logger logger = LoggerFactory.getLogger(IoUtils.class);

    /**
     * 安静地关闭，不抛出异常
     * @param closeable
     */
    public static void closeQuietly(Closeable closeable) {
        if(closeable != null) {
            try {
                closeable.close();
            } catch (IOException e) {
                logger.error(e.getMessage(),e);
            }
        }
    }
}
```

最后start方法异常时，需要让调用者得到通知，并且一个Socket异常，不影响整个服务器。

重构后再跑单元测试：一切OK

![](/assets/TestServerAcceptRequest.jpg)

到目前为止，一个单线程的可以接收请求的Server就完成了。

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step2

分支step2

![](/assets/git-br-step2.jpg)

