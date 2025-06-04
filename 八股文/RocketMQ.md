![输入图片说明](/imgs/2025-06-04/x6zLKVRsww9WgNT6.png)
**每个消费组在每个队列上维护一个消费位置**
**通过一个主题中维护多个队列提高并发能力**
![输入图片说明](/imgs/2025-06-04/lJ6hatJxYr4QNgxx.png)
`Broker`：主要负责消息的存储、投递和查询以及服务高可用保证。生产者生产消息到 `Broker` ，消费者从 `Broker` 拉取消息并消费。

**一个 `Topic` 分布在多个 `Broker`上，一个 `Broker` 可以配置多个 `Topic` ，它们是多对多的关系**。

如果某个 `Topic` 消息量很大，应该给它多配置几个队列(上文中提到了提高并发能力)，并且 **尽量多分布在不同 `Broker` 上，以减轻某个 `Broker` 的压力** 。

`Topic` 消息量都比较均匀的情况下，如果某个 `broker` 上的队列越多，则该 `broker` 压力越大。

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1NzkwMzUwNTIsLTIwODg3NDY2MTIsLT
IwODg3NDY2MTJdfQ==
-->