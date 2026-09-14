# 03 · Delivery Framework: Running the 45 Minutes

> 各家 framework（Hello Interview、Design Gurus、Alex Xu 4-step method）本质是同一副 skeleton。本章统一成一条 6 阶段 timeline，并给出每阶段的 script templates——template 的作用不是背诵，是让你在高压下不用重新发明 structure。

## 0. 为什么需要 framework

没有 framework 的面试最典型死法：**time sink**。在某一个环节（通常是画图或 deep dive）待了 25 分钟，然后被 interviewer 强行拖走，后面全是赶路，没有一处 deep dive。framework 的本质是**time budget**——每一阶段 timeout 你都要有知觉。

## 1. six-phase overview

```
┌──────────────────────────────────────────────────────────────┐
│ Phase 1 requirements clarification 5 min 问题 checklist + scale estimation │
│ Phase 2 core feature confirmation 3 min 3–5 个 features + 明确不做的东西 │
│ Phase 3 high-level design 10 min component diagram + data flow + API skeleton │
│ Phase 4 data model 5 min storage selection + 表/KV structure + sharding │
│ Phase 5 deep dive 17 min 2–3 个 subsystem（和 interviewer negotiate） │
│ Phase 6 wrap-up 5 min bottleneck / monitoring / operations / out-of-scope items │
└──────────────────────────────────────────────────────────────┘
```

> 注意：Phase 3+4 合起来就是经典的「high-level design」；Alex Xu 的 4-step method 把 1+2 merge 为「理解需求」。structure 可以微调，**时间纪律不能破**。

## 2. Phase 1 · requirements clarification（5 分钟）

### must-ask checklist（背下来，每场都过一遍）

**scale 类**
- DAU / MAU 多少？——决定后面所有 capacity numbers
- read-heavy 还是 write-heavy？read/write ratio？——决定 storage 和 cache 策略
- peak multiplier 多少（日常 QPS × 3~5）？

**feature boundaries 类**
- core user flow 是什么？（让 interviewer 描述一个 user story）
- 有哪些 client？（Web / iOS / Android / third-party API）
- 要不要支持 offline / flaky network？

**non-functional requirements 类（主动提，bonus signal）**
- latency budget：P99 read 200ms 这种 order of magnitude acceptable 吗？
- consistency：eventually consistent（second-level latency OK）还是要 read-your-writes？
- availability target：几个 9？allow degraded mode 吗？
- data 可以丢吗？丢多少？（log 类 vs billing 类天壤之别）

### script templates

> "让我先花几分钟对齐需求。我先问几个 scale 和 boundary 的问题，然后我会做一轮 estimation，你看 numbers 方向对不对。"

> "这个题目没说 read/write ratio，我假设 100:1——如果接近 1:1，我后面 storage selection 会完全不同，所以想先跟你确认。"

**关键动作**：把 interviewer 的回答**write 到 whiteboard 上**。这不是形式——后面每个设计决策你都要回头指它。

## 3. Phase 2 · core feature confirmation（3 分钟）

从需求里**distill 3–5 个 features**，并明确说什么不做：

> "基于刚才的需求，我打算聚焦这四个 feature：①…②…③…④…。multi-region DR 和 third-party 开放 API 我先不展开，放到最后的 trade-off 里聊，可以吗？"

这一步是 E5 signal 最密集的 30 秒：**convergence = 你知道什么重要**。interviewer 几乎总会说 yes，而这句话已经把「over-engineering」的 risk 提前拆掉了。

## 4. Phase 3 · high-level design（10 分钟）

画一张 component diagram，包含：client → gateway/LB → stateless service tier → cache → storage →（async）queue → downstream consumer。

三条纪律：
1. **draw-as-you-talk**，每个框只讲一句话的职责——细节留给 deep-dive phase
2. **data flow 用编号箭头**：走一遍「user 发一个 request，12345 经过哪些组件」
3. **每 2–3 分钟对齐一次**："这个高层 structure OK 吗？OK 的话我进入 data model。"——防止你在错误的方向上狂奔 10 分钟

API skeleton（可选，30 秒过）：只列 endpoint 名 + 动词，不 write parameter 细节，除非面的是 Product Architecture 轮（见 08 章）。

## 5. Phase 4 · data model（5 分钟）

- 选 storage：SQL / NoSQL / KV / time-series / search engine——**必须带理由**（见 05 章 selection 表）
- 画 table schema 或 KV structure：primary key、partition key、secondary index
- 主动说 shard key 选择和 hot spot risk："我用 user_id 做 partition key，celebrity user 的 partition 会 skew，deep-dive phase 我讲怎么处理"

## 6. Phase 5 · deep dive（17 分钟）——E5 的主战场

### 怎么选 deep-dive spots

> "现在有三处值得深入：A（fan-out 的 write amplification）、B（storage 的 scaling path）、C（consistency window）。我自己最想聊 A，因为它是最可能先挂的地方。你想看哪个？"

这句话同时拿到三个高分 signal：主动性、risk 嗅觉、协作。interviewer 通常会选 A，或指出他关心的那个——**他说什么就挖什么**，他选的方向往往是他在打分的方向。

### deep dive 的「drill-three-levels」法

每钻一层回答：**是什么 → 为什么 → what if it breaks**

```
第一层：approach（"message delivery 用 long polling"）
第二层：为什么（"comparison WebSocket：不需要双向低 latency，long polling operations 简单、兼容性好"）
第三层：failure（"connection saturation 怎么办？→ connection gateway partition + heartbeat exclude + fall back to polling"）
```

### 被追问到知识 blind spot 时

> "这块我没实际操作过。基于 X 和 Y 的原理，我的推理是 Z；但我没有把握，如果不对你可以纠正我。"

诚实 + 推理 > 硬编。E5 interviewer 阅人无数，编造 100% 被识破。

## 7. Phase 6 · wrap-up（5 分钟）

留出最后 5 分钟（提前看表），主动覆盖：

- **SPOF**：图里哪个框挂了最伤？→ redundancy approach
- **bottleneck**：traffic ×10 先死哪里？（通常是 DB write、fan-out、network bandwidth 三选一）
- **monitoring**：3 个 SLI（latency P99 / error rate / backlog）
- **operations**：how to deploy、怎么 canary rollout、怎么 rollback
- **out-of-scope items**："geo-routing 和 cost 优化今天没展开，我认为 priority 低于以上这些。"

以「out-of-scope items checklist」wrap-up 是高级 signal：说明你知道自己设计的 boundary 在哪。

## 8. 常见流程 incident 与 rescue

| incident | rescue script |
|------|---------|
| requirements clarification 10 分钟还没完 | "差不多了，剩下的我边做边假设，write 在这边。" |
| interviewer 中途改需求 | 停笔 → "这影响 A 和 B，C 不受影响。我改这两处，然后继续。" （impression 影响分析 = 加分） |
| freezing up 30 秒+ | 说出来："我在 A 和 B 之间犹豫，差别是……我倾向 A。" freezing up 沉默才是 fatal 的。 |
| 时间不够 | "剩下 8 分钟，我想优先把 X 讲完，Y 和 Z 我一句话带过。" interviewer 爱死这种时间感。 |

## 9. practice method

1. **template internalize**：拿着 script templates 做 3 道题，之后扔掉 template
2. **recording retrospective**：手机录屏自己走全流程，重点看——有没有沉默 >10 秒、每个阶段实际用时、interviewer 视角能不能跟上
3. **live mock**：把 framework 练熟之后再上 [Interviewing.io](https://interviewing.io) 或朋友面，别浪费 live mock 在练 framework 上

## Next Module

→ [04 · Estimation & Numbers to Know](04-estimation-numbers.md)
