# System Design @ E5 — Senior Engineer 面试讲义

> 面向北美 E5 / Senior Engineer 职级的 System Design 面试完整备考讲义。
> 综合了 Hello Interview、ByteByteGo、System Design Primer、Design Gurus (Grokking)、System Design Handbook 等北美主流资源的定义、评分标准与框架，并结合 FAANG 面试官视角整理而成。

## 这套讲义是什么

一句话：**从「会做题」到「像 Senior 一样驱动一场设计讨论」**。

E5 的 System Design 面试不是考你背过几个架构图，而是考你能否在一个 45 分钟的开放问题里，**主动收敛需求、提出方案、深入关键子系统、并把每个决策的 trade-off 讲清楚**。这套讲义按五层递进组织：

```
认知（面试考什么）→ 基础（估算/构件）→ 设计能力（真题深挖）→ 真实系统 → 表达与节奏
```

## 目录

| # | 模块 | 内容 | 时间投入 |
|---|------|------|---------|
| 01 | [什么是 System Design](docs/01-what-is-system-design.md) | 定义、面试类型、四维评分标准 | 1 天 |
| 02 | [E5/Senior 的能力预期](docs/02-e5-senior-expectations.md) | E5 vs E4 本质区别、面试官在找什么信号 | 1 天 |
| 03 | [表达框架：45 分钟怎么走](docs/03-delivery-framework.md) | 六阶段时间线 + 每阶段话术模板 | 2 天（练到肌肉记忆）|
| 04 | [估算与必备数字](docs/04-estimation-numbers.md) | Back-of-envelope、Numbers to Know 速查表 | 2 天 |
| 05 | [核心构件](docs/05-building-blocks.md) | LB / 缓存 / 数据库 / 分片 / 一致性 / 消息队列 / CDN | 1–2 周 |
| 06 | [经典真题总览](docs/06-classic-questions-overview.md) | 题目分级、E5 高频清单、练习方法 | — |
| Q1 | [短链服务](docs/questions/01-url-shortener.md) | 入门题 + ID 生成深挖 | 3 小时 |
| Q2 | [限流器](docs/questions/02-rate-limiter.md) | 算法深挖 + 分布式一致性 | 3 小时 |
| Q3 | [Top-K / Heavy Hitters](docs/questions/03-top-k-heavy-hitters.md) | 流式算法 + 精确性 trade-off | 3 小时 |
| Q4 | [聊天系统](docs/questions/04-chat-system.md) | 长连接 / 消息投递语义 / 群聊扩展 | 3 小时 |
| Q5 | [News Feed](docs/questions/05-news-feed.md) | Push vs Pull / Fan-out / 一致性 | 3 小时 |
| Q6 | [分布式消息队列](docs/questions/06-distributed-message-queue.md) | 存储模型 / 投递保证 / Kafka 对比 | 3 小时 |
| Q7 | [票务预订系统](docs/questions/07-ticket-booking.md) | 并发控制 / 强一致 / 排队 | 3 小时 |
| Q8 | [广告点击聚合器](docs/questions/08-ad-click-aggregator.md) | 流处理 / Exactly-once / 计费准确性 | 3 小时 |
| 07 | [真实系统案例研究](docs/07-real-systems-case-studies.md) | Netflix / Uber / 经典论文拆解 | 持续 |
| 08 | [公司风格指南](docs/08-company-guides.md) | Meta Pirate / Google / Amazon 差异 | 0.5 天 |
| 09 | [资源地图](docs/09-resources.md) | 网站 / 视频 / 书籍 / 论文，按用途分类 | — |
| 10 | [6 周备考计划](docs/10-study-plan.md) | 周级计划 + Mock 安排 | — |

## 使用方式

1. **第 1 周**：读 01–04，把评分标准和框架内化。这不是知识，是「考试规则」。
2. **第 2–5 周**：05 过一遍构件，然后进入 06 + Q1–Q8 真题训练。每道题**必须自己先做 45 分钟模拟**，再看讲义对照。
3. **全程**：每周至少 1–2 次 mock（真人优先，[Interviewing.io](https://interviewing.io) / 朋友 / AI 都行）。
4. **临考前 1 周**：过 08 公司风格 + 复盘所有 mock 的失败点。

## 核心信念（来自所有主流资源的共识）

- **没有唯一正确答案**——评估的是你权衡取舍的过程，不是结论。
- **沟通占比 20%**——一半脑子设计，一半脑子讲清楚。
- **Depth > Breadth**——E5 要在至少一个子系统深入到数据模型 / 一致性 / 瓶颈层面。
- **Mock 是最大杠杆**——看 100 个视频不如做 1 次被虐的 mock。

## License

MIT — 随便用，欢迎 PR 补充你被面到的题。
