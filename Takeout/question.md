# 1. Nginx反向代理
**nginx 反向代理的好处：**

-   提高访问速度：因为nginx本身可以进行缓存，如果访问的同一接口，并且做了数据缓存，nginx就直接可把数据返回，不需要真正地访问服务端，从而提高访问速度。
-   进行负载均衡
-   保证后端服务安全：因为一般后台服务地址不会暴露，所以使用浏览器不能直接访问，可以把nginx作为请求访问的入口，请求到达nginx后转发到具体的服务中，从而保证后端服务的安全。

**负载均衡**
nginx有很多负载均衡策略，比如轮询（默认），weight权重方式,url分配方式，我们项目用的是轮询方式，共有3台后端服务器

# 2. 实现布隆过滤器解决缓存穿透问题
## 2.1 什么是缓存穿透
缓存穿透:请求根本不存在的资源（DB本身就不存在，Redis更是不存在），这将导致这个不存在的数据每次请求都要到 DB 去查询，可能导致 DB 挂掉

## 2.2 什么是布隆过滤器
布隆过滤器主要是用于检索一个元素是否在一个集合中。

布隆过滤器的核心思想是使用多个哈希函数来将元素映射到**位数组**中的多个位置上。当一个元素被加入到布隆过滤器中时，它会被多次哈希，并`将对应的位数组位置设置为1`。当需要判断一个元素是否在布隆过滤器中时，我们只需将该元素进行多次哈希，并检查对应的位数组位置是否都为1，如果`其中有任意一位为0，则说明该元素不在集合中`；如果所有位都为1，则说明该元素`可能`在集合中（因为有可能存在哈希冲突），需要进一步检查。

使用BitMap作为布隆过滤器，使用多个hash函数对key进行hash运算

当布隆过滤器实际的数据存储量超过预期数据量之后，误判率也会随之上涨。但是布隆过滤器是不能删除已有元素的，在这里我们采取的方案是再创建一个布隆过滤器

# 3. MySQL主从复制，读写分离
主从复制实际上就是将主库的数据同步到从库数据，通过Mysql主从复制就可以实现从库中的数据和主库中的数据一致。MySQL复制过程分为三步：

第一：主库在事务提交时，会把数据变更记录在二进制日志文件 Binlog 中。

第二：从库读取主库的二进制日志文件 Binlog ，写入到从库的中继日志 Relay Log 。

第三：从库重做中继日志中的事件，在slave库上做相应的更改。

主库配置
```sql
[mysqld]
server-id = 1               # 必须唯一
log_bin = mysql-bin         # 开启二进制日志
binlog_format = ROW         # 推荐使用ROW格式
binlog-do-db = 需要复制的数据库名 # 可选，指定复制哪些库
```
从库配置
```sql
[mysqld]
server-id = 2               # 不同于主库
relay-log = mysql-relay-bin # 中继日志
read_only = ON              # 从库设为只读(不影响super用户)
```
主库执行
```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 查看主库状态，记录File和Position
SHOW MASTER STATUS;
```
从库执行
```sql
CHANGE MASTER TO
MASTER_HOST='master_ip',
MASTER_USER='repl',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.000001', -- 主库SHOW MASTER STATUS结果
MASTER_LOG_POS=120;                 -- 主库SHOW MASTER STATUS结果

START SLAVE;
```

### Sharding-JDBC怎么实现读写分离

对于同一时刻有大量并发读操作和较少写操作类型的应用来说，将数据库拆分为主库和从库，主库就负责处理事务性的增删改操作，从库负责处理查询操作，能够有效的避免由数据更新导致的行锁（innodb引擎支持的就是行锁），使得整个系统的性能得到极大改善。

Sharding-JDBC介绍Sharding-JDBC定位为轻量级java框架，在java的JDBC层提供的额外服务。它使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。使用Sharding-JDBC可以在程序中轻松的实现数据库读写分离,优点在于数据源完全有Sharding托管，写操作自动执行master库，读操作自动执行slave库。不需要程序员在程序中关注这个实现了。
```yml
spring:
  shardingsphere:
    datasource:
      names: master,slave1,slave2
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://master_ip:3306/db
        username: root
        password: password
      slave1:
        # 类似配置...
      slave2:
        # 类似配置...
    masterslave:
      load-balance-algorithm-type: round_robin # 轮询
      name: ms
      master-data-source-name: master
      slave-data-source-names: slave1,slave2
    props:
      sql.show: true
```

# 4. 使用Redis，采用一主两从＋哨兵的集群方案

### 4.1 为什么用Redis
频繁的访问数据库导致数据库压力大，系统的性能下降，用户体验感差。因此使用Redis对数据进行缓存，从而减小数据库的压力，在数据更新时删除缓存，从而保证数据库和缓存的一致性，同时有效提高系统的性能和访问速度。

### 4.2 为什么采用一主两从+哨兵
单Redis的并发能力是有上限的，搭建主从集群实现读写分离。主节点负责写数据，从节点负责读数据（解决高并发问题）

主数据库若崩溃。哨兵作为一个独立运行的进程，通过发送ping命令等待Redis服务器响应，从而监控运行的多个Redis实例。当检测到master宕机，会自动将slave切换成master。通过发布订阅模式通知其他slave修改配置文件，让它们切换master

### 4.3 redis主从同步流程：
1. 全量同步（第一次）
- slave携带自己的replication id和offset请求master同步数据
- master根据主从节点的replication id是否一样判断是否为第一次同步（若不一样说明是第一次同步，master会把自己的replication id和offset发给slave）
- master执行bgsave，生成RDB文件后发给slave，slave先清空自己，再执行RDB文件
- 若在RDB文件生成期间master发生了变化，master会把变化用命令的形式记录到AOF缓冲区，再将AOF日志文件发给slave

2. 增量同步
当从节点服务重启之后，数据就不一致了，所以这个时候，从节点会请求主节点同步数据，主节点还是判断不是第一次请求，不是第一次就获取从节点的offset值，然后主节点从命令日志中获取offset值之后的数据，发送给从节点进行数据同步

# 5. 如何解决数据一致性问题
### 1.redisson读写锁解决

我们项目使用redisson实现的读写锁来解决双写一致性问题。在读的时候添加共享锁，可以保证读读不互斥，读写互斥(其他线程可以一起读，但是不能写)。当我们更新数据的时候，添加排他锁，它是读写，读读都互斥（其他线程不能读也不能写），这样就能保证在写数据的同时是不会让其他线程读数据的，**避免了脏数据**。这里面需要注意的是读方法和写方法上需要使用同一把锁才行。

那这个排他锁是如何保证读写、读读互斥的呢？

其实排他锁底层使用也是setnx，保证了同时只能有一个线程操作锁
```java
RReadWriteLock rwLock = redisson.getReadWriteLock("myLock");

// 获取读锁(共享锁)
RLock readLock = rwLock.readLock();
try {
    readLock.lock();
    // 执行读操作
    // 多个线程可以同时进入此代码块
} finally {
    readLock.unlock();
}

// 获取写锁(排他锁)
RLock writeLock = rwLock.writeLock();
try {
    writeLock.lock();
    // 执行写操作
    // 同一时间只有一个线程可以进入此代码块
} finally {
    writeLock.unlock();
}
```

### 2. 旁路缓存

如果不想加锁的话，可以采用旁路缓存模式,先更新 db，后删除 cache

在普通"更新数据库+删除缓存"模式下，可能出现以下时序问题：

1.  线程A更新数据库（但还未完成）
    
2.  线程B读取缓存（未命中）→ 读取旧数据库数据 → 写入缓存
    
3.  线程A完成数据库更新并删除缓存
    
4.  结果：缓存中仍然是旧数据
    

延时双删通过第二次延迟删除，可以清除这种"中间状态"的脏数据。
#### 2.什么是延迟双删?

延迟双删，如果是写操作，我们先把缓存中的数据删除，然后更新数据库，最后再延时删除缓存中的数据，其中这个延时多久不太好确定，在延时的过程中可能会出现脏数据，并不能保证强一致性，所以没有采用它。-   延迟时间需要大于"读数据库+写缓存"的耗时，但这个时间难以准确预估

# 6. 乐观锁解决超卖问题
项目中使用版本号实现乐观锁
在商品表中增加一个版本号字段，每次更新库存时，都会携带这个版本号。如果版本号没有发生变化，说明在此期间没有其他线程修改过数据，更新操作可以进行；如果版本号发生了变化，则说明有其他线程已经修改过数据，此时需要重新获取最新的数据并尝试再次更新。
```sql
"UPDATE goods SET stock = stock - 1, version = version + 1 WHERE id = #{id} AND stock > 0 AND version = #{version}
//当商品的库存大于0且版本号与传入的版本号相同时，将库存减1，并将版本号加1。
```
### 2.CAS法实现乐观锁

CAS用库存量代替版本号，在执行操作时判断 where 用的库存量和在最初查询的时候，是否相等。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwNDkxMzIzMDcsLTE5ODgxNDc3OSwtMz
kyMTg4NTYyLDIwNDc0ODEzODMsMTU2OTA1OTYzNCwyMDgzMzg3
NzE2LDE0OTY1MzI2MDRdfQ==
-->