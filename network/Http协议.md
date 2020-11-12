https://developers.google.com/web/fundamentals/performance/http2

https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/TRACE
https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html





## HTTP/2
Google SPYDY 解决HTTP/1.0的性能问题

Binary Framing二进制分帧：支持多个请求和响应复用同一个TCP连接

三个概念：数据流stream、消息message、帧frame

Header compression 使用HPack对header压缩

ServerPush服务端推送

Stream prioritization 数据流优先级支持

One connection per origin 一个域名一个链接

Flow control 流量控制

## HTTP/1.x相关优化技术
https://hpbn.co/optimizing-application-delivery/#optimizing-for-http1x

Leverage HTTP pipelining
管道

Apply domain sharding
一个域名默认只能有6个连接，用多域名分片来突破。

Bundle resources to reduce HTTP requests
资源绑定

Inline small resources
内联小资源

HTTP/2 eliminates the need for all of the above HTTP/1.x workarounds, making our applications both simpler and more performant. Which is to say, the best optimization for HTTP/1.x is to deploy HTTP/2.