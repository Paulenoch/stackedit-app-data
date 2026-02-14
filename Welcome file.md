# 四、接下来你只需要做的“最小 Gate104 输入准备”

为了让你不被工程细节拖慢，我建议你先做一个最小风险集版本：

-   只收集 **12m/s** 的 30 条 episode（因为方差最大、最痛）
    
-   每条只存触发窗口（collision/d_min<0.2）
    
-   teacher 回放打标签
    
-   微调 3 epoch
    
-   评测 12m/s 10 trials + 9m/s 5 trials
    

如果这个版本能明显降低 12m/s 方差/碰撞，你再扩展到 5/9。
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDQ1MzY2NzczLDg1ODUxMTM2M119
-->