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



