# 接收HTTP Body

解析请求中的Body需要注意Transfer-Encoding头。

当Transfer-Encoding的值为chunked时，不应该使用Content-Length去读取Body。

chunked的Body格式为：

> 数据以一系列分块的形式进行发送。
>
> [`Content-Length`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Length)
>
> 首部在这种情况下不被发送。。在每一个分块的开头需要添加当前分块的长度，以十六进制的形式表示，后面紧跟着 '
>
> `\r\n`
>
> ' ，之后是分块本身，后面也是'
>
> `\r\n`
>
> ' 。终止块是一个常规的分块，不同之处在于其长度为0。终止块后面是一个挂载（trailer），由一系列（或者为空）的实体消息首部构成

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Transfer-Encoding





