# 09 · Resource Map

> categorized by use，不是按名气。原则：**免费优先、少而精、每个 resource 都绑定一个明确的面试用途**。

## 1. websites（按 prep phase 使用）

### Tier 1（core，全部免费）

| resource | 是什么 | 用在哪 |
|------|--------|--------|
| [Hello Interview](https://www.hellointerview.com) | FAANG 前 interviewer 团队做的免费指南 + classic problems 库 + 评分表 | **主力**。Delivery Framework、four-dimension rubric、按 Difficulty tiered 的 classic problems walkthrough（E5 high-frequency：Ticket Booking / Ad Click / Top-K / Rate Limiter / Distributed Cache）、Meta E5 专题指南 |
| [System Design Primer](https://github.com/donnemartin/system-design-primer) | GitHub ~280k star 的知识总纲 | 查漏补缺的地图：每个 building blocks（LB/cache/consistency/CDN）都有讲解 + 优缺点表 |
| [ByteByteGo](https://bytebytego.com) | Alex Xu 团队，动画图解 + 周刊 | 建立**building blocks 直觉**：每种 storage/queue/cache 模式的可视化原理 |

### Tier 2（专项补强）

| resource | 用途 |
|------|------|
| [Design Gurus](https://designgurus.io)（Grokking 出品方）| 付费课程 + 免费 blog；面试四分类（backend/API/frontend/OOD）的出处 |
| [interviewing.io](https://interviewing.io) | **live mock**，北美口碑最好；录音回放机制极好 |
| [Codemia](https://www.codemia.io) | 号称 system design 版 LeetCode，AI 评分练题 |
| [Exponent](https://www.tryexponent.com) | 付费课程 + mock 平台，题库和讲解质量稳定 |
| [Tech Interview Handbook](https://www.techinterviewhandbook.org) | 免费 overview，behavioral 与 algorithm 侧更全 |

## 2. YouTube 频道（补概念用，不是主路径）

| 频道 | style | 什么时候看 |
|------|------|-----------|
| **ByteByteGo / Alex Xu** | 动画图解，10 分钟一个概念 | 学 building blocks 的第一站（和 websites 配套）|
| **Gaurav Sen** | 热情讲解 + 面试式走题 | 入门期建立"面试长什么样"的感觉 |
| **Jordan Has No Life** | 2 小时长 videos 硬核 deep dive | 某道题想听到第 8 层细节时 |
| **Tech Dummies (Narendra L)** | whiteboard 一步步画经典题 | 跟着画一遍 Kafka/short URL/YouTube |
| **Hussein Nasser** | database/network 底层原理 | **面试不直接考**，但把底子打穿；通勤听 |
| **codeKarle** | 真实大厂 case studies style | 补 case study |
| **sudo code** | frontend system design | frontend 岗专用 |

> 社区 consensus（Reddit r/ExperiencedDevs 多次讨论）：**YouTube 只能建立直觉，不能替代动手**。看 videos 的每一分钟都要问"我能不看它 whiteboard 重讲吗"。

## 3. 书（只 read 三本，按 priority）

| 书 | read vs write 部分 | priority |
|----|-----------|--------|
| **Alex Xu《System Design Interview》Vol.1** | 全书 12 章，一题一章 | E5 prep 主力，先 read |
| **《System Design Interview》Vol.2** | 挑没见过的题 + 公司内部 scaled 案例 | Vol.1 完成后 |
| **DDIA（Designing Data-Intensive Applications）** | 第 5 章（replication）、6 章（partition）、9 章（consistency 与 consensus）| 只 read 这三章；全书 read 完是 Staff+ 的功课，时间紧可跳过 |

## 4. classic papers（see [07 case studies](07-real-systems-case-studies.md)）

Dynamo → Bigtable → Kafka papers → TAO → Raft（可视化版 raft.github.io）。每篇 30–60 分钟，只提取"他们牺牲了什么"。

## 5. Engineering Blogs

| Blog | 高价值内容 |
|------|-----------|
| Netflix TechBlog | elasticity、degradation、playback architecture |
| Uber Engineering | H3、marketplace、schemaless |
| Meta Engineering | TAO、memcache、cache consistency |
| Cloudflare Blog | rate limiting（他们的 sliding window 实现）、edge network |
| High Scalability（汇总站）| 各公司 architecture 速览，每天 10 分钟 |

## 6. resource × prep phase 对照表

```
第 1 周 Hello Interview 全部 guide + 本讲义 01–04 章 （认知层）
第 2–3 周 ByteByteGo building blocks videos + Primer 查漏 + Q1–Q4 classic problems （building blocks 层）
第 4–5 周 Q5–Q8 classic problems + DDIA 三章 + interviewing.io mock ×2 （depth 层）
第 6 周 Alex Xu 两册过一遍 + company style（08）+ retrospective （wrap-up 层）
```

## 7. one-line principle

> resource 多 ≠ 准备充分。**判断标准只有一个：不看任何材料，你能对着一道新题 whiteboard 讲满 40 分钟并带 trade-off。** 做不到就回到 classic problems 和 mock，而不是再开一个新 resource。

## Next Module

→ [10 · 6-Week Study Plan](10-study-plan.md)
