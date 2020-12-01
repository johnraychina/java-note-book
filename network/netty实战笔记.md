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

## Netty 组件
Channel(socket),
EventLoop(控制流，多线程处理，并发), 
ChannelFuture(异步通知)
ChannelHandler, ChannlePipeline

## 传输
核心是Channel

## ByteBuf
Heap模式
Direct Memory模式
Composite复合缓冲区

readerIndex/writerIndex

## ChannelHandler, ChanelPipeline
channel生命周期：registered, active, inactive, unregistered.
channelHandler生命周期: add, remove, exceptionCaught

## Encoder,Decoder



