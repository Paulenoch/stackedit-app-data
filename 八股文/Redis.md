# 1. Redis常用的数据结构
- String：简单动态字符串
- List：双向列表
- Set：HashSet
- Hash：HashMap
- ZSet：类似Set，加了一个权重参数score，使元素能按照score排序

# 2. Redis单线程模型了解吗
对于读写命令，在 Redis 4.0 版本之后引入了多线程来执行一些大键值对的异步删除操作，Redis 6.0 版本之后引入了多线程来处理网络请求（提高网络 IO 读写性能）

# 3. 单线程如何监听大量的客户端连接
通过IO多路复用，I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户端的大量连接，降低了资源的消耗

Red
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyMTUyMTU4MTgsLTIwODg3NDY2MTJdfQ
==
-->