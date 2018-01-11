# Connection接口

继续抽象的过程，无论Socket还是SocketChannle，其实都可以抽象为一个表示通信连接的Connection接口。每当Connector监听到端口有请求时，即建立了一个Connection。

NIO的接口和BIO的接口差别实在太大了，没办法只能加了一个不伦不类的ChannelConnection接口，肯定有更好的方案，但是以我现在的水平暂时只能这样设计下了。等以后看了Netty或者Undertow的源码再重构吧。

重构后UML大致如下：![](/assets/UML.jpg)Server包含了1个或者多个Connector，Connector包含一个EventListener，一个EventListener包含一个EventHandler。

每当Connector接受到请求时，就构造一个Connection，Connector将Connection传递给EventListener，EventListener再传递给EventHandler。EventHandler调用Connection获取请求数据，并写入响应数据。

之后如果需要加入Servlet的功能，则需要添加对于的EventHandler，再通过EventHandler将请求Dispatcher到相应的Servlet中，而服务器的其余部分基本不用修改。

面向对象的设计模式功力比较弱，先设计一个勉强能用的架构先。这样单线程Server的IO部分基本就搞好了。

完整代码：[https://github.com/pkpk1234/BeggarServletContainer/tree/step6](https://github.com/pkpk1234/BeggarServletContainer/tree/step6)

分支step6

![](/assets/git-br-step6.jpg)

