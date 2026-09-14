# 01 · What Is the System Design Interview

> 本章回答三个问题：这场面试**到底在考什么**、有**哪几种 formats**、interviewer**按什么标准打分**。

## 1. 各家权威 definitions

### Design Gurus（Grokking 系列出品方）

> "system 设计面试是一个开放式技术面试：你被 requirement 从零开始设计一个大 scale distributed system。它没有唯一正确答案——评估的是你**澄清需求、handling ambiguity、并做出关键 architecture 决策**的能力。"

这是最标准的教科书式 definitions。注意两个关键词：**开放式**（意味着主动权在你）和**没有对错**（意味着评分的是过程）。

### Hello Interview（FAANG 前 interviewer 团队，目前最好的免费 resource）

他们不纠结 definitions，直接给评分表。他们对 System Design 面试的理解是：

> 一场 45 分钟的协作设计 session，interviewer 扮演资深同事/评审者的角色，看你如何把一个模糊的产品需求变成一个可辩护的技术 architecture。

### System Design Primer（GitHub ~280k star）

> "学习如何设计大 scale system。帮助你成为更好的工程师，并附带面试准备。"

定位是 resource 总纲：按主题（load balancing、consistency、CDN、cache……）组织知识，附 classic problems 与参考答案。适合当地图用，不适合当路线用。

### System Design Handbook

强调一个所有 senior 都该刻在脑子里的观念：

> **不存在完美的设计。每一个选择都有 cost。** 好的设计不是「最优解」，而是在给定约束下最合理的折中。

另外一个冷知识：Meta 内部把 system 设计轮叫 **"Pirate" 轮**（因为主持这轮的 interviewer 小组代号 "Pirates"），近年又分化出偏 API/产品设计的 **"Pirate X"**（see [08 Company Style Guide](08-company-guides.md)）。

## 2. 面试的四种 formats

按 Design Gurus 的分类（E5 backend 方向 90% 是第一种）：

| types | 考什么 | classic problems | 出现 scenario |
|------|--------|--------|---------|
| **backend / distributed system 设计** | 可 scaling architecture、data modeling、bottleneck 分析 | short URL、Feed、ticketing system | backend/全栈 E5 主战场 |
| **API 设计** | REST semantics、resource 建模、版本化、pagination | 设计 Twitter API、payment webhook | 部分公司独立一轮 |
| **frontend system 设计** | 组件 architecture、state 管理、渲染性能 | 设计 Netflix 播放器、表格组件 | frontend 岗专属 |
| **OO 设计（OOD）** | 类图、设计模式、领域建模 | 设计停车场、象棋 | 多见于中小厂 |

> Meta 特别提示：E5 可能遇到 **Product Architecture 轮**（偏 API/data modeling/user flow）而非传统 System Design 轮（偏基础设施/scalability）。rubric 相同，但 deep dive 方向不同——见过太多人按纯 backend 准备结果撞上 Product Architecture。

## 3. four-dimension rubric（Hello Interview Rubric）

这是目前公开最接近大厂内部评分表的 framework，**四个 dimension、按百分比加权**：

### 3.1 Problem Navigation — 30%

- **主动澄清**：scale（QPS / DAU / data 量）、read/write ratio、consistency requirement、latency budget、client types
- **scope narrowing**：明确说出「我打算先做 X，Y 放到 follow-up」——interviewer 最怕你什么都想做
- **对齐 definitions**：把模糊词翻译成具体 feature（「快」是多快？「活跃 user」怎么 definitions？）

### 3.2 Solution Design — 30%

- 高层 architecture 清晰：画得出 component diagram，说得清 data flow 向
- 能覆盖 core feature 与 non-functional requirements（availability、scalability、latency）
- **approach 与需求 matching**：日活 1 万的 system 不要上来就 sharding + multi-region DR（over-engineering 在 E5 是减分项）

### 3.3 Technical Depth & Differentiation — 20%

- deep dive 至少一个 subsystem 到实现层面：storage engine selection、index 设计、consistency 协议、failure 模式
- 能讲清 trade-off：「选 A，cost 是 X，因为我们的 scenario 里 X acceptable 而 Y 不 acceptable」
- 展现超出「standard answers」的理解：知道常见 approach 的坑（如 Redis cache invalidation storm、Kafka partition skew）

### 3.4 Communication & Collaboration — 20%

- structure 化表达：先总后分，随时让 interviewer 知道你在哪一层
- **接受引导**：interviewer 纠偏时快速调整而不是硬扛（interviewer 的 hint 是礼物）
- 驾驭 whiteboard：图要能 read 懂，draw-as-you-talk

> 注意 30/30/20/20 这个 structure 传递的 signal：**「走对了流程」和「approach 合理」占 60 分，比「技术多硬核」更重要**。很多人挂在这一关不是因为技术差，而是因为流程失控——不问需求直接画图，或者被自己带进死角出不来。

## 4. interviewer 在打分时实际在想什么

把 rubric 翻译成 interviewer 的内心 OS：

| Rubric 条目 | interviewer 内心 OS |
|------------|--------------|
| 主动澄清 | 「我还没给需求，他就把该问的都问了」 |
| scope narrowing | 「他知道什么重要什么不重要，像个做过线上 system 的人」 |
| approach matching 需求 | 「这个 scale 这个 approach 刚好，不过度也不过简」 |
| deep dive trade-off | 「他不只是 read 过 ByteByteGo，他知道这些组件真实的坑」 |
| 接受引导 | 「我 hint 了一句他立刻调头，合作起来会很舒服」 |

**E5 的 dividing line**：E4 interviewer 拉着你走完全程也能过；E5 必须是你驱动全程，interviewer 只是在 review 你的设计。

## 5. 一场 45 分钟面试的真实解剖

```
0:00–2:00 interviewer 介绍题目（很模糊的一句话）
2:00–7:00 你澄清需求、estimation scale、definitions scope ← Navigation
7:00–20:00 high-level design：component diagram + API + data model ← Design
20:00–38:00 deep dive 2–3 个 subsystem（interviewer 选方向） ← Depth
38:00–43:00 bottleneck、monitoring、failure、scaling、wrap-up ← Depth + wrap-up
43:00–45:00 你的提问
```

每一阶段的具体打法见 [03 delivery framework](03-delivery-framework.md)。

## Next Module

→ [02 · E5/Senior 的 Expectations](02-e5-senior-expectations.md)
