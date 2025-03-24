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
4. InnoDB支持数据库异常崩溃后的安全回复（r
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU3NDc1NTYyN119
-->