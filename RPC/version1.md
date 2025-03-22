# V1.0
1. 客户端获取ClientProxy对象获取代理对象，ClientProxy通过反射构建request对象，通过IOClient的sendRequest方法和服务端进行数据传输
2. 服务端指定监听端口，如果没有链接会一直处于堵塞状态，有链接来了会创建一个新的线程WorkThread执行处理，WorkThread类负责启动线程和客户端进行数据传输
3. WorkThread类中的getResponse方法负责解析收到的request信息，寻找服务进行调用并返回结果。因为一个服务器会有多个服务，所以需要设置一个本地服务存放器serviceProvider存放服务。在接收到服务端的request信息之后，我们在本地服务存放器找到需要的服务，通过反射调用方法，得到结果并返回

# V1.1
在客户端与服务端进行网络传输，采用Java原生的socket编程方式，效率低
引入`netty`高性能网络框架，进行优化

netty和传统socket编程相比有哪些优势

-   io传输由BIO ->NIO模式；底层 池化技术复用资源
    -   **BIO**：同步阻塞I/O，线程在读写数据时会阻塞，直到操作完成。
						   基于流（Stream），数据按字节流或字符流处理。

    
	-   **NIO**：同步非阻塞I/O，线程可以立即返回，无需等待操作完成。
											   基于通道（Channel）和缓冲区（Buffer），数据通过缓冲区进行读写。通过Selector实现多路复用，单个线程可以管理多个通道，适合高并发场景。

-   可以自主编写 编码/解码器，序列化器等等，可拓展性和灵活性高
  
-   支持TCP,UDP多种传输协议；支持堵塞返回和异步返回

## 1. 将IOClient重写为NettyRpcClient
1.  初始化 Netty 客户端（`Bootstrap` 和 `EventLoopGroup`）。
    
2.  连接到服务端。
    
3.  发送 `RpcRequest` 请求。
    
4.  阻塞等待服务端返回 `RpcResponse`。
    
5.  从通道属性中获取响应并返回。

-   **优点**：
    
    -   基于 Netty 的高性能异步通信。
        
    -   支持线程安全的属性存储（`AttributeKey`）。
        
    -   可扩展性强，可以通过 `NettyClientInitializer` 配置自定义的处理器链。
    
    
## 2. NettyClientInitializer类，配置对发送消息的处理机制
1. 指定消息格式，消息长度，解决沾包问题
2. 指定编码器（将消息转为字节数组），解码器（将字节数组转为消息）
3. 指定对接收的消息的处理handler为自定义的业务处理器，用于处理服务端返回的响应。

## 3. NettyClientHandler，指定对接收消息的处理方式
**功能**

-   处理从服务端接收到的 `RpcResponse` 消息。
    
-   将 `RpcResponse` 对象存储到 `Channel` 的属性中，供客户端在 `sendRequest` 方法中读取。
    
-   关闭 `Channel`，表示本次请求-响应交互完成。

## 4. NettyRPCRPCServer类实现RpcServer接口
## 5. NettyServerInitializer类 和NettyClientInitializer类似
## 6. NettyRPCServerHandler类，接收来自客户端的信息，并解析调用


# V1.2 引入ZooKeeper
在part1 和part2 中，我们在调用服务时，对目标的ip地址和端口port都是写死的，默认本机地址和9999端口号

在实际场景下，服务的地址和端口会被记录到【注册中心】中。服务端上线时，在注册中心注册自己的服务与对应的地址，而客户端调用服务时，去注册中心根据服务名找到对应的服务端地址。

`zookeepepr`是一个经典的**分布式**数据一致性解决方案，致力于为分布式应用提供一个高性能、高可用,且具有严格顺序访问控制能力的分布式协调存储服务。

-   1. 高性能
    
    -   `zookeeper`将全量数据存储在**内存**中，并直接服务于客户端的所有非事务请求，尤其用于以读为主的应用场景
        
    

-   2. 高可用
    
    -   `zookeeper`一般以集群的方式对外提供服务，一般`3~5`台机器就可以组成一个可用的 `Zookeeper`集群了，每台机器都会在内存中维护当前的服务器状态，井且每台机器之间都相互保持着通信。只要集群中超过一半的机器都能够正常工作，那么整个集群就能够正常对外服务
        
    

-   3. 严格顺序访问
    
    -   对于来自客户端的每个更新请求，`Zookeeper`都会分配一个全局唯一的递增编号，这个编号反应了所有事务操作的先后顺序

1. 在testClient和ClientProxy中不用再写死ip和端口，改为在rpcClient.sendRequest方法中，先去注册中心中查找服务对应的ip和port，再去连接对应的服务器
```java
InetSocketAddress address = serviceCenter.serviceDiscovery(request.getInterfaceName()); 
String host = address.getHostName(); 
int port = address.getPort();
```
2. ZKServiceCenter类: 实现向注册中心中查询 服务地址的类
3. 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYzODM2MDQyNiwtMTY4MjIzMDkxNyw5Nj
k5NzgwMzEsMzAzMDM1MTcsMzEzNDc4NjU3XX0=
-->