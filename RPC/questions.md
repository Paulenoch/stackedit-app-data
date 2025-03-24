## 1. Netty传输位于网络结构模型的哪一层
传输层：
- Netty支持TCP和UDP的传输层协议，通过对这些协议的封装和抽象，Netty可以完成传输层的数据传输任务，如建立连接、数据传输和连接关闭等
- Netty的EventLoop和EventLoopGroup等组件基于Java NIO的多路复用器(Selector)，实现了高效的IO处理机制

应用层：
- Netty提供了丰富的协议支持如HTTP，WebSocket，SSL，Protobuf等，Netty通过编解码器，可以在应用层对数据进行编解码
- Netty的ChannelPipeline和ChannelHandler等组件构成了一个灵活的事件处理链，允许开发者在应用层自定义各种事件处理逻辑，如身份验证、消息加密、业务逻辑处理等

## 2. Ne
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg2MDQ2NDI3MCwtNjY1MzQyMzFdfQ==
-->