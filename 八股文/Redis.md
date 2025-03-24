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

# 4. Redis为何给缓存数据设定过期时间，如何判断是否过期，过期数据删除策略？
1. 缓存大小有限，短信验证码之类的功能也要设定过期时间
2. 通过一个过期字典（hash表）保存过期时间
3. redis采用定期删除（周期性地随机从设置了过期时间的 key 中抽查一批，然后逐个检查这些 key 是否过期）+ 惰性删除（只会在取出/查询 key 的时候才对数据进行过期检查）

# 5. Redis内存淘汰策略
六种
volatile/allkeys * lru/ttl/random

后续加入lfu

# 6. RDB持久化
Redis通过创建snapshot来获得存储在内存里面的数据在某个时间点上的副本

# 7. AOF持久化
每执行一条更改Redis中的数据的命令，Redis将该命令写入到内存缓存`server.aof_buf`中，再更具`appendfsync`配置决定什么时候同步到硬盘中的AOF文件

# 8. 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTk1Mjk4MTUyNSwtMjA4ODc0NjYxMl19
-->