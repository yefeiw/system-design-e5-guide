# Q8 · 广告点击聚合器（Ad Click Aggregator）

> 难度：Medium–Hard ｜ 母题：写多 + 削峰 + 计费准确性
> Hello Interview 标注的 **E5 高频题**，覆盖流处理全家桶：去重、窗口、exactly-once、lambda/kappa。和 Q3 Top-K 是姊妹题但多了「钱」的维度。

## 1. 题目原文

"设计一个广告数据管道：收集广告展示（impression）和点击（click）事件，聚合出每个广告的点击率（CTR）、按维度出报表，并给计费系统提供准确的点击账单。"

## 2. 澄清问题清单

| 问题 | 意图 |
|------|------|
| 事件量级？延迟要求（计费要小时级，报表要天级？）| 决定流/批分层 |
| **点击怎么和展示关联**（click 晚到怎么办）？| 本题核心技术难点 |
| 重复点击（欺诈/重试）去重窗口？ | 计费准确性 |
| 报表维度有哪些（广告主×时间×地域）？| 维度建模 |
| 实时反欺诈要不要？ | 范围控制 |

## 3. 估算演示

```
1B impressions/day ≈ 12K/s 均值，峰值 60K/s；CTR 1% → 10M clicks/day
原始事件（保留 30 天用于审计/重算）：1.1B × 200B × 30 ≈ 6.6TB —— 对象存储
聚合后（广告×小时×维度）：GB 级 —— OLAP 库轻松
结论 → 两个存储世界：原始事件进湖（不可变、可重算），
       聚合结果进服务存储（低延迟查询）。架构必然是流批分层
```

## 4. 高层设计

```
SDK/前端 ──▶ 事件采集 API（校验、打标）──▶ Kafka（impressions / clicks 各自 topic）
                                              │
             ┌────────────────────────────────┤
             ▼                                ▼
      实时流（Flink/Beam）                批层（夜间 Spark）
      · 去重（click 对 impression）        · 全量重算精确值
      · 会话窗口关联（click↔imp）          · 审计对账
      · 分钟级预聚合                        │
             │                                │
             ▼                                ▼
      Redis / 内存OLAP（实时预览）      OLAP（ClickHouse/BigQuery，报表）
             │                                │
             └────────────► 查询服务 ◄────────┘
                                │
                                ▼
                      计费系统（以批层精确值为准）
```

**叙事主线**："实时层给运营看（分钟级、近似、可重算），批层给钱（小时/天级、精确、审计）——**同一份数据两条时间线**，这是 lambda 的教科书场景。"

## 5. 数据模型

```
事件（不可变，JSON→Parquet 入湖）:
  impression: {imp_id, ad_id, user_id, page, geo, ts}
  click:      {click_id, imp_id(FK), ad_id, ts}

聚合表（OLAP，按 (ad_id, hour, dim...) 聚簇）:
  ad_stats_hourly (ad_id, ts_hour, geo, imp_count, click_count, ctr)
去重状态（流层，RocksDB/Flink state）: click_id → TTL 窗口
```

**关键点**：click 里带 `imp_id` 外键——"关联不是靠事件时间相近去猜，是靠 ID 外键精确关联；**事件顺序不可信，ID 关系才可信**。"

## 6. 深挖

### 深挖 A · Click 和 Impression 的乱序关联（本题的 E5 拷问点）

问题：click 事件先到、impression 后到（不同端、不同链路延迟）怎么办？

- **事件时间会话窗口**：click 和 impression 按 `imp_id` keyBy 到同一流分组，窗口内 join；乱序靠 **watermark**（如允许 5 分钟迟到）
- 超过 watermark 的迟到 click → side output → 落湖，批层收编
- "CTR 的分子分母必须来自**同一时间窗的同一口径**——流层 5 分钟 watermark 内的 join 算实时口径，全天重算算最终口径，两者永远有差异，报表上要标注。"（口径意识 = 数据系统的 senior 信号）

### 深挖 B · 去重与反欺诈（计费的钱维度）

- 重复点击：`click_id` 幂等 + 窗口内 `user_id+ad_id` 频次规则（连点 10 次只计 1）
- 状态存储：流层本地 RocksDB state（不进外部 Redis，避免状态双写不一致）；state TTL = 去重窗口
- 欺诈信号（提一句展示广度）：IP 段聚集、无 mousemove、点击/展示时距分布异常 → 规则引擎打标 → **计费侧剔除**
- "计费的单位是'有效点击'——去重和反欺诈规则必须**在计费之前**完成，这就是流层存在的理由，不只是为了好看的大盘。"

### 深挖 C · Exactly-once 落到 OLAP

- 流层计算 exactly-once：Flink checkpoint（两阶段提交 barrier）+ sink 事务性写
- **重放能力是一切的兜底**：Kafka retention 之外原始事件全量落湖——"流层挂了/逻辑改了，从湖里重放重算，**管道是可重跑的程序而不是一次性数据流**"（kappa 的可重放思想 + lambda 的批精确层，混合即答案）
- 幂等写 OLAP：以 (ad_id, hour, dim) 为主键 upsert 覆盖

### 深挖 D · 洪峰与背压

- 大促/超级碗时刻流量 ×50：采集 API 丢弃策略分级（计费事件绝不丢、遥测可采样）
- Kafka 做缓冲，消费者按能力拉取；**背压传导**到采集端返回 503+Retry-After 让 SDK 退避
- "丢什么的优先级是产品决策不是技术决策——我会把三类事件的 SLA 写成表让业务签字。"

## 7. 红牌答案

- 只有实时层没有批层（计费不能靠可能重复/丢失的流）
- click 和 impression 靠时间戳就近匹配
- 没有去重讨论（重复点击直接进账单 = 计费事故）
- 聚合结果直接写 MySQL 供报表（维度聚合查询会打爆 OLTP）
- 没有迟到数据/watermark 概念

## 8. 一分钟电梯版

"两层数据流：Kafka 承接原始事件（impression/click 双 topic，click 带 imp_id 外键精确关联而非时间猜匹配）；实时层 Flink 做去重（click_id 幂等 + 用户频次规则）、watermark 乱序 join、分钟级预聚合进内存 OLAP 给运营看；批层夜间全量重算落 ClickHouse，**计费以批层为准**。原始事件全量入湖保证可重放——流层挂了从湖重跑。洪峰时按事件分级丢弃（计费事件永不丢）。实时和最终数字永远有差，报表标注口径，这就是 lambda 的代价与收益。"

---
← [Q7 票务预订](07-ticket-booking.md) ｜ [返回 06 总览](../06-classic-questions-overview.md)
