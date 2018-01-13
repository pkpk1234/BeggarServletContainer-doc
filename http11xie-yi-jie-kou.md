# HTTP1.1协议实体接口

好不容易完成了一个相对容易扩展的IO层，终于可以开始进入HTTP部分的编写了。早知道就直接找Netty之类的IO框架了，当然那样就没啥意思，也无法找到自己技术上缺陷并改进了。

我们先把HTTP1协议实体接口，用Java对象把HTTP1协议中需要的实体表示出来。然后再编写HTTP Parser相关功能，将IO中获取到的信息转换为上一步中定义的实体对象。这里先将HTTP1协议接口定下来。

根据[HTTP1.1协议标准](https://tools.ietf.org/html/rfc2616)，HTTP协议实体如下：

```
generic-message = start-line
                          *(message-header CRLF)
                          CRLF
                          [ message-body ]
```

分为四个部分：

1. 开始行
2. 消息头
3. 空行
4. 消息体

开始行有分为请求开始行和响应开始行两类：SP代表空白字符

```
Request-Line   = Method SP Request-URI SP HTTP-Version CRLF
```

```
Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

转换为Java后，结构如下![](/assets/http-interface-diagrams.jpg)

StartLine用于表示开始行，RequestLine和ResponseLine都实现该接口，同时添加了返回特定各自信息的方法。

HttBody用表示Body内容，使用泛型指定Body内容的格式，StringContentHttBody表示Body的内容是字符串，ByteContentHttBody表示Body内容是二进制，并与InputStream进行返回。

IMessageHeaders用户持有所有的HttpHeader。HttpMessageHeaders内部使用Guava的ArrayListMultimap&lt;String, HttpHeader&gt;保持HttpHeader，以实现对同名多个消息头的支持。

HttpMessage接口则同时使用 StartLine、IMessageHeaders、HttpBody三个接口，用于表示HTTP Message。

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step7

分支step7

![](/assets/git-br-step7.jpg)

