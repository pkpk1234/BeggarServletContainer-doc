# HTTP Get静态资源请求功能编写

HTTP协议处理真是麻烦，早知道找个现成HTTP框架，只编写Servlet相关部分……

先把HTTP处理流程定下来，根据前面定下来的框架，需要继承AbstractHttpEventHandler，在doHandle方法中编写接受请求内容、生成响应内容，并输出到客户端的代码，模板类：

```java
AbstractHttpEventHandler
```



