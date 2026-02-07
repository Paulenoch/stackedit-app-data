# 方案总览：DAgger 放哪、怎么改、改多少

## 必做改动（3处）

1.  **第4章 4.8 改名+改口径**
    

-   从《DAgger 数据增强扩展方案 / 扩展计划》改为  
    **《DAgger 闭环数据增强：方法与实现》**
    
-   把 4.8.3 的“计划”改成“本文已实现并在第6章评测”，不要再写“后续工作”。
    
    thesis
    

2.  **第6章新增一个小节：6.8（或 6.9）DAgger 闭环数据增强实验**
    

-   这里放所有 DAgger 的实验图表与结论（你现在那套 D1–D7/D3稳定性等）。
    
-   同时把现在第6章 6.8 的 “MambaVision 替换实验框架” 往后挪一个编号，或者保持 6.8=DAgger、6.9=MambaVision（推荐）。
    
    thesis
    

3.  **结论与展望部分要同步**
    

-   结论里新增一条“DAgger 的实验发现”（作为已完成工作）。
    
-   展望里不再把 DAgger 写成“未来工作”，改成“未来扩展：更多轮次/更困难场景/更大规模采集”等。
    
    thesis
    

----------

# 第4章怎么写：只写“方法与实现”，不写结果

你的小论文“ViT+Mamba=BC baseline”的方法线在第4章已经很完整。DAgger 在第4章只需要做到两点：

## 4.8.1 动机（2段）

-   闭环分布偏移（covariate shift）
    
-   高速闭环下偏移更显著 → 引入 DAgger 合理
    

## 4.8.2 算法与执行策略（把你的真实执行参数写死）

必须写清这些“可复现关键点”（你已经都有）：

-   baseline 策略：ViT+Mamba 的 BC checkpoint（R0）
    
-   β schedule：R1=0.7, R2=0.3, R3=0.0（强调“执行的是 3 轮，不是配置里计划的 5 轮”）
    
-   每轮采集：18 episodes，速度桶分配（9/12 各 6；3/5/7 各 2）
    
-   环境：仅 spheres_medium 采集，Trees 严格 zero-shot
    
-   训练：每轮从上一轮 checkpoint 继续 fine-tune 30 epochs，lr=5e-5，warmup=5，exponential decay（强调“实际执行与 YAML 计划不同，以 pipeline 为准”）
    

> 这一节用一张表就够：Round / β / 新增轨迹 / 累计轨迹 / balance后总量 / epochs / lr / warmup / expert_ratio / checkpoint来源。

## 4.8.3 审计与指标重算（把你“data.csv重算”写成论文规范）

这点会极大加分（因为你论文主打“可审计”）：

-   collision_count 定义为上升沿
    
-   collision_duration 的主结论口径（含0） vs 分布图口径（条件分布 v>0）
    
-   finish_time 用 timestamp-based
    
-   mean_vx 用 state vel_x
    
-   所有指标从 data.csv 逐帧重算，避免 summary 脚本不一致
    

**注意**：第4章不要放结果，不然第6章会重复；你只说“指标定义与审计方式”，结果放第6章。

----------

# 第6章怎么写：DAgger 作为“在 BC baseline 上的新增实验块”

把 DAgger 写成“**在 ViT+Mamba BC baseline 基础上的闭环数据增强实验**”，这完全符合你“前面基于小论文”已经完成的叙事：

-   你先证明 ViT+Mamba vs ViT+LSTM
    
-   再证明 streaming 一致性
    
-   再证明 RACS
    
-   **最后**：在同一 ViT+Mamba 上，引入 DAgger 看能否进一步改善（增量贡献 + 增加工作量）
    

## 建议新增章节结构（6.8 DAgger）

### 6.8.1 实验设置（复述最关键三条即可）

-   R0=BC 的 ViT+Mamba
    
-   3 轮 DAgger（β=0.7/0.3/0.0）
    
-   采集只在 spheres_medium；评测 ID+OOD（Spheres + Trees），速度 3/5/7/9/12，10 trials
    

### 6.8.2 主要结果：碰撞事件次数与碰撞持续时间随轮次下降

放你的两张主图：

-   **Fig D1：Collision Count vs Round (Spheres, 9/12)**
    
-   **Fig D2：Mean Collision Duration vs Round (Spheres, 9/12)**
    

写作要点：

-   强 baseline 下 success 已趋于饱和 → 重点看 count/duration
    
-   解释 count/duration 分解为 “频次” 与 “接触持续” 两个维度
    

### 6.8.3 行为一致性：方差/IQR 收敛是 DAgger 的主要收益

放：

-   **Fig D3：Stability Evolution (std) vs Round**
    

写作要点：

-   std 下降 = 更可预测、更稳定 → 工程部署价值更大
    
-   这也能解释为什么你在有限新增数据（每轮 18 条）下仍能看到收益
    

### 6.8.4 分布可视化（防盲审质疑“只看均值”）

放两张分布图：

-   **Fig D6：Collision Count Distribution per Round**
    
-   **Fig D7：Collision Duration Distribution per Round**
    

必须写清口径差异：

-   D7 主文结论用“含0”，而分布图为“条件分布 v>0”，并说明原因（看发生碰撞的回合更有解释性）。
    

### 6.8.5 BC vs DAgger-final：ID/OOD 的碰撞率与 OOD 效率/平滑性

放：

-   **Fig D5：Collision Rate by Speed (Spheres & Trees)**
    
-   **Fig D4：Trees OOD Multi-metric (jerk/finish_time/mean_vx)**
    

写作要点：

-   强调 zero-shot Trees 仍保持/改善
    
-   把效率和平滑性作为“系统级收益”补充，不要夸大
    

### 6.8.6 小结：增量结论（3条 bullet）

-   DAgger 在 BC baseline 上进一步降低 count 与 duration
    
-   显著降低跨试验方差（稳定性提升）
    
-   在 OOD 上保持/改善效率与平滑性
    
-   限制：轮次少、每轮新增数据少、环境难度导致 success 饱和（这段很诚实，盲审友好）
    

----------

# 你论文里还需要同步修改的地方（避免自相矛盾）

根据你 thesis.pdf 目前内容，以下地方必须改：

1.  **第1章技术路线（阶段A）里写“拟扩展 DAgger”**要改成“已实现 DAgger 并在第6章验证”。
    
    thesis
    
2.  **第4章 4.8.3 “本文的扩展计划”**要改为“本文已完成实现与评测”。
    
    thesis
    
3.  **结论/未来工作**：现在还写“尚未引入 DAgger”，必须删除，改成“已引入并验证；未来扩展更多轮次/更复杂场景”。
    
    thesis
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwNDM5MTQwOThdfQ==
-->