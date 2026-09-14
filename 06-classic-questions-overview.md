# 06 · Classic Problems Overview

> E5 prep 不需要刷 50 道题。**把 8–10 道题打到「能 deep dive 一层」的程度，比刷 30 道「都画过图」强得多。**

## 1. problem grading（对齐 Hello Interview Difficulty 体系）

### Easy（必须无瑕疵）
- URL shortener（[Q1](questions/01-url-shortener.md)）——考 ID generate、cache、redirect
- rate limiter（[Q2](questions/02-rate-limiter.md)）——考 algorithmic deep dives + distributed clock
- Top-K / Heavy Hitters（[Q3](questions/03-top-k-heavy-hitters.md)）——考 streaming algorithm + accuracy trade-off

### Medium（E5 core 火力区）
- chat system（[Q4](questions/04-chat-system.md)）
- News Feed（[Q5](questions/05-news-feed.md)）
- distributed message queue（[Q6](questions/06-distributed-message-queue.md)）
- Ticket Booking（[Q7](questions/07-ticket-booking.md)）——Hello Interview flagged E5 high-frequency
- ad click aggregation（[Q8](questions/08-ad-click-aggregator.md)）——E5 high-frequency，流处理全家 bucket

### Hard（有余力再碰）
- Google Drive / Dropbox（文件 storage + 分块 + sync）
- YouTube / Netflix（videos transcoding pipeline + CDN）
- Google Maps（分块 index + 路径规划）
- search engine（爬取 → index → ranking）
- Uber（geolocation index GeoHash + matching 调度）

## 2. 题目背后的「archetype」

几乎所有题都是这 5 个 archetype 的 variant，认出 archetype 就认出了考官的采分点：

| archetype | core 矛盾 | 覆盖题目 |
|------|---------|---------|
| **read-heavy + cache** | hit rate vs consistency | short URL、Feed、个人主页 |
| **write-heavy + peak shaving** | throughput vs latency vs 丢 data | log、click aggregation、message queue |
| **state sync（long-lived connection）** | connection scale vs push latency | 聊天、协作编辑、游戏 |
| **concurrency resource contention** | consistency vs availability | ticketing、flash sale、转账 |
| **Fan-out write amplification** | push-on-write fan-out vs aggregate-on-read | Feed、通知、social graph |

**practice 后期专门练「archetype 迁移」**：拿到没见过的题，先花 30 秒归类 archetype，直接套用 deep-dive repertoire。

## 3. 每道题的 practice workflow（不要反过来！）

```
1. cold-solving（45 min，timed、whiteboard、recording） ← 最重要，禁止先看 walkthrough
2. 对照 walkthrough（30 min），标记三类差距：
   □ 流程性差距（忘了问 X / 时间失控）
   □ 知识性差距（不懂某个组件的原理）
   □ depth 差距（没挖到 trade-off 层）
3. write 一页 personal cheat sheet（决策 checklist，不是知识摘要）
4. 3 天后重做同题（不看 cheat sheet），comparison recording
5. 一周后讲给朋友听 15 分钟版
```

## 4. deep-dive repertoire：每题必备的「pick-one-of-three deep dives」

面试 deep-dive phase（Phase 5）基本是 interviewer 从这三个方向选一个，提前备好：

- **data 层**：storage selection / shard key / index / consistency
- **failure 层**：SPOF / overload / 部分 invalidation（split-brain、network partition）
- **演化层**：traffic ×10 / ×100 之后 architecture 怎么改

## 5. 本 repo 的题目 walkthrough structure

每道题按统一 structure（和 [03 delivery framework](03-delivery-framework.md) 完全对齐）：

1. **problem statement**（interviewer 那张模糊的卡片）
2. **clarifying-questions checklist**（你应该问什么、每个问题的 intent）
3. **estimation walkthrough**（numbers → takeaway）
4. **high-level design**（component diagram ASCII + data flow）
5. **data model**
6. **deep dive 1/2/3**（本题最常见的三个 deep dive 方向及 E5 level 的答案）
7. **red-flag answers**（哪些答案直接暴露不是 E5）
8. **one-minute elevator pitch**（mock 前的复习卡）

## 进入题目

→ [Q1 · URL shortener](questions/01-url-shortener.md)
