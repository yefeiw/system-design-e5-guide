# 09 · 资源地图

> 按用途分类，不是按名气。原则：**免费优先、少而精、每个资源都绑定一个明确的面试用途**。

## 1. 网站（按备考阶段使用）

### 第一梯队（核心，全部免费）

| 资源 | 是什么 | 用在哪 |
|------|--------|--------|
| [Hello Interview](https://www.hellointerview.com) | FAANG 前面试官团队做的免费指南 + 真题库 + 评分表 | **主力**。Delivery Framework、四维 rubric、按难度分级的真题解析（E5 高频：Ticket Booking / Ad Click / Top-K / Rate Limiter / Distributed Cache）、Meta E5 专题指南 |
| [System Design Primer](https://github.com/donnemartin/system-design-primer) | GitHub ~280k star 的知识总纲 | 查漏补缺的地图：每个构件（LB/缓存/一致性/CDN）都有讲解 + 优缺点表 |
| [ByteByteGo](https://bytebytego.com) | Alex Xu 团队，动画图解 + 周刊 | 建立**构件直觉**：每种存储/队列/缓存模式的可视化原理 |

### 第二梯队（专项补强）

| 资源 | 用途 |
|------|------|
| [Design Gurus](https://designgurus.io)（Grokking 出品方）| 付费课程 + 免费 blog；面试四分类（后端/API/前端/OOD）的出处 |
| [interviewing.io](https://interviewing.io) | **真人 mock**，北美口碑最好；录音回放机制极好 |
| [Codemia](https://www.codemia.io) | 号称 system design 版 LeetCode，AI 评分练题 |
| [Exponent](https://www.tryexponent.com) | 付费课程 + mock 平台，题库和讲解质量稳定 |
| [Tech Interview Handbook](https://www.techinterviewhandbook.org) | 免费总览，behavioral 与算法侧更全 |

## 2. YouTube 频道（补概念用，不是主路径）

| 频道 | 风格 | 什么时候看 |
|------|------|-----------|
| **ByteByteGo / Alex Xu** | 动画图解，10 分钟一个概念 | 学构件的第一站（和网站配套）|
| **Gaurav Sen** | 热情讲解 + 面试式走题 | 入门期建立"面试长什么样"的感觉 |
| **Jordan Has No Life** | 2 小时长视频硬核深挖 | 某道题想听到第 8 层细节时 |
| **Tech Dummies (Narendra L)** | 白板一步步画经典题 | 跟着画一遍 Kafka/短链/YouTube |
| **Hussein Nasser** | 数据库/网络底层原理 | **面试不直接考**，但把底子打穿；通勤听 |
| **codeKarle** | 真实大厂案例研究风格 | 补 case study |
| **sudo code** | 前端 system design | 前端岗专用 |

> 社区共识（Reddit r/ExperiencedDevs 多次讨论）：**YouTube 只能建立直觉，不能替代动手**。看视频的每一分钟都要问"我能不看它白板重讲吗"。

## 3. 书（只读三本，按优先级）

| 书 | 读什么部分 | 优先级 |
|----|-----------|--------|
| **Alex Xu《System Design Interview》Vol.1** | 全书 12 章，一题一章 | E5 备考主力，先读 |
| **《System Design Interview》Vol.2** | 挑没见过的题 + 公司内部 scaled 案例 | Vol.1 完成后 |
| **DDIA（Designing Data-Intensive Applications）** | 第 5 章（复制）、6 章（分区）、9 章（一致性与共识）| 只读这三章；全书读完是 Staff+ 的功课，时间紧可跳过 |

## 4. 经典论文（详见 [07 案例研究](07-real-systems-case-studies.md)）

Dynamo → Bigtable → Kafka 论文 → TAO → Raft（可视化版 raft.github.io）。每篇 30–60 分钟，只提取"他们牺牲了什么"。

## 5. Engineering Blogs

| Blog | 高价值内容 |
|------|-----------|
| Netflix TechBlog | 弹性、降级、播放架构 |
| Uber Engineering | H3、marketplace、schemaless |
| Meta Engineering | TAO、memcache、cache 一致性 |
| Cloudflare Blog | 限流（他们的滑动窗口实现）、边缘网络 |
| High Scalability（汇总站）| 各公司架构速览，每天 10 分钟 |

## 6. 资源 × 备考阶段对照表

```
第 1 周   Hello Interview 全部 guide + 本讲义 01–04 章        （认知层）
第 2–3 周 ByteByteGo 构件视频 + Primer 查漏 + Q1–Q4 真题      （构件层）
第 4–5 周 Q5–Q8 真题 + DDIA 三章 + interviewing.io mock ×2    （深度层）
第 6 周   Alex Xu 两册过一遍 + 公司风格（08）+ 复盘             （收尾层）
```

## 7. 一句话原则

> 资源多 ≠ 准备充分。**判断标准只有一个：不看任何材料，你能对着一道新题白板讲满 40 分钟并带 trade-off。** 做不到就回到真题和 mock，而不是再开一个新资源。

## 下一章

→ [10 · 6 周备考计划](10-study-plan.md)
