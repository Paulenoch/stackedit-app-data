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


# V2.1 客户端建立本地服务缓存/缓存动态更新

V1版本中调用方每次调用服务，都要去注册中心zookeeper中查找地址，性能较差
可以在客户端建立一个本地缓存，缓存服务地址信息，作为优化的方案

### 1 本地缓存serviceCache类
用```Map<String, List<String>>```存储服务名称与服务提供者地址列表的映射关系。

添加服务地址方法
-   **功能**：将服务名称和服务提供者地址添加到缓存中。
    
-   **逻辑**：
    
    1.  检查缓存中是否已经存在该服务名称：
        
        -   如果存在，获取对应的地址列表，并将新地址添加到列表中。
            
        -   如果不存在，创建一个新的地址列表，并将新地址添加到列表中，然后将服务名称和地址列表存入缓存。
            
    2.  打印日志，提示服务地址已添加到缓存中。添加服务地址

查找和删除服务地址方法同理

### 2 **修改ZKServiceCenter的serviceDiscovery方法**
先从本地缓存中找 ，如果找不到，再去zookeeper中找

### 3 动态缓存更新
首先，目前的项目框架不能使用经典的Cache Aside旁路缓存策略（首先去本地缓存中读，读不到，再去注册中心中读，返回数据时刷新缓存....等等）

假设A服务有三个地址abc，此时服务端感受到A服务请求压力过大，于是增加了一个新地址并注册到zookeeper中，若使用旁路缓存策略，d地址永远无法被存入客户端缓存中

正确做法：**通过在注册中心注册Watcher，监听注册中心的变化，实现本地缓存的动态更新**

#### 事件监听机制

**watcher概念**

-   `zookeeper`提供了数据的`发布/订阅`功能，多个订阅者可同时监听某一特定主题对象，当该主题对象的自身状态发生变化时例如节点内容改变、节点下的子节点列表改变等，会实时、主动通知所有订阅者
    

-   `zookeeper`采用了 `Watcher`机制实现数据的发布订阅功能。该机制在被订阅对象发生变化时会异步通知客户端，因此客户端不必在 `Watcher`注册后轮询阻塞，从而减轻了客户端压力
    

-   `watcher`机制事件上与观察者模式类似，也可看作是一种观察者模式在分布式场景下的实现方式
    

#### watcher架构

`watcher`实现由三个部分组成

-   `zookeeper`服务端
    

-   `zookeeper`客户端
    

-   客户端的`ZKWatchManager对象`

流程
客户端**首先将 `Watcher`注册到服务端**，同时将 `Watcher`对象**保存到客户端的`watch`管理器中**。当`Zookeeper`服务端监听的数据状态发生变化时，服务端会**主动通知客户端**，接着客户端的 `Watch`管理器会**触发相关 `Watcher`**来回调相应处理逻辑，从而完成整体的数据 `发布/订阅`流程![输入图片说明](/imgs/2025-03-23/dkMY7SCp64RsJ2fe.png)

```java
//加入zookeeper事件监听器
watchZK watcher = new watchZK(client, cache);
//监听启动
watcher.watchToUpdate(ROOT_PATH);
```
-   **功能**：注册 Zookeeper 事件监听器，并启动监听。
    
-   **参数**：
    
    -   `client`：Zookeeper 客户端实例。
        
    -   `cache`：本地缓存实例。
        
-   **作用**：
    
    -   `watchZK` 是一个自定义的事件监听器类，用于监听 Zookeeper 节点的变化。
        
    -   `watcher.watchToUpdate(ROOT_PATH)`：启动对根路径（`ROOT_PATH`）的监听，当节点发生变化时，更新本地缓存。
    ![输入图片说明](/imgs/2025-03-23/mZiUPNFeGt0tR53W.jpeg)
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA4NjA3MTQ1MiwtMjA2ODY2MjA1LDExOD
cyNDc0MDIsLTQ4MzkxMzY3NSwtNTA0MDY1MDYxLDEwMDc5MzA5
NTVdfQ==
-->