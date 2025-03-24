# 1. 说说MySQL的架构
包含Server层和存储引擎层

# 2. 一条SQL语句再MySQL中的执行过程
1. 先经过连接器（管理链接，权限验证）
2. 若命中缓存直接返回
3. 分析器（语义分析）
4. 优化器（执行计划sh
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTI2MDU0OTE4XX0=
-->