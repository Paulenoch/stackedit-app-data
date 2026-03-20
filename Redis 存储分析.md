
# cs_inhouse_channel_all_ctl_live
- channel_chat_chat_access_token.live.
约 11.04 GB / 2970 万 key
    
- cs-tocauth:user_info_cache:token
约 9.41 GB / 1146 万 key，单个key远大于chat_access_token
    
## casetracking:392348641336307168
约 5.43 GB / 6879 万 key，单 key 很小，但 key 数极大
    
## chat:session:qp_msg:
约 1.45 GB / 46.7 万 key，平均每个 key 比较大
    
# cs_inhouse_user_all_ctl_live
## 1-user:tag:v2:live
约 5.87 GB / 1.01 亿 key，占整个集群的 72% 左右
    
## 1-match_
1-match_channel_live/1-match_shopee_user_alive 约 1.27 GB / 890 万 key。
若是匹配结果缓存，可考虑缩短TTL

## 1-user_info_str_live
约 388 MB / 104 万 key。同一个 user 同时存在带 region 和不带 region 的两份缓存


<!--stackedit_data:
eyJoaXN0b3J5IjpbODEwNDM4NTE5LDgwOTA2MDA3MSwzNjM2OD
E0NCwtMTQwNDkxMTQ1MCwtMTIzNDI2NzY2MywyMDkyNjU1MzQ0
LC0xMTYzNjYyNTE4LC0xMTI2ODg5ODg5LDkzNDM4OTYyMl19
-->