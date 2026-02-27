Wu Dong 大佬，关于CS User给ChatBot User Tag提供支持这个需求有几个点想和你讨论一下，这个是TD文档，目前总结了一下我们这边需要提供三个能力
1.  查询系统中所有 User Tag 配置
2.  根据 ChatBot 传入的 UserID 计算 User Tag，CS计算Tag后落库并返回Tag list
3. 根据ChatBot 传入的 UserID 查询该用户当前已有的所有 Tag（不触发重新计算）

这三个能力需要的接口目前User中已经有了，想要确定的是
1. 2和3功能ChatBot传给我们的统一为ShopeeID
2. 我们目前返回的的Tag list包含每个tag的具体对象信息，是否需要这些对象信息还是只需要tag的名字
3.PM 
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAwNzM5ODAyNiwtMTI3MzM5NTk1NSwzNz
U0ODIxODEsNDQ1MzY2NzczLDg1ODUxMTM2M119
-->