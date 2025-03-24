## 1. Netty传输位于网络结构模型的哪一层
传输层：
- Netty支持TCP和UDP的传输层协议，通过对这些协议的封装和抽象，Netty可以完成传输层的数据传输任务，如建立连接、数据传输和连接关闭等
- Netty的EventLoop和EventLoopGroup等组件基于Java NIO的多路复用器(Selector)，实现了高效的IO处理机制

应用层：
- Netty提供了丰富的协议支持如HTTP，WebSocket，SSL，Protobuf等，Netty通过编解码器，可以在应用层对数据进行编解码
- Netty的ChannelPipeline和ChannelHandler等组件构成了一个灵活的事件处理链，允许开发者在应用层自定义各种事件处理逻辑，如身份验证、消息加密、业务逻辑处理等

## 2. Netty在项目中的作用和执行流程
netty是一个高性能的网络框架，可以实现高效的信息传输；抽象了Java NIO底层的复杂性，提供了简单的API；提供各种组件处理网络数据

流程：
1. 客户端发起请求：
- 客户端根据服务地址通过netty客户端API创建一个客户端Channel，连接到服务端的指定端口
- 客户端将RPCRequest封装成请求信息，通过Netty的编码器将消息序列化成字节流
- 客户端将序列化后的字节流通过网络发送给服务端

2. 服务端接受请求并处理
- 服务端通过Netty服务端API监听指定端口，等待客户端的连接请求
- 接收到连接请求后，通过Netty的解码器将接收到的字节流反序列化成请求信息
- 响应同上

## 3. 为什么会出现沾包问题，如何解决
netty底层默认通过TCP进行传输，TCP是面向流的协议，接收方在接收数据时无法直接得知一条消息的具体字节数，TCP有流量控制机制，可能导致一个包内包含多条消息或不足一条消息

通过在发送消息时，在头部加入消息长度信息；项目中通过自定义消息的传输协议来解决沾包问题
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTk5NjIyMTY2NiwtNjY1MzQyMzFdfQ==
-->