# E5 / Senior Engineer 系统设计面试讲义

> 面向北美 E5 / Senior Engineer 岗位的系统设计面试准备讲义。
> 参考 Hello Interview、ByteByteGo、System Design Primer、Design Gurus（Grokking）和 System Design Handbook 等主流资料，结合常见评分维度与 FAANG 面试官视角整理而成。

## 这套讲义解决什么问题

一句话：**帮助你从“会做题”，进阶到“能像 Senior 一样主导一场设计讨论”。**

系统设计面试并不考察你是否背过几张架构图。它考察的是：在 45 分钟的开放题里，你能否主动澄清需求、收敛范围、提出可辩护的方案，深入关键子系统，并清楚说明每项决策的取舍。

讲义按五层递进组织：

```
认知（面试考什么）→ 基础（估算与组件）→ 设计能力（经典题深入分析）→ 真实系统 → 表达与节奏
```

## 目录

| # | 模块 | 内容 | 时间投入 |
|---|------|------|---------|
| 01 | [系统设计面试到底考什么](docs/01-what-is-system-design.md) | 定义、面试类型、四维评分标准 | 1 天 |
| 02 | [E5 / Senior 的面试预期](docs/02-e5-senior-expectations.md) | E5 与 E4 的本质差异、面试官在寻找的信号 | 1 天 |
| 03 | [45 分钟面试推进框架](docs/03-delivery-framework.md) | 六阶段时间线与各阶段表达模板 | 2 天（练到形成习惯） |
| 04 | [估算与必备数字](docs/04-estimation-numbers.md) | 数量级推算与关键数字速查表 | 2 天 |
| 05 | [基础组件](docs/05-building-blocks.md) | 负载均衡、缓存、数据库、分片、一致性、消息队列、CDN | 1–2 周 |
| 06 | [经典题总览](docs/06-classic-questions-overview.md) | 题目分级、E5 高频题、练习方法 | — |
| Q1 | [URL Shortener](docs/questions/01-url-shortener.md) | 入门题：ID 生成的深入分析 | 3 小时 |
| Q2 | [Rate Limiter](docs/questions/02-rate-limiter.md) | 算法、一致性与分布式限流 | 3 小时 |
| Q3 | [Top-K / Heavy Hitters](docs/questions/03-top-k-heavy-hitters.md) | 流式算法与精度取舍 | 3 小时 |
| Q4 | [Chat System](docs/questions/04-chat-system.md) | 长连接、消息投递语义与群聊扩展 | 3 小时 |
| Q5 | [News Feed](docs/questions/05-news-feed.md) | 推拉模型、扇出与一致性 | 3 小时 |
| Q6 | [Distributed Message Queue](docs/questions/06-distributed-message-queue.md) | 存储模型、投递语义与 Kafka 对比 | 3 小时 |
| Q7 | [Ticket Booking](docs/questions/07-ticket-booking.md) | 并发控制、强一致性与排队 | 3 小时 |
| Q8 | [Ad Click Aggregator](docs/questions/08-ad-click-aggregator.md) | 流处理、恰好一次与计费准确性 | 3 小时 |
| 07 | [真实系统案例研究](docs/07-real-systems-case-studies.md) | Netflix、Uber 与经典论文拆解 | 持续 |
| 08 | [公司面试风格](docs/08-company-guides.md) | Meta、Google、Amazon 的差异 | 0.5 天 |
| 09 | [资料地图](docs/09-resources.md) | 网站、视频、书籍与论文的使用场景 | — |
| 10 | [六周学习计划](docs/10-study-plan.md) | 周计划与模拟面试安排 | — |

## 使用方式

1. **第 1 周**：阅读 01–04，把评分标准和推进框架真正内化。它们不是知识点，而是这场考试的规则。
2. **第 2–5 周**：先通读 05，再进入 06 与 Q1–Q8。每道题都应先独立完成一次 45 分钟模拟，再与讲义对照。
3. **全程**：每周至少安排 1–2 次模拟面试。真人优先；[Interviewing.io](https://interviewing.io)、朋友和 AI 都可以。
4. **临考前一周**：复习 08，并集中复盘所有模拟面试中反复出现的问题。

## 核心理念

- **没有唯一正确答案**：被评估的是你做取舍的过程，而不是背出的标准解。
- **沟通本身是评分项**：既要设计，也要让面试官始终跟得上你的推理。
- **深度胜过广度**：E5 至少要把一个子系统讲到数据模型、一致性和瓶颈层面。
- **模拟面试的杠杆最大**：看一百个视频，往往不如做一次有复盘的模拟面试。

## License

MIT — 欢迎使用，也欢迎通过 PR 补充你遇到过的题目。
