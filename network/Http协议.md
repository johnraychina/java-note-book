https://developers.google.com/web/fundamentals/performance/http2

https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/TRACE
https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
https://www.cnblogs.com/xiaolincoding/p/12442435.html


## HTTP/2
Google SPYDY 解决HTTP/1.0的性能问题

Binary Framing二进制分帧：支持多个请求和响应复用同一个TCP连接

数据流stream、消息message、帧frame

Header compression 使用HPack对header压缩

ServerPush服务端推送

Stream prioritization 数据流优先级支持

One connection per-origin 一个域名一个连接

Flow control 流量控制

### 数据流stream、消息message、帧frame
- stream 双向数据流，有唯一编号，可以承载多个message.
- message 一个完整的请求或者响应，由一个或多个frame.
- frame 最小传输单元，header中有stream编号，会按stream编号重组为消息.

### 请求和响应的多路复用
将请求和响应切分为多个独立的frame, 交错发送，在另外一段重组。


## HTTP/1.x相关优化技术
https://hpbn.co/optimizing-application-delivery/#optimizing-for-http1x


### http1.1默认打开 keep-alive 
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Keep-Alive

作用：复用连接，避免频繁建立关闭连接导致低下的网络通信效率（握手开销：延迟，网络包数量）。

> 打开keep-alive时，客户端如何判断服务端的一个message消息发送完毕?

Content-length 基于长度
Transfer-Encoding: chunked 
当客户端向服务器请求一个静态页面或者图片时，服务器可以很清楚的知道内容大小，然后通过content-length来告知客户端。
但是对于动态页面等，服务器无法预知内容大小，需要使用transfer-encoding，Chunked编码将使用若干个Chunk串连而成，由一个标明长度为0的chunk标示结束。

### Leverage HTTP pipelining 


### Apply domain sharding 
一个域名默认只能有6个连接，用多域名分片来突破。

### Bundle resources to reduce HTTP requests
资源绑定

### Inline small resources
内联小资源

HTTP/2 eliminates the need for all of the above HTTP/1.x workarounds, 
making our applications both simpler and more performant. 

Which is to say, the best optimization for HTTP/1.x is to deploy HTTP/2.

