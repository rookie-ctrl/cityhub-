# Redis 五大应用场景速查表(代码级)

> 数据结构和 key 名均以 `RedisConstants.java` 为准。每个场景按「数据结构 → key 设计 → 核心逻辑 → 面试要点/追问」整理。

## 总览

| 场景 | 数据结构 | Key 设计 | 核心命令 | 代码位置 |
|---|---|---|---|---|
| 点赞排行榜 | ZSet | `blog:liked:{blogId}` member=userId score=时间戳 | ZSCORE / ZADD / ZREM / ZRANGE | `BlogServiceImpl.updateLike` `queryBlogLikes` |
| 关注 & 共同关注 | Set | `follows:{userId}` member=被关注者id | SADD / SREM / SINTER | `FollowServiceImpl.follow` `followCommons` |
| 关注推送 Feed 流 | ZSet(收件箱) | `feed:{粉丝id}` member=blogId score=时间戳 | ZADD / ZREVRANGEBYSCORE | `BlogServiceImpl.saveBlog` `quertBlogOfFollow` |
| 附近商户 | GEO | `shop:geo:{typeId}` member=shopId | GEOSEARCH | `ShopServiceImpl.queryShopByType` |
| 签到统计 | BitMap | `sign:{userId}:{yyyyMM}` offset=天数-1 | SETBIT / BITFIELD | `UserServiceImpl.sign` `signCount` |
| UV 统计 | HyperLogLog | (README 提及,代码未实现) | PFADD / PFCOUNT | — |

---

## 1. 点赞排行榜 — ZSet(`BlogServiceImpl.java:73`)

**核心逻辑**(`updateLike`):
1. `ZSCORE key userId` 判断是否已点赞(score==null 即未点赞)
2. 未点赞 → DB `liked=liked+1` 成功后再 `ZADD key 时间戳 userId`
3. 已点赞 → DB `liked=liked-1` 成功后再 `ZREM`
4. 查 Top5(`queryBlogLikes`)→ `ZRANGE key 0 4` → `IN (ids)` 查 DB,用 **`order by field(id, ...)` 保持 Redis 返回的顺序**

**面试要点**:
- 为什么 score 用 `System.currentTimeMillis()`:实现"按点赞时间倒序"
- ZSet 底层是**跳表**(跳跃表),增删改查 O(logN),范围查询快;追问:和红黑树对比(跳表实现简单、范围查找友好、Redis 并发友好)
- 为什么 DB 和 Redis 都存:DB 持久化兜底 + 页面总数展示,Redis 做实时排行
- 一致性顺序:先改 DB 成功再动 Redis(DB 失败就整体失败);追问:DB 成功 Redis 失败怎么办 → 短暂不一致,点赞数是弱一致场景
- **`order by field` 的坑**:MySQL `IN` 查询不保证返回顺序,必须显式 FIELD 排序,这是面试常挖的细节

## 2. 共同关注 — Set(`FollowServiceImpl.java:40`)

**核心逻辑**:
- 关注:`save` DB → `SADD follows:{userId} followUserId`(DB 成功才写 Redis)
- 取关:DB `remove` → `SREM`
- 共同关注(`followCommons`):`SINTER follows:{A} follows:{B}` → `listByIds` 返回

**面试要点**:
- Set 特点:无序、去重、O(1) 判存在
- SINTER 复杂度 O(N×M);追问:N、M 很大怎么办(数据倾斜、拆 key、用 Bloom 预筛)——能说出思路即可
- 为什么要 Redis 冗余一份:DB 也能 JOIN 出共同关注,但全量集合运算在 DB 是灾难;Redis 求交集是它的杀手锏场景
- 追问:关注关系数据一致性(DB 成功 Redis 失败)→ 同上,弱一致可接受

## 3. 关注推送 Feed 流(收件箱模式)— ZSet(`BlogServiceImpl.java:124`)

**⭐ 这是代码里有、README 没写的隐藏亮点,面试讲出来是加分项**

**核心逻辑**:
- **写扩散(Push)**:发笔记时查 `tb_follow` 拿到所有粉丝 → 循环把 blogId 写进每个粉丝的收件箱 `feed:{粉丝id}`,score=时间戳(`saveBlog:134-142`)
- **读**:`ZREVRANGEBYSCORE key min max offset 2`(倒序取 2 条)→ 滚动分页
- **滚动分页的 offset 设计**(`quertBlogOfFollow:160-173`):score 相同时 offset+1,否则 offset 重置为 1——防止两条笔记时间戳相同导致翻页丢数据

**面试要点**:
- 为什么用 ZSet 不是 List:按时间倒序 + 支持按分数范围滚动分页
- Push 模式(收件箱) vs Pull 模式(发件箱):Push 读快写慢,Pull 写快读慢
- **必被追问**:大 V 几百万粉丝,发一条笔记循环写几百万次收件箱,怎么办?
  - 答:Push/Pull 混合模式——活跃用户 Push,不活跃用户 Pull(读时现算);或粉丝分级、异步化。这是区分度很高的加分讨论
- 追问:滚动分页为什么不用传统 `page/limit`(深分页性能 + 数据变化导致重复/遗漏)

## 4. 附近商户 — GEO(`ShopServiceImpl.java:247`)

**核心逻辑**(`queryShopByType`):
1. 请求带坐标 (x,y) 且带 typeId → `GEOSEARCH shop:geo:{typeId} FROMLONLAT x y BYRADIUS 5000 m ASC`(代码用 `GeoReference.fromCoordinate` + `Distance(5000)` + `limit(end)`)
2. `includeDistance` 返回距离 → 内存 `skip(from)` 实现分页
3. 按返回顺序查 DB(`order by field` 保序)→ 回填 distance 字段

**面试要点**:
- GEO 底层是 **GeoHash**:经纬度 → 二进制编码 → base32 字符串,前缀相同的格子距离近
- GeoHash 的两个缺陷:① 边界问题(相邻很近的点可能编码前缀完全不同)→ 解决:查询时把周围 8 个格子一起查;② 精度与字符串长度有关(Redis 内部用 52 位整数编码,精度约 1 米内)
- Redis GEO 本质是 ZSet 的封装(score 是 geohash 编码值),所以 `ZREM` 也能删坐标
- Redis 6.2 的 `GEOSEARCH` 取代了旧 `GEORADIUS`/`GEORADIUSBYMEMBER`(你的代码用的就是 GEOSEARCH,可以主动提)
- 追问:为什么不在 MySQL 算距离 → 全表扫 + 三角函数,数量大必挂;GEO 索引查询 O(logN)

## 5. 用户签到 — BitMap(`UserServiceImpl.java:123`)

**核心逻辑**:
- 签到(`sign`):`SETBIT sign:{userId}:{yyyyMM} (dayOfMonth-1) 1`
- 连续签到天数(`signCount`):`BITFIELD key GET u{dayOfMonth} 0`(无符号读当月截止今天的位)→ 循环 `num & 1` 判断末尾位是否为 1,`num >>>= 1` 右移,数连续 1 的个数

**面试要点**:
- 空间收益:1 个用户 1 年签到只需 365bit ≈ 46 字节,MySQL 要 365 行
- 为什么用无符号读(`u`):防止最高位被当成符号位
- `& 1` + `>>> 1` 的位运算统计连续签到是手写亮点,这段代码(158-175 行)建议能默写
- 追问:统计全站某天签到人数 → 按天分 key 反过来设计,或 `BITCOUNT`;BitMap 稀疏时浪费 → 引出 Roaring Bitmap(可选加分)

## 6. UV 统计 — HyperLogLog(⚠️ 仅 README 提及,无代码)

讲的时候定位成"设计选型":百万级 UV,误差允许 0.81% 内,12KB 固定内存搞定(不随数据量增长)。追问:原理(概率算法,基于 Bernoulli 试验 + 调和平均);和 BitMap 比:精确但要存 userId 集合,亿级用户内存爆炸。

---

## 面试速记表(一句话版)

- **ZSet**:点赞排行榜,score 放时间戳实现按时间排序;Top5 用 ZRANGE,查库后 FIELD 保序
- **Set**:共同关注,SINTER 求交集——集合运算是 Redis 杀手锏
- **ZSet(收件箱)**:Feed 流 Push 模式写扩散,滚动分页靠 offset 防重;大 V 场景引出 Push/Pull 混合
- **GEO**:GeoHash 编码的 ZSet,GEOSEARCH 查 5km;边界问题查周围 8 格
- **BitMap**:签到,365 天 46 字节;BITFIELD + 位运算数连续签到
- **HyperLogLog**:UV 统计,12KB 换 0.81% 误差
