# 一、网络模型
Java网络编程模型：Socket + NIO Selector


Future和Callback机制
Channel -> EventLoop -> ChannelHandler 

入站 inboud
出站 outbound


# 二、编写Echo服务器


## ServerBootStrap
- localAddress: port/InetAddress
- group: EventLoopGroup: NioEventLoopGroup
- channel:ChannelHandler: ChannelInboundHandlerAdapter
- childHanlder: ChannelInitializer 将自定义的handler设置到channel的pipline中




