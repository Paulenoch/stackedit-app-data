# 1. 说说MySQL的架构
包含Server层和存储引擎层

# 2. 一条SQL语句再MySQL中的执行过程
1. 先经过连接器（管理链接，权限验证）
2. 若命中缓存直接返回
3. 分析器（语义分析）
4. 优化器（执行计划生产，索引选择）
5. 执行器（操作引擎返回结果）

# 3. MySQL存储引擎架构了解吗
MySQL存储引擎采用插件式架构，存储引擎基于表，而不是数据库

# 4. MyISAM和InnoDB的区别
1. MyISAM只有表锁，InnoDB支持行锁
2. InnoDB支持事务
3. InnoDB支持外键
4. InnoDB支持数据库异常崩溃后的安全回复（redo log）
5. InnoDB支持MVCC
6. 索引实现不一样：InnoDB 引擎中，其数据文件本身就是索引文件。相比 MyISAM，索引文件和数据文件是分离的

# 5. 什么是事务
逻辑上的一组操作，要么都执行要么都不执行

# 6. 事务四大特性
A：Atomicity：原子性：事务是最小的执行单位，要么都执行要么都不执行
C：Consistency：一致性：执行事务前后，数据保持一致
I：Isolation：隔离性：并发访问数据库时，各并发事务之间的数据库是独立的
D：Durability：持久性：事务提交后，它对数据库的改变是持久的

AID是手段，C是目的

# 7. 并发事务带来哪些问题
- 脏读：

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNjc2MzUzNDddfQ==
-->