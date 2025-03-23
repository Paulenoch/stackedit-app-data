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


### 2 自定义解码器myDecoder
`MyDecoder` 的主要功能是将接收到的字节流按照自定义的消息格式解码为 Java 对象。解码过程包括以下步骤：

1.  **读取消息类型**：2 字节，确定消息是请求还是响应。
    
2.  **读取序列化方式**：2 字节，确定使用的序列化器。
    
3.  **读取消息长度**：4 字节，确定序列化后的字节数组的长度。
    
4.  **读取序列化数组**：根据长度读取字节数组。
    
5.  **反序列化为对象**：使用序列化器将字节数组反序列化为 Java 对象，并将其添加到输出列表中。

---
使用自定义编/解码器的好处：
-  将编码解码的过程进行封装，代码变得简洁易读，维护更加方便
    

-   在内部实现消息头的加工，解决沾包问题
    

-   消息头中加入messageType消息类型，对消息的读取机制有了进一步的拓展
---
### 3 自定义序列化器

Serializer接口中定义serializer方法，用于将对象序列化为字节数组；

定义deserializer方法用于反序列化（若使用java自带的序列化器不适用messageType也可以得到相对应的对象，其他方式需要指定消息格式，再根据message转换成相应的对象


引入fastjson包用于将Java 对象序列化为 JSON 字节数组

Deserializer
case0: 处理RpcRequest类型消息
-   **步骤**：
    
    1.  使用 `JSON.parseObject(bytes, RpcRequest.class)` 将字节数组反序列化为 `RpcRequest` 对象。
        
    2.  创建一个 `Object[]` 数组 `objects`，用于存储反序列化后的参数。
        
    3.  遍历 `request.getParams()`，逐个检查参数的类型是否与 `request.getParamsType()` 中定义的类型一致。
        
        -   如果不一致，使用 `JSONObject.toJavaObject` 将 JSON 对象转换为目标类型。
            
        -   如果一致，直接赋值。
            
    4.  将处理后的参数数组 `objects` 设置回 `request` 对象。
        
    5.  将 `request` 对象赋值给 `obj`。

case1:  处理 `RpcResponse` 类型的消息
-   **步骤**：
    
    1.  使用 `JSON.parseObject(bytes, RpcResponse.class)` 将字节数组反序列化为 `RpcResponse` 对象。
        
    2.  获取 `response` 中 `data` 的目标类型 `dataType`。
        
    3.  检查 `response.getData()` 的类型是否与 `dataType` 一致。
        
        -   如果不一致，使用 `JSONObject.toJavaObject` 将 JSON 对象转换为目标类型。
            
    4.  将处理后的 `response` 对象赋值给 `obj`。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MjkxNzQwMywtNTA0MDY1MDYxLDEwMD
c5MzA5NTVdfQ==
-->