# V2.0 自定义编码器，解码器和序列化器
**ChannelHandler 与 ChannelPipeline** ChannelHandler 是对 Channel 中数据的处理器，这些处理器可以是系统本身定义好的编解码器，也可以是用户自定义的。这些处理器会被统一添加到一个 ChannelPipeline 的对象中，然后按照添加的类别对 Channel 中的数据进行依次处理。

V1版本中使用netty自带的编码器和解码器 来实现数据的传输

在V2版本中，我们可以通过**继承netty提供的基类**，实现自定义的编码器和解码器

### 1 自定义编码器myEncoder
主要功能是将 `RpcRequest` 或 `RpcResponse` 对象按照自定义的消息格式编码为字节流。编码后的字节流包含以下部分：

1.  **消息类型**：2 字节，表示消息是请求还是响应。
    
2.  **序列化方式**：2 字节，表示使用的序列化器类型。
    
3.  **消息长度**：4 字节，表示序列化后的字节数组的长度。
    
4.  **序列化后的字节数组**：实际的消息内容。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAxOTM2Mzk2NCwtNTA0MDY1MDYxLDEwMD
c5MzA5NTVdfQ==
-->