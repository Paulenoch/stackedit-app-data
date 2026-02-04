> 用户进线 → CS 系统要不要打 tag？

1.  读取 **Tag Define**（CS 自己定义的一条“如果用户满足 X 条件，我就认为他是 Y 类人”， 标签模版/配置）
    
2.  对每个 tag：
    
    -   查 UPP：在不在某个 segment？
        
    -   查 SOP：有没有某些 tag？
        
    -   查 CS Criteria：数值是否达标？
        
3.  命中 → 得到 tag_id 列表
    
4.  写入：
    
    -   User tag（可刷新）
        
    -   Contact tag（快照）
        
    -   Case tag（快照）
        
5.  Agent Console 展示：
    
    -   tag 名字 + 颜色
        
    -   promote_message
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2OTUxMDg2N119
-->