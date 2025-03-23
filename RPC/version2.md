# V2.0 自定义编码器，解码器和序列化器
**ChannelHandler 与 ChannelPipeline** ChannelHandler 是对 Channel 中数据的处理器，这些处理器可以是系统本身定义好的编解码器，也可以是用户自定义的。这些处理器会被统一添加到一个 ChannelPipeline 的对象中，然后按照添加的类别对 Channel 中的数据进行依次处理。

V1版本中使用netty自带的编码器和解码器 来实现数据的传输

在V2版本中，我们可以通过**继承netty提供的基类**，实现自定义的编码器和解码器

### 1 自定义编码器myEncoder

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTUwNDA2NTA2MSwxMDA3OTMwOTU1XX0=
-->