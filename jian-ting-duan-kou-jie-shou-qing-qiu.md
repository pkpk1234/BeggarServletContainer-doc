# 监听端口接收请求

上一步中我们已经定义好了Server接口，并进行了多次重构，但是实际上那个Server是没啥毛用的东西。现在要为其添加真正有用的功能。大师说了，饭要一口一口吃，衣服要一件一件脱，那么首先来定个小目标——启动ServerSocket监听请求，不要什么多线程不要什么NIO，先完成最简单的功能。下面还是一步一步来写代码并进行重构优化代码结构。

关于Socket和ServerSocket怎么用，网上很多文章写得比我好，大家自己找找就好。



代码写起来很简单：

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



