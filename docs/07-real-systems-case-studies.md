# 07 · Real-World System Case Studies

> The classic problems drill the "design process"; case studies fill in the "real-world constraints." One sentence in an interview — "Netflix actually does it this way, but their problem is X" — instantly separates you from someone who memorized the problem list.
> This chapter is organized as "read vs. write + what to remember + how to use it in an interview."

## 1. Why read real-world systems

Three reasons that pay off in mocks:
1. **Hard evidence for your trade-offs**: you say "we chose eventual consistency," and real-world systems hand you evidence at the level of "this is exactly how Netflix loses playback licenses"
2. **A reference frame for scale**: once you know Instagram ran 10M users on 12 Postgres machines, you'll stop frantically sharding a 1M DAU problem
3. **Ammunition for the deep dive**: when the interviewer asks "is this approach viable in reality?" you have a first-hand story

## 2. must-read cases (ranked by interview value)

### 2.1 Netflix (the benchmark for streaming media + elastic cloud)

- **read vs. write**: Netflix TechBlog's playback architecture and chaos engineering series; Evans & OMS (Open Connect CDN)
- **remember**:
  - **Eventually consistent** end to end, availability > consistency: showing stale recommendations or a stale license during service degradation is fine, and **playback never breaking** is the north star
  - Every service gets a fallback (a static response) — graceful degradation implemented in engineering terms
  - Chaos Monkey deliberately kills instances to validate redundancy — the ceiling of operational maturity impressions
- **interview usage**: in any "availability vs. consistency" discussion, "Netflix treats a playback interruption as its worst SLA violation, so the whole system is eventually consistent + degradation" is one sentence worth three paragraphs of argument

### 2.2 Uber (geolocation + real-time matching)

- **read vs. write**: Uber Engineering's H3 hexagonal grid, schemaless storage, marketplace matching
- **remember**:
  - GeoHash's precision/boundary problems → H3 hexagons: equal area, equidistant neighbors, no boundary seams
  - A write-heavy, read-heavy location stream (drivers report every 4 seconds) → their in-house Schemaless (KV sharding atop MySQL)
  - CAP trade-offs in matching: when supply and demand are unbalanced, matching quality degrades but the system doesn't deny service
- **interview usage**: a dimensionality-reduction weapon for geolocation problems (ride-hailing, food delivery, people nearby); "why hexagons and not squares" is the cleanest micro case study in trade-offs

### 3.3 Instagram (a small team carrying huge traffic)

- **read vs. write**: the classic blog post about 10M users on 12 machines; Instagram Engineering blogs
- **remember**:
  - Postgres sharding + application-layer routing (shard by user_id); playing tricks with Postgres features (counters via Redis INCR, photo ids sharded by photo_id with a secondary mapping so queries by user_id still work)
  - **Anti-pattern warning**: their 2012 technology selection (Cassandra was too new back then) proves that "boring technology + good sharding" goes far enough
- **interview usage**: the strongest argument for the "scale matching phase" — and a collection of cautionary tales about over-engineering

### 2.4 Meta (social graph + cache philosophy)

- **read vs. write**: TAO (Facebook's distributed graph storage), Facebook's cache layer papers, mcrouter/memcache architecture
- **remember**:
  - TAO: social graph reads are graph traversals (one-to-many fan-out reads) that a general-purpose DB can't do → a graph-specialized cache layer with a 95%+ hit rate
  - memcache + mcrouter's **regionalized cache** (regional pools; cross-DC reads hit local replicas, writes go to the home region)
  - Look-aside cache + the lease mechanism to resolve expiration races
- **interview usage**: the arsenal for Feed/social graph/cache deep dives all at once; the TAO papers take 20 minutes to read and are extremely high value

### 2.5 WhatsApp (radical simplicity)

- **read vs. write**: the "1M connections per server" blog post (Erlang, FreeBSD)
- **remember**: 50 engineers for 900M users; per-connection memory squeezed down to the KB range; Erlang's actor model maps naturally onto long-lived connection sessions
- **interview usage**: the capacity argument for chat system problems ("WhatsApp proved a single machine can hold 1M long-lived connections, so 10M connections starts at 10 machines")

### 2.6 classic papers (ranked by value per minute)

| Papers | Core Idea | Interview Coverage |
|------|---------|---------|
| **Amazon Dynamo** | Eventually consistent KV, consistent hashing, vector clock, hinted handoff | The parent of every storage problem |
| **Google Bigtable** | LSM, wide-column, partitioned tablets, Chubby | The principles behind Cassandra/HBase |
| **Google MapReduce / GFS** | The prototypes of batch processing and distributed file systems | Data pipeline problems |
| **Kafka papers** | The log as a messaging system | The source text for Q6 |
| **Spanner** | TrueTime, external consistency, globally distributed transactions | Strongly consistent multi-region problems |
| **Raft** | Understandable consistency consensus | When you're asked "how do you implement leader election?" |

> DDIA (Designing Data-Intensive Applications) chapters 5–6 cover almost every idea above; if you're short on time, read just those two chapters plus chapter 9 (consistency).

## 3. How to "use" cases in an interview

**Wrong usage** (recitation): "Netflix's blog said..." — it sounds memorized.

**Right usage** (argument):

> "Availability-first is a proven strategy in streaming media — Netflix is eventually consistent end to end, because a playback interruption is irreversible while a license arriving 10 seconds late is imperceptible. Our scenario is also 'reading stale data is imperceptible, a denial of service is fatal,' so I'd make the same trade-off."

The formula: **their constraint → their choice → does my constraint look like theirs → my choice**.

## 4. The 15-minutes-a-day case habit

1. Pick one post from High Scalability or an engineering blog
2. While reading, answer only three questions: How big is the scale? What's the hardest technical constraint? What did they sacrifice?
3. Store a one-line note in your repo: `cases.md` (your own case library — flipping through your notes before a mock is 10x faster than flipping through the originals)

## Next Module

→ [08 · Company Style Guide](08-company-guides.md)
