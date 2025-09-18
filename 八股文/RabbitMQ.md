![输入图片说明](/imgs/2025-09-18/ED7VbMAbi1Xl8MTb.png)

在 RabbitMQ 中，消息并不是直接被投递到 **Queue(消息队列)** 中的，中间还必须经过 **Exchange(交换器)** 这一层，**Exchange(交换器)** 会把我们的消息分配到对应的 **Queue(消息队列)** 中。

**Exchange(交换器)** 用来接收生产者发送的消息并将这些消息路由给服务器中的队列中，如果路由不到，或许会返回给 **Producer(生产者)** ，或许会被直接丢弃掉 。这里可以将 RabbitMQ 中的交换器看作一个简单的实体。

生产者将消息发给交换器的时候，一般会指定一个 **RoutingKey(路由键)**，用来指定这个消息的路由规则，而这个 **RoutingKey 需要与交换器类型和绑定键(BindingKey)联合使用才能最终生效**。

RabbitMQ 中通过 **Binding(绑定)** 将 **Exchange(交换器)** 与 **Queue(消息队列)** 关联起来，在绑定的时候一般会指定一个 **BindingKey(绑定建)** ,这样 RabbitMQ 就知道如何正确将消息路由到队列了
![输入图片说明](/imgs/2025-09-18/CTTrYzcV3AVmjfAk.png)

# Broker
对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 服务实例。大多数情况下也可以将一个 RabbitMQ Broker 看作一台 RabbitMQ 服务器。

# Exchange Types
**1、fanout**

fanout 类型的 Exchange 路由规则非常简单，它会把所有发送到该 Exchange 的消息路由到所有与它绑定的 Queue 中，不需要做任何判断操作，所以 fanout 类型是所有的交换机类型里面速度最快的。fanout 类型常用来广播消息。

**2、direct**

direct 类型的 Exchange 路由规则也很简单，它会把消息路由到那些 Bindingkey 与 RoutingKey 完全匹配的 Queue 中。

**3、topic**
-   RoutingKey 为一个点号“．”分隔的字符串（被点号“．”分隔开的每一段独立的字符串称为一个单词），如 “com.rabbitmq.client”、“java.util.concurrent”、“com.hidden.client”;
-   BindingKey 和 RoutingKey 一样也是点号“．”分隔的字符串；
-   BindingKey 中可以存在两种特殊字符串“*”和“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词(可以是零个)。
![输入图片说明](/imgs/2025-09-18/7NZBg7qRiagncMfr.png)
以上图为例：

-   路由键为 “com.rabbitmq.client” 的消息会同时路由到 Queue1 和 Queue2;
-   路由键为 “com.hidden.client” 的消息只会路由到 Queue2 中；
-   路由键为 “com.hidden.demo” 的消息只会路由到 Queue2 中；
-   路由键为 “java.rabbitmq.demo” 的消息只会路由到 Queue1 中；
-   路由键为 “java.util.concurrent” 的消息将会被丢弃或者返回给生产者（需要设置 mandatory 参数），因为它没有匹配任何路由键。

# 死信队列
DLX，全称为 `Dead-Letter-Exchange`，死信交换器，死信邮箱。当消息在一个队列中变成死信 (`dead message`) 之后，它能被重新发送到另一个交换器中，这个交换器就是 DLX，绑定 DLX 的队列就称之为死信队列。

**导致的死信的几种原因**：

-   消息被拒（`Basic.Reject /Basic.Nack`) 且 `requeue = false`。
-   消息 TTL 过期。
-   队列满了，无法再添加。

# RabbitMQ消息如何传输
由于 TCP 链接的创建和销毁开销较大，且并发数受系统资源限制，会造成性能瓶颈，所以 RabbitMQ 使用信道的方式来传输数据。信道（Channel）是生产者、消费者与 RabbitMQ 通信的渠道，信道是建立在 TCP 链接上的虚拟链接，且每条 TCP 链接上的信道数量没有限制。就是说 RabbitMQ 在一条 TCP 链接上建立成百上千个信道来达到多个线程处理，这个 TCP 被多个线程共享，每个信道在 RabbitMQ 都有唯一的 ID，保证了信道私有性，每个信道对应一个线程使用。

# RabbitMQ如何保证可靠性
### 1. 生产者（Producer）到 Broker 的可靠性

确保消息成功从生产者发送到 RabbitMQ 服务器。

-   **Publisher Confirms（发布者确认机制）**：
    
    -   **作用**：这是一种轻量级的、异步的确认机制，用来确认消息是否已经被 RabbitMQ Broker 成功接收并处理。
        
    -   **工作原理**：
        
        1.  生产者将信道（Channel）设置为 Confirm 模式。
            
        2.  在此模式下，所有发布的消息都会被分配一个唯一的ID。
            
        3.  一旦消息被 Broker 成功处理（例如，写入队列或持久化到磁盘），Broker会向生产者发送一个**确认（Ack）**。
            
        4.  如果消息因为内部错误等原因未能处理，Broker会发送一个**否定确认（Nack）**。
            
    -   **优点**：异步处理，对性能影响较小，是保证生产者消息可靠性的**首选方式**。
        
    -   **替代方案：事务（Transactions）**：RabbitMQ也支持事务，生产者可以开启一个事务，发布消息，然后提交事务。这种方式是**同步阻塞**的，会极大地降低吞吐量，现在已**不推荐使用**，Publisher Confirms 是更好的选择。
        

### 2. Broker 自身的可靠性

确保消息在 RabbitMQ Broker 中存储时，不会因为服务器宕机等原因而丢失。

-   **消息持久化（Message Persistence）**：
    
    -   **作用**：默认情况下，消息是存在内存中的，如果 RabbitMQ 重启，消息会丢失。持久化机制可以将消息写入磁盘，确保在 Broker 重启后依然存在。
        
    -   **实现条件（必须同时满足）**：
        
        1.  **将队列声明为持久化（Durable）**：在创建队列时，设置 `durable=true`。这保证了队列的元数据在 Broker 重启后得以恢复。
            
        2.  **将消息标记为持久化**：在发送消息时，设置消息的投递模式（delivery mode）为2（persistent）。
            
    -   **注意**：即使开启了持久化，从消息到达 Broker 到它被写入磁盘之间仍然有一个微小的时间窗口。对于需要极高可靠性的场景，还需要结合 Publisher Confirms 来确保消息已被安全地持久化。
        
-   **高可用性集群（High Availability Clustering）**：
    
    -   **作用**：单个 Broker 节点仍然存在单点故障的风险。通过集群和队列镜像，可以实现数据冗余和故障转移。
        
    -   **实现方式：队列镜像（Mirrored Queues）**：
        
        1.  将多个 RabbitMQ 节点组成一个集群。
            
        2.  通过策略（Policy）将队列设置为镜像模式，使其在集群中的多个节点上拥有完整的副本（一个主节点，若干个镜像节点）。
            
        3.  所有发送到该队列的消息，都会被同步到所有镜像节点上。
            
        4.  如果主节点宕机，其中一个镜像节点会自动被选举为新的主节点，继续提供服务，从而保证了服务的高可用和消息的可靠性。
            
    -   **替代方案：Quorum Queues**：这是比传统镜像队列更新、更可靠的高可用方案，基于 Raft 共识算法，提供了更强的数据一致性保证。
        

### 3. Broker 到消费者（Consumer）的可靠性

确保消息被消费者成功处理，避免因消费者处理失败或中途宕机而导致消息丢失。

-   **消费者确认机制（Consumer Acknowledgements）**：
    
    -   **作用**：确保 RabbitMQ 只有在收到消费者明确的“处理完成”信号后，才会将消息从队列中彻底删除。
        
    -   **工作原理**：
        
        1.  消费者在订阅队列时，可以设置**手动确认模式（manual acknowledgement mode）**，即 `auto_ack=false`。
            
        2.  RabbitMQ 将消息投递给消费者后，会暂时将该消息标记为“unacked”（未确认）状态。
            
        3.  消费者在成功处理完业务逻辑后，会向 RabbitMQ 发送一个**确认（Ack）**。RabbitMQ收到Ack后，才会将消息从队列中删除。
            
        4.  如果消费者在处理消息时发生故障（例如，进程崩溃），或者明确地发送了一个**否定确认（Nack/Reject）**，RabbitMQ 就不会收到Ack。
            
        5.  当消费者连接断开后，RabbitMQ 会将所有未确认的消息**重新放回队列**，以便投递给其他健康的消费者进行处理。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI2ODMwOTY5NCwtMTY3NzI1MzY2OV19
-->