# 模块 05 · Redis 深度准备

> **岗位关联**：JD 里"熟悉 MySQL、Redis 等主流数据库"是**并列的两个场景**，面试官通常会连着问——先问完 MySQL 的索引和事务，紧接着问"那 Redis 呢？"。这个问题背后真正想确认的是：**你知不知道什么时候该用 Redis、什么时候不该用**。OA/客服类系统对 Redis 的典型用法是：会话与登录态、组织权限与部门树缓存、待办数量计数、分布式锁（防止审批重复提交 / 定时任务重复执行）、接口限流、以及首页统计的排行榜。这些场景恰好覆盖了 Redis 的**五大核心考点**：数据结构选型、持久化、高可用、缓存三大问题、分布式锁与一致性。
>
> **你的独特优势**：简历里 **LoveSpouse 的金币计费**是纯 Redis 场景（原子扣减 + 幂等 + 跨实例通话控制），**FlashRoot 的返点结算**是典型的"计数 + 结算"场景，**POS 数据贯通的 RabbitMQ 消费**涉及幂等与重复消费防护。这三段经历足以支撑"我不只是用过 Redis，我是在**正确性和并发**上用过 Redis"的叙事。**这是本模块的答题主线——把每个八股考点都落到一个真实做过的决策上。**
>
> **准备策略**：和 04-MySQL 一样按「原理（讲到源码结构）→ 高频 Q&A 参考回答 → 结合项目的话术」组织。**Redis 的面试特点是"追问密度极高"**：说"Redis 快"会被追问为什么快，说"单线程"会被追问 6.0 为什么多线程，说"跳表"会被追问为什么不用红黑树，说"分布式锁"会被追问锁过期怎么办、RedLock 行不行。所以本模块的原理部分重点是**把每个"为什么"都答到第二层和第三层**。
>
> **时间分配建议**：数据结构与单线程模型 15 分钟、持久化 10 分钟、高可用 10 分钟、缓存三大问题 + 一致性 15 分钟、分布式锁与限流 10 分钟。

---

## 目录

- [1. 数据结构与底层实现](#1-数据结构与底层实现)
- [2. 持久化：RDB / AOF / 混合持久化](#2-持久化rdb--aof--混合持久化)
- [3. 高可用：主从 / 哨兵 / Cluster](#3-高可用主从--哨兵--cluster)
- [4. 缓存三大问题与缓存一致性（含 Go 代码）](#4-缓存三大问题与缓存一致性含-go-代码)
- [5. 应用场景：分布式锁 / 限流 / 计数与排行榜](#5-应用场景分布式锁--限流--计数与排行榜)
- [6. 精选 18 道高频面试题与参考回答](#6-精选-18-道高频面试题与参考回答)
- [7. 大 key 与热 key 专题](#7-大-key-与热-key-专题)
- [8. 速记表与自测清单](#8-速记表与自测清单)

---

## 1. 数据结构与底层实现

### 1.0 全局视角：Redis 的一个 key 到底长什么样

在讲每个数据结构之前，先建立一个**全局的存储模型**，这是很多八股文不讲但面试官很爱问的部分（"Redis 的 key 和 value 都存在哪里？"）。

Redis 的顶层数据结构是 `redisDb`（源码 `server.h`）：

```c
typedef struct redisDb {
    dict *dict;                 /* 键空间：存所有 key → value(redisObject) */
    dict *expires;              /* 过期字典：key → 过期时间戳（毫秒） */
    dict *blocking_keys;        /* BLPOP 等阻塞命令 */
    dict *ready_keys;
    dict *watched_keys;         /* WATCH 命令监视的 key */
    int id;                     /* 数据库编号 0~15 */
    long long avg_ttl;
    ...
} redisDb;
```

也就是说：**Redis 的所有 key 存在一个全局哈希表 `dict` 里，每个 value 是一个 `redisObject`**。

`redisObject` 是 Redis 对象系统的核心（源码 `server.h`，**16 字节**）：

```c
typedef struct redisObject {
    unsigned type:4;        /* 4 bit：对象类型 —— STRING/LIST/HASH/SET/ZSET/STREAM */
    unsigned encoding:4;    /* 4 bit：底层编码 —— int/embstr/raw/ziplist/listpack/quicklist/... */
    unsigned lru:24;        /* 24 bit：LRU 时间戳 或 LFU 计数（由 maxmemory-policy 决定）*/
    int refcount;           /* 4 字节：引用计数（用于对象共享，如 0~9999 的整数） */
    void *ptr;              /* 8 字节：指向真正的底层数据结构 */
} robj;   /* 合计 4bit+4bit+24bit = 4字节，+4字节 refcount + 8字节 ptr = 16 字节 */
```

**记住这个 16 字节很重要**——后面讲 `embstr` 为什么以 44 字节为界，就要用到它。

**`encoding` 是理解 Redis 内存优化的钥匙**：Redis 会**根据数据的大小和数量自动选择最省内存的编码**，并在超过阈值时**自动升级**（但**不会自动降级**，这是常被忽略的点）。这张表是面试必背：

| 类型 | 编码（encoding） | 切换条件（默认） | 相关的配置 |
|---|---|---|---|
| **STRING** | `int` | 值为整数且可用 long 表示 | — |
| | `embstr` | 长度 ≤ **44** 字节 | — |
| | `raw` | 长度 > 44 字节 | — |
| **LIST** | `listpack`（7.0+）/ `ziplist` | 元素数 ≤ 128 且单个元素 < 64 字节 | `list-max-listpack-size` / `list-max-ziplist-size` |
| | `quicklist` | 超出时（其实是 quicklist 包裹多个 listpack/ziplist 节点） | |
| **HASH** | `listpack` / `ziplist` | 字段数 ≤ 128 且所有 value < 64 字节 | `hash-max-listpack-entries` / `hash-max-listpack-value` |
| | `hashtable` | 超出时 | |
| **SET** | `intset` | 全是整数且元素数 ≤ 512 | `set-max-intset-entries` |
| | `listpack`（7.2+） | 元素数 ≤ 128 且元素 < 64 字节 | `set-max-listpack-entries` |
| | `hashtable` | 超出时 | |
| **ZSET** | `listpack` / `ziplist` | 元素数 ≤ 128 且元素 < 64 字节 | `zset-max-listpack-entries` / `-value` |
| | `skiplist` + `dict` | 超出时 | |

> **面试话术**：**"小数据量用紧凑编码（listpack/ziplist/intset），省内存但操作是 O(n)；超过阈值转成标准结构（hashtable/skiplist），操作 O(1)/O(log n) 但内存占用高。"** 这个"阈值触发转换"的设计是 Redis 在**内存和性能之间做的自动权衡**。
>
> **踩坑提示（很实战）**：这个阈值在**生产上经常需要调**。比如存用户标签的 Hash，如果只有 100 个字段（"用户数 × 字段数" 得出），在小规模下是 listpack，一旦某个用户贴了第 129 个标签，**整个 Hash 会转成 hashtable，内存占用可能翻好几倍**（hashtable 每个 entry 要 2 个指针 + dictEntry 结构）。所以大 Hash 场景要把 `hash-max-listpack-entries` 调大（比如 1000）。反过来，如果 Hash 很大且字段访问很随机，保持 listpack 反而会让每次访问都 O(n) 扫描。

### 1.1 String：SDS（简单动态字符串）

#### SDS 的结构（源码 `sds.h`）

C 语言原生字符串是 `char *` + `\0` 结尾。Redis 自己实现了 SDS：

```c
/* 以 sdshdr8 为例（8 表示 len/alloc 用 8 位存） */
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t  len;      /* 1 字节：已使用的字节数（不含 '\0'）*/
    uint8_t  alloc;    /* 1 字节：已分配的总字节数（不含 '\0' 和 header）*/
    unsigned char flags; /* 1 字节：低 3 位标识类型（SDS_TYPE_5/8/16/32/64）*/
    char buf[];        /* 柔性数组：真正的字符数据 */
};
/* 另外还有 sdshdr16/32/64，len/alloc 用 2/4/8 字节，
   根据字符串长度自动选择最小够用的 header，节省空间 */
```

**注意 `len` 和 `alloc` 是两个不同的值**：`len` 是实际长度，`alloc` 是容量。**`len < alloc` 意味着有未使用的预留空间** —— 这就是 SDS 的"空间预分配"。

#### 为什么不用 C 字符串？（五个理由，面试必答）

| # | C 字符串的问题 | SDS 的解决 |
|---|---|---|
| 1 | **获取长度是 O(n)**（要遍历到 `\0`） | `len` 字段直接读，**O(1)** → `STRLEN` 是 O(1) |
| 2 | **非二进制安全**：字符串里不能有 `\0`（会被当结尾截断） | **靠 `len` 判断结尾，不靠 `\0`** → 可以存图片、序列化对象、任意字节 |
| 3 | **缓冲区溢出**：`strcat` 不检查空间 | 拼接前检查 `alloc - len`，不够就扩容 |
| 4 | **每次修改都要重新分配内存**（改长要扩容、改短要释放） | **空间预分配 + 惰性释放**（见下） |
| 5 | 不能兼容部分 C 字符串函数 | `buf` 结尾仍然保留 `\0`，可以复用 `<string.h>` 的部分函数 |

#### 空间预分配与惰性释放（细节考点）

**预分配策略**（`sdsMakeRoomFor`）：

```
需要扩容时：
  若 newlen < 1MB  →  alloc = 2 × newlen        （翻倍，多给一倍）
  若 newlen >= 1MB →  alloc = newlen + 1MB      （每次多给 1MB，避免翻倍浪费）
```

**惰性释放（lazy free）**：缩短字符串时**不立即释放内存**，只改 `len`，把空间留着以备将来使用（`sdsclear`、`sdsrange` 等）。

**目的**：**把 N 次连续追加操作的内存重分配次数从"最多 N 次"降到"最多 N 次中的少数几次"**。这是典型的**用空间换时间**。

> **面试延伸**："`APPEND` 命令的性能怎么样？" → 由于预分配，**`APPEND` 在大多数情况下是 O(1)（摊还）**，而不是每次 O(n)。但如果字符串超过 1MB，每次追加最多多分配 1MB，此时**大量 APPEND 会导致内存膨胀**（最坏情况下 alloc 会远大于 len），所以生产上要避免对超大 key 频繁 APPEND。

#### 三种编码：int / embstr / raw（**44 字节的由来**）

```bash
127.0.0.1:6379> SET k1 12345
127.0.0.1:6379> OBJECT ENCODING k1
"int"        # 值能转成 long 且不超范围 → 直接存在 redisObject 的 ptr 里（不额外分配）

127.0.0.1:6379> SET k2 "hello"
127.0.0.1:6379> OBJECT ENCODING k2
"embstr"     # 长度 <= 44 → redisObject 和 SDS 一次性连续分配

127.0.0.1:6379> SET k3 "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # 52 字节
127.0.0.1:6379> OBJECT ENCODING k3
"raw"        # 长度 > 44 → redisObject 和 SDS 分两次分配
```

**为什么 44 字节？**（超高频追问，一定要能算）

```
jemalloc 的内存分配档位是：8/16/32/48/64/80/96...
  —— 也就是说小于 64 字节的分配都会占用 64 字节的"内存块"

embstr 要一次分配"redisObject + sdshdr + buf + '\0'"这一整块：
  redisObject            = 16 字节
  sdshdr8 (len+alloc+flags) = 3 字节
  buf 末尾的 '\0'         = 1 字节
  ------------------------------------
  固定开销 = 20 字节

剩余给字符串内容的 = 64 - 20 = 44 字节  ✅
```

**所以 44 不是一个随便定的数字，而是 jemalloc 64 字节内存块减去结构开销的结果。**

**`embstr` vs `raw` 的关键区别（面试爱问）**：

| | embstr | raw |
|---|---|---|
| 内存分配次数 | **1 次**（连续内存） | 2 次 |
| 内存是否连续 | ✅ 是（缓存友好） | ❌ 否（多一次指针跳转） |
| 是否只读 | **✅ 是** —— **任何修改都会先转成 raw 再改** | 可修改 |
| 适用 | 短字符串（key、短值） | 长字符串 |

> **面试点**："`APPEND` 一个 embstr 会发生什么？" → **embstr 是只读的**（因为它是和 redisObject 连续分配的，没有预留空间做 append），所以任何修改操作都会**先把编码转成 raw**，再执行修改。所以 `SET k "short"` 后再 `APPEND k "x"`，`OBJECT ENCODING` 会变成 `raw`。**这是"编码只升不降"的一个具体例子。**

### 1.2 List：从 ziplist+linkedlist 到 quicklist 到 listpack

#### 演进史（面试讲出这个演进，说明你了解 Redis 的发展）

| 版本 | List 的底层实现 |
|---|---|
| 3.0 之前 | `ziplist`（元素少）或 `linkedlist`（双向链表，元素多） |
| **3.2 ~ 6.2** | **`quicklist`**（双向链表 + 每个节点是一个 ziplist） |
| **7.0+** | **`quicklist` + 节点用 `listpack`**（彻底移除 ziplist） |

#### quicklist 的结构（源码 `quicklist.h`）

```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* 所有节点里的元素总数 */
    unsigned long len;          /* quicklistNode 的个数 */
    int fill : QL_FILL_BITS;    /* 单个节点的容量限制（即 list-max-listpack-size）*/
    unsigned int compress : QL_COMP_BITS;  /* 两端不压缩的节点数（LZF 压缩）*/
    ...
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *entry;       /* 指向 ziplist / listpack */
    size_t sz;
    unsigned int count : 16;    /* 本节点里的元素个数 */
    ...
} quicklistNode;
```

**结构示意**：

```
quicklist
   ↓
[节点1: listpack(元素1..N)] <-> [节点2: listpack(元素N+1..M)] <-> [节点3: ...]
        ↑ 每个节点内部是紧凑存储（省内存、缓存友好）
                              ↑ 节点之间是双向链表（插入删除 O(1)）
```

**为什么这么设计？（面试核心）**：**单纯的链表内存开销太大**（每个元素要 prev/next 两个指针 + 一个单独分配的节点头，64 位系统下至少 24~32 字节/元素，而元素本身可能只有 8 字节）；**单纯用 ziplist 又太大**（一个几万元素的 ziplist 是连续内存，插入删除要整体 memmove，O(n) 且会触发大块内存重分配）。

**quicklist 是两者的折中**：**分段连续存储** —— 段内紧凑（省内存、缓存友好），段间用链表（插入删除不用搬移整块数据）。

**容量限制参数** `list-max-listpack-size`（3.2~6.x 叫 `list-max-ziplist-size`）：

```bash
# 正数：限制单个节点里的元素个数
list-max-listpack-size 128     # 每个节点最多 128 个元素

# 负数（推荐）：限制单个节点的字节大小
# -1: 4KB   -2: 8KB(默认)   -3: 16KB   -4: 32KB   -5: 64KB
list-max-listpack-size -2
```

> **为什么推荐用字节大小而不是元素个数？** 因为元素大小差异很大（一个元素可能是 10 字节也可能是 1KB），按个数限制会导致节点大小不可控。**按字节大小限制（-2 = 8KB）能让每个节点的内存占用可预测**。

**压缩参数** `list-compress-depth`：quicklist 支持对**中间的节点**做 LZF 压缩（首尾节点不压缩，因为最常被访问）：

```bash
list-compress-depth 0    # 不压缩（默认）
list-compress-depth 2    # 首尾各 2 个节点不压缩，中间的压缩
```

#### listpack（Redis 7.0 引入，为什么取代 ziplist？）

**ziplist 的致命缺陷：级联更新（cascade update）**。

ziplist 里每个 entry 的头部有 `prevlen` 字段记录前一个 entry 的长度：

```
[entry1: prevlen=0, len=5, "hello"] [entry2: prevlen=7, len=..., ...]
```

- 如果前一个 entry 的长度 < 254，`prevlen` 占 **1 字节**；
- 如果 ≥ 254，`prevlen` 占 **5 字节**。

**问题**：假设一个 ziplist 里有 100 个都是 253 字节的 entry。在**头部插入一个新 entry 导致第一个 entry 的 prevlen 从 1 字节变成 5 字节**，于是第二个 entry 的 `prevlen` 也要更新……**连锁反应，整条 ziplist 都要重写**。最坏情况是 O(n²)。

**listpack 的解法**：**只记录"当前 entry 的长度"，不记录"前一个 entry 的长度"**。每个 entry 是 `[encoding + data + backlen]`，其中 `backlen` 记录的是**本 entry 的总长度**，这样**可以从右往左遍历**（从末尾开始，读 backlen 就能定位到前一个 entry 的起点）。

**关键收益**：**修改一个 entry 不影响其他 entry 的 `backlen`**（因为它记录的是自己的长度），**彻底消除了级联更新**。

> **面试话术**：**"listpack 相比 ziplist 的核心改进是消除了级联更新——ziplist 存的是前一个 entry 的长度，导致修改一个 entry 可能引发连锁的头部扩张；listpack 存的是自己的长度，每个 entry 自描述，改一个不牵连别人。"** 这个回答能直接体现你读过源码。

### 1.3 Hash：listpack / hashtable

```bash
# 小 Hash：listpack（紧凑，O(n) 查找但 n 很小、内存连续，实际很快）
127.0.0.1:6379> HSET user:1 name tom age 25
127.0.0.1:6379> OBJECT ENCODING user:1
"listpack"

# 超过阈值：转成 hashtable
127.0.0.1:6379> CONFIG SET hash-max-listpack-entries 2
127.0.0.1:6379> HSET user:2 name tom age 25 city sz
127.0.0.1:6379> OBJECT ENCODING user:2
"hashtable"
```

**listpack 里 Hash 的存储方式**：**key 和 value 是成对相邻存放的**：

```
[listpack: "name" | "tom" | "age" | "25" | "city" | "sz"]
              ↑key    ↑value   ↑key   ↑value
```

所以查找某个 field 就是**遍历 listpack 的奇数位置**，O(n)。

**转 hashtable 的时机（源码 `t_hash.c` 的 `hashTypeTryConversion`）**：

```c
if (hashTypeLength(o) > server.hash_max_listpack_entries ||   // 字段数超限
    sdslen(value) > server.hash_max_listpack_value)           // 单个 value 超长
{
    hashTypeConvert(o, OBJ_ENCODING_HT);
}
```

**注意**：**只有"添加新字段"时才检查是否转换**（修改已有字段不会触发转换，因为大小不会因此增长到超限）。而且**转换是不可逆的**——即使你后来删掉了大部分字段，编码**仍然是 hashtable**，不会降回 listpack。这是"编码只升不降"的又一个例子，**也是内存泄漏感的来源**（明明只有 3 个字段却占了 hashtable 的内存）。

> **生产建议**：如果一个 Hash 会先大后小（比如临时聚合），**宁可一开始就用多个小 key 或者定期重建**（`DEL` 后重新 `HSET`），也不要让它卡在 hashtable 编码上。

### 1.4 Set：intset / listpack / hashtable

```bash
# 全整数且元素少 → intset（有序数组，二分查找 O(log n)）
127.0.0.1:6379> SADD nums 1 2 3 4
127.0.0.1:6379> OBJECT ENCODING nums
"intset"

# 加入非整数 → 升级为 listpack 或 hashtable
127.0.0.1:6379> SADD nums abc
127.0.0.1:6379> OBJECT ENCODING nums
"listpack"    # Redis 7.2+；7.0 及之前直接是 hashtable
```

**intset 的结构**（源码 `intset.h`）：

```c
typedef struct intset {
    uint32_t encoding;   /* INTSET_ENC_INT16/32/64 —— 所有元素用统一的编码 */
    uint32_t length;
    int8_t contents[];   /* 有序数组，元素按升序排列 */
} intset;
```

**特点**：
- **有序**（所以支持 `SRANDMEMBER`、以及集合求交的高效归并算法）；
- **二分查找**，`SISMEMBER` 是 O(log n)；
- **编码升级（upgrade）**：加入一个超出当前编码范围的整数时（比如 INTSET_ENC_INT16 里加入 100000），**整个 intset 会升级到 INT32**，需要**重新分配内存并搬移所有元素**（O(n)）。**注意：不支持降级**——即使后来删掉了那个大数，仍然保持 INT32。

> **面试点**："`SADD` 一个很小的整数到 intset，会有什么开销？" → 如果触发编码升级，是 O(n) 的。虽然 Redis 文档说 `SADD` 是 O(1)，但**在触发 intset 升级时是 O(n)**，这是一个容易被忽略的复杂度陷阱。**大数据量下批量写入大量 intset 要注意这一点。**

### 1.5 ZSet：跳表（skiplist）+ 字典（dict）—— **本节最重要的数据结构**

#### 为什么需要两个结构？

ZSet 需要同时支持两类操作，而**没有单一结构能同时高效满足**：

| 操作 | 需要的结构 | 复杂度 |
|---|---|---|
| `ZSCORE key member`（按成员查分数） | **哈希表** | O(1) |
| `ZRANGE` / `ZRANK`（按分数排序、范围查询、排名） | **跳表** | O(log n) |

**所以 Redis 的解法是：两个都上，用指针共享同一个 `sds` 成员对象**（不重复存储 member 字符串，只是两个结构都指向它）：

```c
typedef struct zset {
    dict *dict;          /* member → score，用于 ZSCORE 等 O(1) 查询 */
    zskiplist *zsl;      /* 按 score 排序的跳表，用于范围查询和排名 */
} zset;
```

> **面试话术**：**"ZSet 是'跳表 + 哈希表'的组合：哈希表负责 O(1) 的按成员查分，跳表负责 O(log n) 的范围查询和排名。两者共享同一个成员对象，不重复存字符串。这是典型的'用空间换两个维度的高效'的设计。"**

#### 跳表的结构（源码 `server.h`）

```c
#define ZSKIPLIST_MAXLEVEL 32   /* 最大层数 —— 足够支撑 2^64 个元素 */
#define ZSKIPLIST_P 0.25        /* 层高晋升概率 */

typedef struct zskiplistNode {
    sds ele;                        /* 成员 */
    double score;                   /* 分数（排序依据）*/
    struct zskiplistNode *backward; /* 后退指针（只能指向相邻的前一个节点，用于反向遍历）*/
    struct zskiplistLevel {
        struct zskiplistNode *forward;  /* 前进指针 */
        unsigned long span;             /* 跨度：到下一个节点的"距离"，用于 O(log n) 算排名 */
    } level[];                      /* 柔性数组：每个节点有 1~32 层 */
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;   /* 节点数 */
    int level;              /* 当前最大层数 */
} zskiplist;
```

**结构示意**（层数随机生成，p=0.25）：

```
level 4:  header ──────────────────────────────────→ [score=50]
level 3:  header ──────────────→ [score=20] ────────→ [score=50]
level 2:  header ──────→ [score=10] → [score=20] ──→ [score=50] → NULL
level 1:  header → [5] → [10] → [20] → [35] → [50] → [60] → NULL
                    ↑ 底层是完整的有序链表（范围查询就靠它顺序扫描）
          ←←←←←←←←←←← backward 指针（反向遍历）
```

**两个容易被忽略的设计细节（面试加分项）**：

1. **`span`（跨度）字段**是为了支持 **`ZRANK`（O(log n) 算排名）**。查找时把经过的节点的 `span` 累加，就得到了排名。**没有 span 的话，算排名就只能从头遍历，O(n)。**
2. **`backward` 指针只有在 level 1（最底层）才有完整链表**，上层没有反向指针。所以跳表**只能高效地正向范围查询**（`ZRANGE`），反向范围（`ZREVRANGE`）需要借助 `tail` 指针从后往前沿 `backward` 遍历。

**层高的生成（`zslRandomLevel`）**：

```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))  /* 25% 的概率 */
        level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**期望层高 = 1/(1-p) = 1/0.75 ≈ 1.33 层**，所以**平均每个节点只多消耗 1.33 个指针**，内存开销很小。而**平均查找长度是 O(log₁/ₚ n) = O(log₄ n)**，比二叉搜索树（log₂ n）略差一点，但**常数因子小、实现简单、无旋转操作**。

#### 为什么 ZSet 用跳表而不用红黑树/B+ 树？（**超高频，必答**）

| 维度 | 跳表 | 红黑树 | B+ 树 |
|---|---|---|---|
| 范围查询 | **✅ 底层是有序链表，定位起点后顺序遍历** | ❌ 需要中序遍历（非连续内存跳转，要靠栈或线索） | ✅ 叶子链表 |
| 实现复杂度 | **✅ 简单**（插入删除只需改指针，无旋转/变色） | ❌ 复杂（旋转 + 变色 + 5 条性质维护） | ❌ 很复杂（页分裂/合并） |
| 并发友好 | **✅ 局部修改，加锁范围小**（改指针即可） | ❌ 旋转会影响子树结构 | ❌ 分裂影响大 |
| 内存可调 | **✅ 通过 p 调整层高与内存的权衡** | ❌ 固定（每个节点 1 个颜色位） | ❌ 固定 |
| 反向遍历 | ✅ 有 backward 指针 | ❌ 不容易 | ✅ |
| 磁盘 IO | ❌ 不适合（多级链表随机访问） | ❌ 不适合 | ✅ **为磁盘设计** |
| 点查复杂度 | O(log n) | O(log n) | O(log n) |

**注意最后一行**：**B+ 树的优势在于"为磁盘设计"（一个节点一页，减少 IO）**。而 Redis 是**内存数据库**，**根本不存在磁盘 IO 的问题**，所以 B+ 树最大的优势在 Redis 里毫无价值，反而要付出实现复杂度高的代价。**这就是"Redis 用跳表、MySQL 用 B+ 树"的根本原因——不是数据结构本身的优劣，而是存储介质决定了设计目标。**

**Redis 作者 antirez 的原话（可以引用，很有说服力）**：跳表实现更简单、更容易调试、更适合范围查询；而且跳表的平衡是**概率性**的（不像红黑树需要严格的平衡操作），在并发环境下更容易做局部修改。

### 1.6 渐进式 rehash（**必考**）

#### 背景：dict 的结构

```c
typedef struct dict {
    dictType *type;      /* 各种操作函数指针 */
    void *privdata;
    dictht ht[2];        /* ★ 两个哈希表！正常只用 ht[0]，rehash 时用 ht[1] */
    long rehashidx;      /* ★ rehash 进度：-1 表示没有在进行 rehash */
    int16_t pauserehash;
    ...
} dict;

typedef struct dictht {
    dictEntry **table;   /* 哈希桶数组 */
    unsigned long size;  /* 哈希表大小（总是 2 的幂）*/
    unsigned long sizemask;  /* size - 1，用于 hash & sizemask 取模 */
    unsigned long used;  /* 已有的节点数 */
} dictht;
```

**为什么 size 是 2 的幂？** 因为 `hash & sizemask` 比 `hash % size` 快（位运算 vs 除法）。

#### 扩容与缩容的触发条件

| 操作 | 条件 | 说明 |
|---|---|---|
| **扩容** | 负载因子 `used / size ≥ 1` **且** 没有子进程在跑（BGSAVE/BGREWRITEAOF） | 常规扩容 |
| **扩容** | 负载因子 `≥ 5` | 强制扩容，**即使有子进程在跑** |
| **缩容** | 负载因子 `< 0.1` | `htNeedsResize()` 检查，由 `serverCron` 定期调用 |

**为什么有子进程时不扩容（负载因子 < 5 时）？** 因为 fork 出来的子进程（做 RDB/AOF 重写）用的是 **Copy-On-Write**：如果父进程此时大量写内存，会触发大量的页复制，**内存可能瞬间翻倍**（这就是"COW 导致的 OOM"）。**Redis 的策略是"子进程在跑的时候尽量少写内存"**——这就是 `dict_can_resize` 标志的作用。

#### 渐进式 rehash 的完整流程

```
步骤 1: 给 ht[1] 分配空间
        扩容：ht[1].size = 第一个 >= ht[0].used * 2 的 2 的幂
        缩容：ht[1].size = 第一个 >= ht[0].used 的 2 的幂
步骤 2: 把 rehashidx 从 -1 置为 0，表示"开始 rehash"
步骤 3: 每次对 dict 做增删改查时，除了执行本身的操作，
        还会把 ht[0] 中 rehashidx 位置的所有节点迁移到 ht[1]，
        然后 rehashidx++
步骤 4: 当 ht[0] 的所有桶都迁移完，rehashidx 置为 -1，
        把 ht[1] 赋值给 ht[0]，清空 ht[1]（rehash 结束）
```

**rehash 期间的读写规则（面试常问细节）**：

| 操作 | 行为 |
|---|---|
| **查找** | **先查 ht[0]，查不到再查 ht[1]**（因为数据可能已经在任一表中） |
| **插入（新增 key）** | **直接插入 ht[1]**，不插 ht[0] —— 保证 ht[0] 只减不增，rehash 一定能结束 |
| **删除** | 两个表都要找，找到就删 |
| **更新** | 两个表都要找，找到就改 |

**为什么需要"渐进式"？** 如果是一次性 rehash：一个存了 100 万个 key 的哈希表，rehash 要迁移 100 万个节点，**而 Redis 是单线程的，这期间所有其他请求全部阻塞**——秒级的卡顿。**渐进式把这一次大阻塞分摊到 N 次小操作上，每次只迁移一个桶（O(1)）**。

> **面试延伸追问**："如果客户端一直不访问这个 dict，rehash 是不是永远不会完成？" → **不会**。Redis 还有一个**定时任务兜底**：`serverCron`（默认每 100ms）会调用 `databasesCron()` → `incrementallyRehash()`，在"没有其他事情可做"时**以 1ms 为时间片继续推进 rehash**（源码 `incrementallyRehash` 里的 `if (dictIsRehashing(d))` 循环 + `timelimit` 检查）。所以**即使 dict 完全不被访问，rehash 也会在后台被推完**。
>
> **再追问**："什么叫 'rehash 期间的均摊 O(1)'？" → 单个操作因为是 O(1) 的桶迁移，所以每次操作是 O(1)；但整个 rehash 期间的总工作量是 O(n)，分摊到 n 次操作上，**每个操作平均多做了 O(1) 的工作**。这就是"渐进式 rehash 让均摊复杂度保持 O(1)"的含义。

**查看 rehash 状态**：

```bash
# 查看 dict 的详细信息（info 段落在 debug 命令里）
127.0.0.1:6379> DEBUG HTSTATS 0
[Dictionary HT]
Hash table 0 stats (main hash table):
 table size: 1024
 number of elements: 1024
 different slots: 655
 max chain length: 6
 avg chain length (counted): 1.56
[Dictionary HT]
Hash table 1 stats (rehashing target):
 table size: 2048
 number of elements: 0
```
（能看到 `Hash table 1` 有内容，就说明正在 rehash。）

### 1.7 为什么 Redis 快？（**面试开场必问，要答出层次**）

这个问题看似简单，但**答出 3 点只能算及格，答出 6 点并分层才算优秀**。建议按"内存 → 模型 → IO → 数据结构 → 协议 → 分配器"的顺序讲。

#### ① 纯内存操作（最根本的原因）

内存随机访问约 **100ns**，而磁盘随机 IO 约 **10ms** —— **相差 10 万倍**。Redis 的绝大部分操作只涉及内存读写，**这是性能优势的根本来源**。**注意**：这条要放在第一位说，因为它才是"快"的本质；后面几条是"如何把内存的优势发挥到极致"。

#### ② 单线程，避免了锁竞争和上下文切换

**单线程处理命令**带来的三个好处：

1. **不需要任何锁**：不用加锁就不用抢锁，也就不用担心死锁和锁的上下文切换（`futex` 系统调用）；
2. **没有线程切换开销**：上下文切换一次约 **几微秒**（还可能引发 CPU 缓存失效），高 QPS 下这个开销很可观；
3. **没有锁粒度问题**：多线程共享数据结构就要做细粒度锁，而细粒度锁的复杂度极高（Java 的 `ConcurrentHashMap` 就是例子）。

> **面试关键点：这里的"单线程"指的是"命令执行是单线程"**。Redis 并不是只有一个线程 —— 它还有：
> - `bio_close_file` / `bio_aof_fsync` / `bio_lazy_free` 三个**后台 BIO 线程**（处理文件关闭、AOF fsync、惰性释放）；
> - Redis 4.0 起用于 `UNLINK`、`FLUSHALL ASYNC` 的**异步删除**线程；
> - **6.0 起用于网络 IO 的多线程**（见 ④）。
>
> **说"Redis 是单线程"是不严谨的，准确说法是"Redis 的命令执行（核心逻辑）是单线程的"。** 这句话面试时说出来，立刻区分于背八股的人。

#### ③ IO 多路复用 + 事件驱动（Reactor 模型）

**问题**：单线程怎么同时处理上万个连接？如果每个连接一个线程，那就是 C10K 问题；如果用非阻塞 IO 轮询，会浪费 CPU。**答案是 IO 多路复用**。

**Redis 自己实现了一个事件库 `ae`（A simple event-driven library）**，封装了不同操作系统的多路复用机制：

```
Linux   → epoll
macOS/BSD → kqueue
Solaris → evport
其他    → select（兜底，性能差）
```

**编译时自动选择最优的实现**（`ae.c` 里的一堆 `#ifdef HAVE_EPOLL`）。

**Redis 的事件循环（`aeMain`）本质是 Reactor 模式**：

```
while (!stop) {
    aeProcessEvents();     /* ① 计算最近的时间事件，作为 epoll_wait 的超时 */
                           /* ② epoll_wait 等待事件就绪 */
                           /* ③ 处理就绪的文件事件（读 → 解析 → 执行命令 → 写回）*/
                           /* ④ 处理到期的时间事件（serverCron 定时任务）*/
}
```

**四类文件事件**：`AE_READABLE`（可读，新连接或命令到达）、`AE_WRITABLE`（可写，回复缓冲区有空闲）、以及组合。

**为什么 epoll 比 select 强？**（顺带把操作系统的知识串起来）

| | select | poll | epoll |
|---|---|---|---|
| 时间复杂度 | O(n)（每次遍历全部 fd） | O(n) | **O(1)**（内核用红黑树管理，就绪的放就绪链表） |
| fd 数量上限 | 1024（FD_SETSIZE） | 无上限 | 无上限 |
| 内存拷贝 | **每次调用都要把 fd 集合从用户态拷到内核态** | 同 select | **只在 `epoll_ctl` 时拷贝一次**（mmap 共享内存） |
| 触发模式 | 只有水平触发 | 只有水平触发 | **支持水平触发（LT）和边缘触发（ET）** |

#### ④ Redis 6.0 的多线程 IO（**高频追问：既然单线程快，为什么还要多线程？**）

**结论先行**：**6.0 的多线程只用于"网络数据的读取和协议解析"以及"回复数据的写回"，命令的执行仍然是单线程。**

**为什么要加？** 随着硬件发展，网络带宽和 QPS 越来越高，**瓶颈从"命令执行"转移到了"网络 IO"**——读 socket、解析 RESP 协议、写 socket 这些工作在单线程下会占用大量 CPU（`sys` 时间显著上升）。**多线程 IO 把这部分并行化，而保留单线程执行命令的模型（从而保留了"无锁"的核心优势）。**

**配置**：

```bash
# 开启多线程 IO（默认关闭，io-threads 默认为 1 即不开启）
io-threads 4                  # 建议设置为 CPU 核数的 3/4（比如 8 核设 6）
io-threads-do-reads yes       # 默认 no，是否让多线程也处理"读"（读多写少可以开）
```

**注意**：官方建议 `io-threads` **不要超过 8**（超过后收益递减，且线程调度开销增加），并且**只在 4 核以上、QPS 确实很高的场景开启**。**开多线程 IO 对单个命令的延迟没有改善（因为命令执行还是单线程），它改善的是吞吐量。**

> **面试话术（把 ② 和 ④ 的矛盾圆上，这是加分的关键）**：
> "问 Redis 为什么快，最根本的原因是**纯内存**；单线程是**为了省掉锁和上下文切换的开销**，但这个优势只在**命令执行**这个环节成立。到了 6.0，随着网卡带宽和 QPS 增长，**瓶颈从命令执行转移到了网络 IO**（读 socket、解析协议占了大量 CPU），所以 6.0 引入了多线程 IO —— 但**它多线程化的只是网络读写和协议解析，命令执行依然是单线程**，这样既解决了网络瓶颈，又保留了无锁模型。所以'Redis 单线程'和'Redis 6.0 多线程'并不矛盾，**它们说的是不同环节**。"

#### ⑤ 高效的数据结构与专门优化

- **SDS**：O(1) 取长度、二进制安全、预分配减少内存重分配；
- **跳表**：范围查询 O(log n)，实现简单；
- **listpack/ziplist**：小数据量时紧凑存储，内存连续、缓存友好；
- **共享对象**：0~9999 的整数（`server.h` 里的 `shared.integers[10000]`）在启动时预创建，`SET k 100` 时**直接复用已有的对象**（`refcount++`），不额外分配内存；
- **`maxmemory-policy` 的 LRU/LFU 近似算法**：**不是精确 LRU**，而是**随机采样 N 个 key（默认 5 个），淘汰其中最久未使用的** —— 为了省内存（精确 LRU 需要双向链表，每个 key 多 2 个指针）。这是"近似算法换内存"的经典案例。

#### ⑥ 内存分配器：jemalloc

Redis 默认用 **jemalloc**（`jemalloc` 比 glibc 的 `malloc` 在碎片控制上更好，且支持**多线程 arena**）。编译时的 `MALLOC` 变量可以切换。**jemalloc 的"内存块档位"（8/16/32/48/64...）也是前面 `embstr` 44 字节的来源。**

#### ⑦ RESP 协议简单 + 请求-响应模型

RESP（REdis Serialization Protocol）是**二进制安全的文本协议**，解析极快：用第一个字节标识类型（`+` 简单字符串、`-` 错误、`:` 整数、`$` 批量字符串、`*` 数组），后面跟长度和内容。

**协议简单 → 解析开销小**，这是设计上的刻意选择。

**总结图（面试可以直接照这个结构说）**：

```
为什么快？
├── 根本原因：纯内存操作（比磁盘快 10 万倍）
├── 并发模型：命令执行单线程（无锁、无上下文切换）
├── 网络模型：IO 多路复用（epoll）+ 事件驱动 Reactor
│             └── 6.0 起：网络 IO 多线程（读/解析/写回并行，命令执行仍单线程）
├── 数据结构：SDS / 跳表 / listpack（快且省内存）
├── 内存优化：共享整数对象、近似 LRU/LFU
├── 底层设施：jemalloc 内存分配器
└── 协议：RESP 简单易解析
```

**但快也有代价（面试官爱追问"Redis 有什么缺点"）**：
- **单线程 → 一个慢命令会阻塞所有请求**（`KEYS *`、`FLUSHALL`、大 key 的 `HGETALL`、`DEL` 大 key）；
- **单线程 → 无法利用多核**（一台机器要跑多个实例做集群）；
- **内存昂贵 → 容量受限**；
- **持久化有数据丢失风险**（RDB 会丢，AOF everysec 最多丢 1 秒）；
- **主从是异步复制 → 有丢数据的可能**。

### 1.8 本节高频 Q&A

**Q1：Redis 的过期删除策略是什么？**

> **参考回答**：**惰性删除 + 定期删除**的组合。
> **① 惰性删除（lazy expiration）**：访问一个 key 时，先检查它是否过期，过期就删除并返回 nil。**优点**：只在访问时付出成本，对 CPU 友好；**缺点**：如果一个 key 过期后永远不被访问，它会一直占着内存（内存泄漏）。
> **② 定期删除（active expiration）**：Redis 每秒默认执行 **10 次**（`hz` 配置，默认 10）过期扫描，每次**随机抽取 20 个设置了过期时间的 key**，删除其中已过期的；**如果这 20 个里有超过 25% 已过期，就再抽 20 个继续删**，直到过期比例低于 25% 或达到**时间上限（25ms）**，避免阻塞主线程。
> **为什么是"随机抽样"而不是"全量扫描"？** 因为全量扫描 O(n)，在几百万 key 的实例上会阻塞单线程。**随机抽样 + 比例控制**是"控制阻塞时间"和"清理彻底性"的折中。
> **还有一个重要细节**：**从库不会主动删除过期 key** —— 从库上的过期 key 要等主库发来 `DEL` 命令才删（保证主从一致）。但**从库在读取时会判断逻辑过期并返回 nil**（`expireIfNeeded` 在从库上的行为不同）。这个细节经常被面试官拿来考。

**Q2：内存淘汰策略有哪些？**

> **参考回答**：8 种策略（`maxmemory-policy`），按"淘汰范围"分两大类：
> **只淘汰设置了过期时间的 key（`volatile-*`）**：
> - `volatile-lru`：近似 LRU（最久未使用）
> - `volatile-lfu`：近似 LFU（最少使用频率，4.0+）
> - `volatile-random`：随机
> - `volatile-ttl`：优先淘汰剩余 TTL 最短的
> **在所有 key 里淘汰（`allkeys-*`）**：
> - `allkeys-lru`（**最常用**）
> - `allkeys-lfu`
> - `allkeys-random`
> **不淘汰**：`noeviction`（默认）—— 内存满了直接对新写入**返回错误**（读命令还能正常执行）
>
> **LRU 和 LFU 的区别（高频追问）**：LRU 看"最近一次访问时间"，**问题是一个刚被访问过一次的冷数据会挤掉一个长期高频的热数据**（缓存污染）；LFU 看"访问频率"（用 8 bit 的计数器 + 8 bit 的衰减时间，`lfu-log-factor` 和 `lfu-decay-time` 控制）。**实践建议**：**热点数据集中、访问模式稳定的场景用 LFU 更好**（比如排行榜、商品详情）；**如果访问模式是"扫描式的"（像全表遍历），LFU 的计数器会被污染**——不过 Redis 的 LFU 有**衰减机制**（`lfu-decay-time` 分钟级别的衰减），能缓解这个问题。
> **注意**：**Redis 的 LRU 不是精确 LRU**，是"随机采样 `maxmemory-samples`（默认 5）个 key，淘汰其中最久未使用的"。**把 `maxmemory-samples` 调大（如 10）会更接近真实 LRU，但消耗更多 CPU** —— 又是"精度换性能"的权衡。

**Q3：`KEYS *` 为什么被禁用？生产上怎么替代？**

> **参考回答**：`KEYS pattern` 是 **O(n) 的遍历**（n 是 key 总数），而且它是**同步阻塞**的——在有几百万 key 的实例上执行会阻塞主线程好几秒，期间**所有请求都卡住**，是生产事故级别的命令。**替代方案**：
> - **`SCAN cursor [MATCH pattern] [COUNT n]`**：渐进式遍历，每次返回一批（COUNT 是提示值，不是精确值），**不阻塞主线程**。注意：**SCAN 只能保证"遍历期间一直存在的 key 一定会被返回"，但可能返回重复的 key**（因为 rehash 会导致桶的重新分布），所以应用层要**去重**；
> - **更好的做法：用额外的数据结构维护 key 的索引**。比如给所有 `user:*` 的 key 维护一个 Set（`SADD index:user user:1`），需要遍历时直接 `SMEMBERS index:user`。**这样是把 O(n) 的遍历变成 O(1) 的集合查询，代价是写入时要维护索引**（可以用 Lua 保证原子）；
> - **`UNLINK` 替代 `DEL`**（异步删除大 key，4.0+）。
>
> **同类危险命令清单（面试可以主动报，很加分）**：`KEYS`、`FLUSHALL`/`FLUSHDB`（用 `ASYNC` 变体）、`HGETALL`/`LRANGE`/`SMEMBERS`（大 key 时危险）、`DEL` 大 key（用 `UNLINK`）、`SORT`（复杂度高）、`ZRANGE key 0 -1`（大 ZSet）。**在云厂商的 Redis 服务里，`KEYS` 通常是被直接禁用的。**

**Q4：Redis 的 Pipeline、事务、Lua 脚本有什么区别？**

> **参考回答**：三者都能"一次发送多个操作"，但语义完全不同。
>
> | | Pipeline（管道） | MULTI/EXEC 事务 | Lua 脚本 |
> |---|---|---|---|
> | **目的** | **减少网络 RTT** | **原子性（打包执行）** | **原子性 + 服务端计算逻辑** |
> | 原子性 | ❌ 无（命令之间可能插入其他客户端的命令） | ✅ 有（但**不是"要么全成功要么全失败"**） | ✅ 有（整个脚本是一个原子操作） |
> | 回滚 | ❌ | ❌ **不支持回滚**（`EXEC` 时某条命令出错，其他命令照样执行） | ❌ 不支持（但可以用 `redis.pcall` 捕获错误） |
> | 条件逻辑 | ❌ | ❌ **不能根据前一条命令的结果决定下一条**（这是它最大的局限） | ✅ **可以**（能读、判断、写） |
> | 阻塞 | 不阻塞（只省网络） | 不阻塞（只保证原子执行，执行本身还是 O(1) 逐个命令） | **可能阻塞**（脚本执行期间其他请求全部等待） |
>
> **关键理解**：**Redis 事务是"打包执行 + 隔离"，但不是数据库意义上的原子性**——它**没有回滚**（因为 Redis 的错误分两类：**语法错误**在入队时就检测到，整个事务会失败；**运行期错误**（如对 String 执行 `LPUSH`）只影响那一条命令，其他命令照常执行）。**所以 Redis 事务"不支持原子性回滚"是设计选择，不是缺陷**——因为回滚需要 undo log，会破坏"简单快速"的设计。
>
> **实践建议**：**需要用"读-判断-写"的逻辑，一律用 Lua 脚本**（比如分布式锁释放、库存扣减、限流）。**注意 Lua 脚本要短**（避免阻塞），且 **Redis 7.0 起脚本默认不能再写非确定性的命令**（为了保证主从一致和 AOF 重放的确定性）。
>
> **补充：Redis 7.0 的 Function（`FUNCTION LOAD`）** 是把 Lua 脚本做成"可持久化的服务端函数"，避免了每次调用都要传脚本全文（`EVALSHA` 的 `NOSCRIPT` 问题的根本解法）。

**Q5：`SETNX` 和 `SET key value NX PX` 的区别？**

> **参考回答**：`SETNX`（SET if Not eXists，**已经废弃的写法**）**只做"不存在则设置"，无法设置过期时间**，所以必须配合 `EXPIRE`——但**这两条命令之间不是原子的**，如果客户端在 `SETNX` 成功后崩溃（或网络断开），`EXPIRE` 没执行，这个 key 就**永远不会过期**，形成死锁。**正确做法是用 `SET key value NX PX 30000` 一条原子命令**（Redis 2.6.12+ 起 `SET` 支持 `NX`/`XX`/`EX`/`PX` 等选项），**加锁和设置过期时间一次完成**。
>
> **这是面试官最爱抓的坑**，如果你写 `SETNX` + `EXPIRE` 的分布式锁，直接会被判定为"没实战经验"。

**Q6：`SORT` 命令、`ZRANGE` 大范围的性能问题？**

> **参考回答**：`SORT` 是 **O(n log n) 甚至 O(n × m)** 的（带 `BY`/`GET` 模式会触发多次随机 key 查找），而且是**同步阻塞**的，大集合上执行会阻塞主线程。**实践中基本不用 `SORT`**，排序需求交给 ZSet（天然有序）或者应用层。`ZRANGE key 0 -1` 在百万级 ZSet 上会返回巨大的结果集（网络传输 + 内存峰值），要用 `ZRANGE key 0 99` 分页。
>
> **结合项目**：FlashRoot 的渠道商排行如果用 ZSet 做，一定是 `ZREVRANGE key 0 99 WITHSCORES` 取前 100，而不是全量取回来在应用层排。

### 1.9 本节「结合项目」话术提示

| 问题 | 钩子 | 话术 |
|---|---|---|
| 你项目里用了哪些数据结构？ | **全项目** | "**LoveSpouse 的金币余额用 String（DECR/INCR + Lua 做原子扣减）；FlashRoot 的渠道商排行榜用 ZSet；POS 数据贯通的去重用 Set（`SADD` 返回 0 表示已处理）；组织权限和字典类数据用 Hash（按字段更新，避免整个对象序列化反序列化）；轻量的本地队列/最近消息用 List + LPUSH/LTRIM 做固定长度的最新消息列表。**" |
| 怎么选 String 还是 Hash？ | **通用** | "如果需要整体读写（比如缓存一个 JSON 对象），用 String；如果需要**部分更新/部分读取**（比如只改用户的一个字段），用 Hash——Hash 的 `HSET` 只更新一个 field，而 String 必须把整个对象读出来改完再写回去，在高并发下有丢更新的风险（读-改-写的竞态）。" |
| 大 key 遇到过吗？ | **LoveSpouse / POS** | 见第 7 节。 |

---

## 2. 持久化：RDB / AOF / 混合持久化

> **核心矛盾**：Redis 是内存数据库，**一旦进程挂掉（或机器断电），内存里的数据就全没了**。持久化就是解决"如何把内存数据落到磁盘"的问题。**但 Redis 的持久化不是"绝对不丢"，而是"在性能和安全性之间选一个点"** —— 这就是 RDB 和 AOF 的分野。

### 2.1 RDB（Redis Database）：快照

#### 是什么

RDB 是**某个时间点的全量数据快照**，存成一个**紧凑的二进制文件**（默认 `dump.rdb`）。

#### 触发方式

| 方式 | 命令/配置 | 特点 |
|---|---|---|
| **手动同步** | `SAVE` | **阻塞主线程**直到完成（生产禁用） |
| **手动异步** | `BGSAVE` | **fork 子进程**做，主进程继续服务 |
| **自动（配置）** | `save 900 1` / `save 300 10` / `save 60 10000` | 900 秒内至少 1 次修改、300 秒内至少 10 次、60 秒内至少 10000 次 → 触发 BGSAVE |
| **关闭时** | `SHUTDOWN` | 如果没有开 AOF，会执行一次 `SAVE` |
| **主从全量同步** | `REPLICAOF` | 主库 `BGSAVE` 生成 RDB 发给从库（**这是 RDB 最重要的用途**） |
| **`DEBUG RELOAD`** | 调试用 | |

**检查 `save` 配置的状态**：

```bash
127.0.0.1:6379> CONFIG GET save
1) "save"
2) "3600 1 300 100 60 10000"

# 查看上次保存的状态
127.0.0.1:6379> INFO persistence
rdb_last_save_time:1758100000
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:2
rdb_changes_since_last_save:42
rdb_bgsave_in_progress:0
```

> **生产建议**：**关闭自动 RDB（`save ""`），只保留主从同步时的 RDB**。因为自动 RDB 的触发时机不可控（可能在业务高峰），fork 会带来延迟抖动；而持久化可靠性由 AOF 保证。**这是很多大厂的标准配置。**

#### fork 与 Copy-On-Write（COW）—— **面试核心考点**

`BGSAVE` 的流程：

```
主进程 fork() → 子进程
                    ↓
            子进程把内存中的数据写成 RDB 文件
                 （此时主进程继续处理请求）
                    ↓
            子进程完成，通知主进程
```

**fork 的代价**：

1. **fork 本身会阻塞主进程**，阻塞时间与**进程的页表大小成正比**（不是与内存大小成正比）。一个几十 GB 的实例，页表可能很大，fork 可能阻塞**几十到几百毫秒**。**这是 Redis 延迟抖动最常见的来源之一**（可以用 `INFO stats` 的 `latest_fork_usec` 看到最近一次 fork 的耗时）。
2. **COW 带来的内存开销**：fork 后父子进程**共享物理内存页**（Copy-On-Write）。当**主进程写**某个页时，内核会**复制这个页**给主进程（子进程仍用旧的），于是**内存占用可能增长**。**最坏情况下（主进程写遍了所有页）内存会翻倍** —— 这就是为什么**建议 Redis 实例的内存不超过物理内存的一半**。
3. **`overcommit_memory` 的坑**：Linux 默认的 `vm.overcommit_memory=0` 会在内存不足时拒绝 `fork`（返回 `Cannot allocate memory`），导致 BGSAVE 失败。**生产环境要设置 `vm.overcommit_memory=1`**（允许超量分配）。这个坑 Redis 启动时会打印 WARNING，**面试说出来很加分**。

#### RDB 的优缺点

| 优点 | 缺点 |
|---|---|
| **文件紧凑**（二进制 + 压缩 `rdbcompression`），适合备份和传输 | **可能丢失最后一次快照之后的所有数据**（比如 60 秒内改了 10000 条才触发，这 60 秒内的数据全丢） |
| **恢复速度快**（直接加载进内存，比逐条重放 AOF 命令快得多） | **fork 有阻塞和内存开销**（见上），大实例上代价明显 |
| **对主进程影响小**（子进程完成） | **文件格式与版本相关**（高版本 RDB 低版本 Redis 读不了） |
| 适合**主从全量同步**和**冷备** | 无法做到"秒级持久化" |

### 2.2 AOF（Append Only File）：写命令日志

#### 是什么

AOF 记录的是**所有写命令**（以 RESP 协议格式追加写入），重启时**重放这些命令**来恢复数据。**这是"逻辑日志"，而 RDB 是"物理快照"** —— 这是两者最本质的区别。

#### 写入流程：三个缓冲区

这是面试的重点，**很多人只答"追加到文件"，答不出这三个阶段**：

```
客户端命令
    ↓ ① 
[redis 主线程] 把命令追加到 aof_buf（AOF 缓冲区，内存）
    ↓ ② write()
[内核 page cache（OS 缓冲区）]
    ↓ ③ fsync()
[磁盘]
```

**对应的 `appendfsync` 三种策略**：

| 策略 | 行为 | 丢失窗口 | 性能 | 适用 |
|---|---|---|---|---|
| **`always`** | **每个写命令都 fsync** | 几乎不丢（最多丢一条） | **最差**（每次都要等磁盘） | 金融级强要求（但 Redis 用这个性能会跌到几百 QPS） |
| **`everysec`（默认，推荐）** | **每秒 fsync 一次**（由**后台 BIO 线程**执行，不阻塞主线程） | **最多丢 1 秒** | **好** | **绝大多数生产场景** |
| **`no`** | **不主动 fsync，交给操作系统决定**（Linux 默认 30 秒） | 最多丢 30 秒 | 最好（不用管磁盘） | 纯缓存场景，数据丢了能重建 |

> **面试要点（很细节，能体现深度）**：**`everysec` 的 fsync 是在后台线程做的**（Redis 2.4 起引入 BIO 线程），所以**正常情况下主线程不会被磁盘 IO 阻塞**。但如果上一次 fsync **还没完成**（比如磁盘很慢），主线程会被迫**等待**（源码 `writeToAofBuf` 里的检查）—— 这就是"Redis 被慢磁盘拖死"的场景。**监控指标 `aof_delayed_fsync`** 就记录了"因为延迟 fsync 而阻塞主线程"的次数，**这个值持续增长说明磁盘有性能问题**，要立刻告警。

#### AOF 重写（AOF Rewrite）

**为什么需要重写？** AOF 是**追加写**的，一个 key 被改了 100 次就有 100 条命令，但**恢复时只需要最后一条**。所以文件会无限膨胀（比如一个计数器被 `INCR` 了 100 万次，同一时间只有一个 key，但 AOF 有 100 万条命令）。

**重写的原理**：**不是分析旧 AOF 文件，而是直接读当前内存里的数据**，用**最少的命令**把它表达出来：

```
旧 AOF（100 万条命令）:
  INCR counter        # 1
  INCR counter        # 2
  ... （100 万次）
  INCR counter        # 1000000

重写后（1 条命令）:
  SET counter 1000000
```

**重写的完整流程（面试必答，要讲清楚"重写缓冲区"）**：

```
1. 主进程 fork 出子进程
2. 子进程遍历当前内存，把数据写成"最少命令集"到新的 AOF 文件（临时文件）
   ↓ 关键问题：这期间主进程还在处理写命令，新数据怎么办？
3. ★ 主进程在重写期间，把新的写命令：
      - 同时追加到【旧的 AOF 缓冲区（aof_buf）】→ 保证旧 AOF 文件仍然完整（可以用于恢复）
      - 同时追加到【AOF 重写缓冲区（aof_rewrite_buf）】→ 留给新 AOF 文件
4. 子进程写完，通知主进程
5. ★ 主进程把 aof_rewrite_buf 里的内容追加到新的 AOF 文件末尾
6. ★ 用新 AOF 文件原子替换旧文件（rename）
```

**为什么要用两个缓冲区？** 因为**如果新 AOF 只靠子进程写，主进程在重写期间的新命令就丢了**；而**如果让主进程直接写新 AOF 文件，就会和子进程写的文件内容冲突（父子的文件偏移量无法共享）**。所以需要一个**专门的重写缓冲区**，在子进程完成后由主进程**一次性合并**。

**触发方式**：

```bash
# 自动触发（按增长比例和最小大小）
auto-aof-rewrite-percentage 100    # AOF 文件比上次重写后增长 100%（即翻倍）时触发
auto-aof-rewrite-min-size 64mb     # 且文件至少 64MB

# 手动触发（不会阻塞主线程，fork 子进程）
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started
```

**注意**：重写期间的**第 5、6 步是在主线程做的**（追加缓冲区 + rename），如果缓冲区很大，会有短暂的阻塞。**这是 AOF 重写对主线程的唯一影响点**。

#### AOF 的优缺点

| 优点 | 缺点 |
|---|---|
| **数据安全性高**（everysec 最多丢 1 秒） | **文件比 RDB 大得多**（同样的数据） |
| **可读性好**（文本命令，能看懂、能手工修复） | **恢复速度慢**（要逐条重放命令，比 RDB 慢一个数量级） |
| **格式简单**（`APPEND` 追加，不会因为写入中断而损坏已写部分） | **写入有额外开销**（每条命令都要追加，虽然 fsync 可以后台做） |
| 支持 `BGREWRITEAOF` 在线瘦身 | 重写期间有内存开销（重写缓冲区） |

#### AOF 文件损坏怎么办？

Redis 提供了修复工具：

```bash
# 检查并修复 AOF 文件
redis-check-aof --fix appendonly.aof
# 会找到第一条不完整的命令并截断（因为 append 可能被中途中断）

# 对应的配置项
aof-load-truncated yes   # 默认 yes：加载时遇到 AOF 末尾不完整，直接截断并继续（并打日志）
                         # 设为 no：不完整就直接启动失败（更严格，避免静默丢数据）
```

### 2.3 混合持久化（4.0+，**生产推荐**）

#### 为什么需要混合？

**RDB 恢复快但不安全；AOF 安全但恢复慢。** 于是 Redis 4.0 引入混合持久化：**"RDB 的全量快照 + AOF 的增量命令"**。

#### 文件格式

开启 `aof-use-rdb-preamble yes`（**4.0 起默认 yes**）后，AOF 文件变成：

```
┌─────────────────────────────────┬──────────────────────────────┐
│  RDB 格式的全量数据（preamble）  │  AOF 格式的增量写命令          │
│  （重写那一刻的内存快照）        │  （重写之后的所有写命令）      │
└─────────────────────────────────┴──────────────────────────────┘
        以 "REDIS" 魔数开头                 以 RESP 文本命令开头
```

**注意**：**只有在 AOF 重写（BGREWRITEAOF）之后**，文件才会变成混合格式；重写之前 AOF 文件仍然是纯纯的命令。

**恢复流程**：Redis 加载文件时先看开头是不是 `REDIS` 魔数——是则按 RDB 格式加载前半部分，再按 AOF 格式重放后半部分。

**收益**：
- **恢复速度快**（前半部分是 RDB，加载快）；
- **数据安全性高**（后半部分保留了重写之后的增量命令）；
- **文件体积小**（RDB 是紧凑的二进制）。

> **面试话术**：**"混合持久化本质上是'用 RDB 做基线 + 用 AOF 做增量'——它巧妙地结合了两者的优点：RDB 快，AOF 全。这也解释了为什么 4.0 之后它成为默认配置。"**

### 2.4 数据恢复流程与选型建议

#### 启动时的加载优先级

```
Redis 启动
    ↓
开启了 AOF（appendonly yes）？
    ├── 是 → 加载 AOF 文件（优先级更高！因为 AOF 更完整）
    │         └── 如果是混合格式，前半部分按 RDB 加载
    └── 否 → 加载 RDB 文件
              └── 找不到 RDB → 空实例启动
```

**注意**：**AOF 的优先级高于 RDB**。所以如果一个实例同时开了两者，重启时会用 AOF 恢复（因为 AOF 通常更新更全）。**这个细节面试常考。**

#### 生产环境的持久化配置建议（**可以直接背的答案**）

```ini
# ============ 主库（对数据安全性有要求）============
# 关闭自动 RDB（避免不可控的 fork 抖动）
save ""
# 开启 AOF
appendonly yes
appendfsync everysec                 # 每秒一次，最多丢 1 秒
no-appendfsync-on-rewrite no         # 重写期间是否停止 fsync（no 更安全，yes 性能更好但可能丢更多）
aof-use-rdb-preamble yes             # 混合持久化（4.0+ 默认）
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 1gb        # 调大，避免频繁重写
aof-load-truncated yes               # 末尾损坏时截断启动
# 内存不超过物理内存的一半（给 fork/COW 留空间）
maxmemory 16gb                       # 假设物理内存 32GB
vm.overcommit_memory = 1             # 系统参数，必须设！否则 fork 可能失败

# ============ 纯缓存场景（数据可重建）============
save ""
appendonly no                        # 完全关闭持久化，性能最好
maxmemory-policy allkeys-lru
```

> **面试话术**："持久化方案的选择取决于**数据能不能丢**。如果 Redis 是纯缓存（数据在 MySQL 里有权威副本），我倾向于**关闭持久化**，因为重启后从数据库回源重建更简单可靠，还省掉了 fork 和 fsync 的开销；如果是**存储了唯一数据的场景**（比如用 Redis 做队列，消息还没被消费），必须开 AOF everysec + 混合持久化，而且要考虑**主从 + 哨兵**保证可用性。**LoveSpouse 的金币余额就是这种'不能丢'的场景**，所以我们的 Redis 是开了 AOF + 主从的。"（**LoveSpouse 金币计费**）

### 2.5 本节高频 Q&A

**Q1：RDB 和 AOF 的本质区别是什么？**

> **参考回答**：**RDB 是"数据快照"（物理/逻辑混合的二进制），AOF 是"命令日志"（逻辑日志）**。这个区别决定了它们的一切差异：RDB 记录"某个时刻数据长什么样"，所以文件紧凑、恢复快，但只能恢复到快照点；AOF 记录"数据是怎么变成现在这样的"，所以文件大、恢复慢，但可以做到秒级的持久化粒度。**用数据库类比：RDB 像 `mysqldump`，AOF 像 `binlog`。** —— 这个类比通常能让面试官点头。

**Q2：fork 的时候会发生什么？为什么 Redis 内存要控制在物理内存的一半？**

> **参考回答**：`fork()` 会**复制父进程的页表**（不是复制内存数据），所以阻塞时间与**页表大小**正相关；然后父子进程共享物理内存页，靠 **Copy-On-Write** 隔离——**父进程写哪个页，内核就复制哪个页**。极端情况下，如果 BGSAVE/AOF 重写期间主进程把**所有页都写了**，内存会翻倍。所以：
> - 内存建议不超过物理内存的 **50%~60%**（留出 COW 的空间）；
> - 必须设置 `vm.overcommit_memory=1`，否则内存紧张时 `fork` 会直接失败；
> - 监控 `latest_fork_usec`（fork 耗时）和 `rdb_bgsave_in_progress`；
> - **避免在业务高峰期触发 BGSAVE**（所以生产上直接把自动 RDB 关掉）。

**Q3：AOF 重写期间的新写入会丢吗？**

> **参考回答**：不会。主进程在重写期间会把新的写命令**同时**写入两个地方：**旧的 AOF 缓冲区**（保证旧文件仍然完整可用）和 **AOF 重写缓冲区**（`aof_rewrite_buf`）。子进程写完后，主进程把重写缓冲区的内容**追加到新文件末尾**，然后 `rename` 原子替换。**这个"双缓冲区"的设计是 AOF 重写正确性的关键**。**唯一的风险点**是最后合并 + rename 这两步在主线程做，如果重写缓冲区积累了很多数据（比如重写耗时很长、写入量很大），合并时会有一个短暂的阻塞。

**Q4：`BGSAVE` 和 `BGREWRITEAOF` 能同时执行吗？**

> **参考回答**：**不能同时执行，会串行化**（两个操作都要 fork，同时做的话内存开销太大）。
> - 如果 `BGSAVE` 正在执行，客户端发 `BGREWRITEAOF`，Redis 会**把它排到 BGSAVE 之后**（返回"Background append only file rewriting scheduled"）；
> - 如果 `BGREWRITEAOF` 正在执行，客户端发 `BGSAVE`，Redis 会**拒绝**（因为 AOF 重写通常更紧急，且 BGSAVE 可以稍后再做）。
>
> **另外要注意**：**这两个操作期间都不会停止服务**，但 fork 的瞬间会阻塞主线程。

**Q5：如果 Redis 突然断电，会丢多少数据？**

> **参考回答**：取决于配置：
> - **只有 RDB**：丢失最后一次快照之后的所有数据（可能是几分钟、甚至几小时，取决于数据量是否达到 `save` 的阈值）；
> - **AOF everysec**：最多丢 **1 秒**的数据；
> - **AOF always**：几乎不丢（最多丢最后一条命令）；
> - **混合持久化 + everysec**：同 everysec，最多丢 1 秒；
> - **完全不开持久化**：全丢，重启后是空实例。
>
> **注意一个容易被忽略的点**：**即使开了 AOF everysec，主从架构下的故障切换仍然可能丢数据**——因为主从复制是**异步**的，主库写成功但还没同步到从库时就挂了，这部分数据（可能不止 1 秒）就丢了。**Redis 官方明确说"Redis 不保证强一致"**，如果要更强的一致性可以用 **`WAIT numreplicas timeout`**（阻塞等待 N 个副本确认），但即使这样也只是"减少"而不是"消除"丢数据的可能（因为主库可能在确认后自己挂掉）。

### 2.6 本节「结合项目」话术提示

- **"你们 Redis 怎么配的？"** → "LoveSpouse 的金币余额是**不能丢的**，所以我用的是 **AOF everysec + 混合持久化 + 主从**，并且监控 `aof_delayed_fsync` 和 `latest_fork_usec`；POS 数据贯通里的 Redis 主要是做**缓存和幂等去重**，数据在 MySQL 有权威副本，所以配的是 `allkeys-lru` + 不开持久化，重启后回源重建。**我觉得持久化策略应该由'数据能不能丢'决定，而不是统一配一套。**"（**LoveSpouse + POS 数据贯通**）
- **"你遇到过 Redis 的性能抖动吗？"** → "最常见的是 **fork 导致的延迟抖动**。我的处理是：把 `auto-aof-rewrite-min-size` 调大避免频繁重写、关掉自动 RDB、控制单实例内存在物理内存的 60% 以内，并且监控 `latest_fork_usec` 这个指标。"（**通用**）

---

## 3. 高可用：主从 / 哨兵 / Cluster

> **三个层次的可用性**：**主从复制**解决"读扩展 + 数据备份"，**哨兵**解决"主库挂了自动切换"，**Cluster** 解决"单机内存不够 + 写扩展"。**面试时要能把这三者的定位说清楚，而不是混在一起讲。**

### 3.1 主从复制（Replication）

#### 建立复制的三种方式

```bash
# ① 配置文件（重启生效）
replicaof 192.168.1.1 6379     # 5.0 之前叫 slaveof

# ② 命令行（立即生效，但重启后失效）
127.0.0.1:6379> REPLICAOF 192.168.1.1 6379

# ③ 取消复制（变回主库）
127.0.0.1:6379> REPLICAOF NO ONE
```

#### 全量同步（Full Resynchronization）—— 首次复制

**流程（面试必背）：**

```
从库                                        主库
 │                                           │
 │──── ① PSYNC ? -1 ────────────────────────→│   （? 表示不知道主库 runid）
 │                                           │   ② 判定需要全量同步
 │←─── ③ +FULLRESYNC <runid> <offset> ───────│
 │                                           │   ④ fork 子进程生成 RDB
 │                                           │   ⑤ 同时把新写命令存入 repl_buffer
 │←─── ⑥ 发送 RDB 文件 ──────────────────────│
 │  ⑦ 清空自己的数据，加载 RDB                 │
 │                                           │   ⑧ 发送 repl_buffer 里的命令
 │←─── ⑨ 发送缓冲区中的写命令 ───────────────│
 │  ⑩ 之后主库持续把写命令传播给从库           │
 │←──── 持续传播 ────────────────────────────│
 │                                           │
 │──── ⑪ 每 1 秒 REPLCONF ACK <offset> ─────→│   （心跳 + 汇报自己的偏移量）
```

**关键点**：
- `runid`（replication ID）：每个 Redis 实例启动时生成的 40 位随机字符串，**用于判断"这个从库之前连的是不是我这个主库"**；
- `offset`（复制偏移量）：主从各自维护，**主库每发送 N 字节就 `master_repl_offset += N`，从库每收到 N 字节就 `slave_repl_offset += N`**。两者相等说明同步完成；
- 全量同步期间主库的写命令要**缓存到 `repl_buffer`**（给这个从库专用的输出缓冲区），RDB 发完后补发。

**查看复制状态**：

```bash
127.0.0.1:6379> INFO replication
role:master
connected_slaves:2
slave0:ip=10.0.0.2,port=6379,state=online,offset=1234567,lag=0
slave1:ip=10.0.0.3,port=6379,state=online,offset=1234560,lag=1
master_repl_offset:1234567          # ★ 主库的偏移量
repl_backlog_active:1
repl_backlog_size:1048576           # ★ repl_backlog 的大小（默认 1MB）
repl_backlog_first_byte_offset:1222222
repl_backlog_histlen:12346
```

> **`lag`** 是从库最后一次向主库发 ACK 距今的秒数。**`lag` 持续 > 1 说明网络或从库有瓶颈**。

#### 增量同步（Partial Resynchronization）—— 断线重连

**场景**：从库和主库的网络短暂断开（比如 5 秒），重连后**不需要重新传全量 RDB**（那样代价太大，尤其主库内存几十 GB 时）。

**依赖的三要素**：

1. **主库的 `runid`**（从库记住上次连的主库 ID）；
2. **复制偏移量 `offset`**；
3. **复制积压缓冲区 `repl_backlog`**（**主库上的一个环形缓冲区，默认 1MB**）。

**流程**：

```
从库重连 → 发 PSYNC <runid> <offset>
    ↓
主库判断：
  ├── runid 不匹配（换过主库）→ 全量同步
  ├── offset 不在 repl_backlog 范围内（断线太久，数据已被覆盖）→ 全量同步
  └── runid 匹配 且 offset 在 repl_backlog 内 → +CONTINUE，只发缺失的那部分 ✅
```

**`repl_backlog` 的大小怎么定？（面试常考）** 公式：

```
repl_backlog_size = 平均写入速率（字节/秒） × 预计最长的断线时间（秒）
```

**举例**：如果主库的写入速率是 1MB/s，你希望能容忍 60 秒的断线，那 `repl_backlog_size` 至少要 **60MB**。**默认的 1MB 在高写入场景下几分钟就会绕一圈**，导致任何稍长的断线都触发全量同步（**"全量同步风暴"**）。**这是生产上非常常见的调优点，面试说出来是加分项。**

**查看**：`INFO replication` 里的 `sync_full`（累计全量同步次数）和 `sync_partial_ok`（增量同步成功次数）。**如果 `sync_full` 持续增长，说明 `repl_backlog` 太小或网络不稳定。**

#### 主从复制的其他关键点

| 点 | 说明 |
|---|---|
| **从库是只读的吗？** | 默认 `replica-read-only yes`（**强烈建议保持 yes**）。从库可写会导致主从数据不一致 |
| **复制是异步的** | 主库写成功即返回，不等从库确认 → **可能丢数据**（见 2.5 Q5） |
| **`repl-diskless-sync`** | 无盘复制（4.0+）：主库不生成 RDB 文件，直接通过 socket 发给从库。适合**磁盘慢但网络快**的场景（如云环境） |
| **复制风暴** | 一个主库挂很多从库，或者从库再挂从库（级联复制）。**全量同步时主库要 fork 多次**，压力大。**解法**：级联复制（从库挂从库）、控制单主库的从库数量、错开全量同步时间 |
| **主从延迟的常见原因** | 网络延迟、从库执行慢命令（`KEYS`）、从库在做 BGSAVE、主库写入量突发（超过网络带宽）、`repl_backlog` 太小导致反复全量同步 |
| **`min-replicas-to-write`** | 主库只有在**至少有 N 个从库的 lag 不超过 M 秒**时才接受写入（防止脑裂时的数据丢失，见 3.2） |

### 3.2 哨兵（Sentinel）：自动故障转移

#### 哨兵的四项职责

1. **监控（Monitoring）**：持续检查主库、从库、其他哨兵是否正常；
2. **通知（Notification）**：实例故障时通过 API 通知管理员或告警系统；
3. **自动故障转移（Automatic failover）**：主库不可用时，**从从库里选一个提升为主库**，并让其他从库指向新主库；
4. **配置提供者（Configuration provider）**：客户端**先连哨兵**问"当前主库是谁"，拿到地址后再连主库。

#### 部署要求（面试常考）

- **哨兵至少 3 个**（**必须是奇数**，且**分布在不同机器上**）—— 因为**选举 leader 需要多数派（majority）**；
- **哨兵本身也是 Redis 进程**（`redis-sentinel` 或 `redis-server --sentinel`），但**不存数据**；
- 哨兵之间通过 **`__sentinel__:hello` 频道（Pub/Sub）** 互相发现和通信；
- **客户端必须支持哨兵**（go-redis 的 `NewFailoverClient`）。

#### 主观下线（SDOWN）与客观下线（ODOWN）—— **核心考点**

```
① 每个哨兵每秒向所有实例（主、从、其他哨兵）发 PING
       ↓
② 如果在 down-after-milliseconds（默认 30 秒）内没收到有效回复
   → 该哨兵单方面认为这个实例"主观下线（SDOWN）"
       ↓
③ 如果下线的是【主库】，该哨兵会向其他哨兵询问：
   "你们也认为这个主库下线了吗？"（SENTINEL is-master-down-by-addr）
       ↓
④ 如果【达到 quorum 数量】的哨兵都认为主库下线了
   → 判定为"客观下线（ODOWN）" → 触发故障转移
```

**`quorum` 的含义（易混点）**：
- `quorum` 是**判定客观下线**所需的票数（配置在哨兵里，如 `sentinel monitor mymaster 127.0.0.1 6379 2` 的最后一个参数就是 quorum=2）；
- **但选举 leader 需要的是 `majority`（多数派），不是 quorum**！即 **`max(quorum, 哨兵总数/2 + 1)`**。

**这是非常经典的陷阱题**：假设有 **5 个哨兵，quorum 配成 2**：
- 判定客观下线只需要 2 票 ✅；
- 但要**选出 leader 需要 3 票（5/2+1）**；
- 如果这时只活下来 2 个哨兵，**虽然能判定客观下线，但永远选不出 leader，故障转移无法完成**。

> **面试话术**：**"quorum 只管'确认主库挂了'，选主需要的是'多数派'。所以哨兵数量要保证：即使挂掉一部分，剩下的仍然构成多数派。这就是为什么哨兵要部署 3 个以上且分散在不同机器上。"**

#### 故障转移（Failover）的完整流程

```
① 判定主库客观下线（ODOWN）
       ↓
② 哨兵之间选举出一个 leader 哨兵（Raft 算法，需要多数派授权）
       ↓
③ leader 哨兵从【从库列表】中挑选一个，提升为新主库（REPLICAOF NO ONE）
       ↓ 挑选规则（按优先级）：
       │   1. 排除已下线的、断线时间超长的从库
       │   2. slave-priority 最小的优先（0 表示永不提升）
       │   3. 复制偏移量（offset）最大的优先 ← 数据最全的
       │   4. runid 最小的优先（字典序，纯兜底）
       ↓
④ leader 把其他从库指向新主库（REPLICAOF <新主库>）
       ↓
⑤ 把【旧主库】标记为从库（等它恢复后，哨兵会让它 REPLICAOF 新主库）
       ↓
⑥ 通过 Pub/Sub 发布切换事件，通知客户端（客户端订阅 +switch-switch 频道）
```

#### 脑裂（Split Brain）与数据丢失

**场景**：主库和哨兵/从库之间的网络分区了。哨兵以为主库挂了，**选了一个新主库**；但**旧主库其实还活着并且在接受客户端写入**（客户端可能还连着它）。等网络恢复，旧主库被降级为从库，**它上面的写入就被清空（进行全量同步）→ 数据丢失**。

**防护措施（Redis 提供了两个参数）**：

```bash
# 主库只有在"至少有 1 个从库"且"这个从库的 lag 不超过 10 秒"时才接受写入
min-replicas-to-write 1
min-replicas-max-lag 10
```

**原理**：网络分区后，旧主库的从库都联系不上了 → `min-replicas-to-write` 条件不满足 → **旧主库拒绝写入**（返回错误）→ 就没有数据可丢了。

> **面试话术**："这是 **CAP 理论在现实中的体现** —— 网络分区（P）发生时，必须在**一致性（C）**和**可用性（A）**之间选。`min-replicas-to-write` 就是**选择一致性**（宁可拒绝写入，也不产生会被回滚的数据）。**对于不能丢数据的场景（比如金币余额），我会开启这两个参数**；对于纯缓存场景，可用性优先，可以不开。"

### 3.3 Cluster：分片集群

#### 为什么要 Cluster？

**主从 + 哨兵解决了"高可用"但没解决"容量"**：单机内存终究有限（一台 64GB 内存的机器，Redis 实例最多用 30GB 左右）。当数据量超过单机内存，或者写入量超过单机能力时，**必须分片（Sharding）**。

#### 核心设计：16384 个哈希槽（Hash Slot）

```
整个集群共有 16384 个槽（slot），每个 key 通过 CRC16 计算后取模决定归属：
    slot = CRC16(key) % 16384

每个主节点负责一部分槽，比如 3 主：
    节点 A: 0     ~ 5460
    节点 B: 5461  ~ 10922
    节点 C: 10923 ~ 16383
```

**为什么是 16384（2¹⁴）而不是 65536？**（**经典拷问，antirez 亲自回答过**）

1. **心跳包的大小**：集群节点间用 gossip 协议交换心跳，**心跳包里有一个 `myslots` 位图（bitmap）记录本节点负责哪些槽**。16384 个槽 = **16384 bit = 2KB**；如果是 65536 个槽 = **8KB**。而心跳包每秒都要发，节点几十个时，**8KB × 节点数 × 每秒次数**的带宽开销不可忽视；
2. **节点数量的现实**：Redis 集群**建议不超过 1000 个节点**，16384 个槽足够分（平均每个节点 16 个槽）；
3. **位图压缩**：当槽数量较少时，**位图可以被压缩**（比如节点只负责连续的一段槽），16384 的压缩效果更好。

#### 节点间的通信：集群总线（Cluster Bus）

- 每个节点额外开一个端口：**服务端口 + 10000**（比如 6379 的服务端口对应 **16379** 的集群总线端口）；
- 用**二进制协议**（比 RESP 更紧凑），用于节点间的**故障检测、配置更新、故障转移授权**；
- 用 **gossip 协议**传播集群状态（每个节点随机选几个节点发消息，最终状态收敛到全集群）；
- **`cluster-node-timeout`**（默认 15000ms）：节点多长时间联系不上就被认为下线。

#### 客户端重定向：MOVED 和 ASK（**核心考点**）

客户端可以连接集群中的**任意节点**，如果请求的 key 不在这个节点上，会收到重定向：

**① MOVED（永久重定向）** —— 槽的归属已经确定了：

```
客户端 → 节点A: GET user:100
节点A  → 客户端: (error) MOVED 866 10.0.0.3:6379
                  ↑ 槽号 866 现在【永久】归 10.0.0.3 管
客户端 → 节点C (10.0.0.3): GET user:100
节点C  → 客户端: "value"
+ 客户端会【更新本地的槽位映射缓存】，下次直接访问节点C
```

**② ASK（临时重定向）** —— 槽正在迁移中：

```
场景：槽 866 正在从 节点A 迁移到 节点C，迁移到一半

客户端 → 节点A: GET user:100
节点A: 这个 key 已经不在我这了 → 计算它的槽 866
       发现槽 866 正在迁移 → 返回 ASK 866 10.0.0.3:6379
客户端 → 节点C: ASKING   （★ 先发 ASKING 命令！）
客户端 → 节点C: GET user:100
节点C: 返回结果
```

**MOVED 和 ASK 的关键区别（面试必答）**：

| | MOVED | ASK |
|---|---|---|
| 含义 | 槽**已经永久**归另一个节点 | 槽**正在迁移**，这个 key 恰好已经迁走了 |
| 客户端行为 | **更新本地槽位缓存**，以后都去新节点 | **不更新缓存**（只这一次去新节点），下次还问原节点 |
| 是否要先发 `ASKING` | ❌ 不需要 | **✅ 必须先发 `ASKING`**（告诉目标节点"允许你处理这个还没正式归属你的 key"） |
| 触发时机 | 槽迁移完成后 / 常规请求 | 槽迁移进行中 |

> **为什么 ASK 需要 `ASKING` 命令？** 因为**正在迁移的槽，目标节点默认会拒绝处理不属于自己的 key**（否则在迁移过程中会破坏一致性）。`ASKING` 是"一次性通行证"——**它只对紧接着的下一条命令有效**。这个细节能答出来，说明真的理解迁移过程。

#### 集群的限制（面试常问"集群有什么坑"）

| 限制 | 说明 | 规避 |
|---|---|---|
| **不支持多 key 操作（跨槽）** | `MGET k1 k2`、`SINTER`、涉及多 key 的 Lua 脚本，**如果 key 不在同一个槽会报 `CROSSSLOT` 错误** | 用 **hash tag**：`{user1000}.name`、`{user1000}.age` —— **只有 `{}` 里的内容参与 CRC16 计算**，所以这两个 key 一定同槽 |
| **不支持多数据库** | 集群模式下只有 `db 0`，`SELECT` 命令不可用 | 用 key 前缀区分业务 |
| **批量操作受限** | `MSET` 要求所有 key 同槽 | 用 hash tag 分组，或拆成多次单 key 操作 |
| **`keys`/`scan` 只扫本节点** | 无法全局遍历 | 遍历所有节点后合并，或维护索引 |
| **事务/Lua 只能操作同槽的 key** | 同上 | hash tag |
| **槽迁移期间有性能抖动** | `MIGRATE` 是阻塞的（迁移单个 key 时用 `MIGRATE`，可以批量但单个 key 大时会阻塞） | 迁移时限制速度（`redis-cli --cluster reshard --cluster-pipeline`）、在低峰期做 |
| **`cluster-require-full-coverage`** | 默认 `yes`：**只要有任何一个槽不可用，整个集群就停止服务**（返回 CLUSTERDOWN）。设为 `no` 则部分槽不可用时其他槽仍可服务（**可用性优先**） | 按业务取舍：**缓存场景建议设为 no**（能服务一部分总比全挂好） |
| **不支持 `SELECT`、`SWAPDB`、`MOVE`** | — | — |

#### 集群的扩容与缩容

```bash
# ① 添加新节点（空的，没有槽）
redis-cli --cluster add-node 10.0.0.4:6379 10.0.0.1:6379

# ② 把新节点加为某个主节点的从库
redis-cli --cluster add-node 10.0.0.5:6379 10.0.0.1:6379 --cluster-slave --cluster-master-id <node-id>

# ③ 重新分片（把一部分槽从旧节点迁到新节点）—— ★ 这是扩容的核心步骤
redis-cli --cluster reshard 10.0.0.1:6379
#    交互式输入：迁移多少个槽、目标节点 ID、从哪些节点迁（all 或指定）

# ④ 删除节点（要先把它负责的槽迁走）
redis-cli --cluster del-node 10.0.0.4:6379 <node-id>

# ⑤ 检查集群状态
redis-cli --cluster check 10.0.0.1:6379
redis-cli --cluster info 10.0.0.1:6379
```

**分片迁移的底层**：集群的槽迁移是**逐 key 的 `MIGRATE`**（把 key 序列化后传给目标节点，然后删除本地 key）。**迁移过程中该槽处于 `MIGRATING` / `IMPORTING` 状态**，客户端会收到 ASK 重定向（见上）。**大 key 会让单个 `MIGRATE` 阻塞**，所以迁移前**要先清理大 key**。

> **面试话术**："集群扩容最需要注意的是**大 key** 和**迁移速度**。大 key 会让 `MIGRATE` 长时间阻塞（虽然是单个 key，但可能几百 MB），所以扩容前我会先用 `--bigkeys` 扫一遍；迁移速度可以用 `--cluster-pipeline` 控制，避免把网络和 CPU 打满。**另外，槽迁移是不可逆的运维操作，一定要先在测试环境演练。**"

#### 数据倾斜（面试加分题）

**"集群的数据倾斜怎么排查和解决？"**

| 倾斜类型 | 原因 | 排查 | 解决 |
|---|---|---|---|
| **槽分配不均** | 扩容时没做 reshard，或者 reshard 分得不匀 | `redis-cli --cluster info` 看每个节点的 `keys` 数和 `slots` 数 | `reshard` 重新分配槽 |
| **key 分布不均（热点 key）** | 某个 key 被高频访问（比如全局计数器、热门商品） | 监控单节点的 CPU/QPS；`redis-cli --hotkeys`（需 LFU 策略） | **把热 key 拆成多个子 key**（`counter:1` ~ `counter:10`，读时随机选一个或求和）；本地缓存；读写分离 |
| **大 key 导致的节点内存倾斜** | 某个 key 特别大（如一个万成员的 Hash） | `redis-cli --bigkeys`、`redis-cli --memkeys` | 拆分大 key（按字段拆成多个 Hash，或用 hash tag 保证同槽的前提下拆） |
| **hash tag 使用不当** | 所有 key 都用同一个 tag（如 `{global}:xxx`）→ **全挤在一个槽** | 看 key 的命名规范 | 重新设计 tag 的粒度（**tag 应该是"需要一起操作的最小集合"**） |

> **面试话术**："数据倾斜的根因通常是"**分片键选得不好**"或者"**热点数据集中**"。我在 FlashRoot 里做渠道商数据分片时，选的是 `customer_id` 作为 hash tag 的基础，这样同一个渠道商的余额、流水、配置都在同一个槽，能支持 `MULTI` 和 Lua 脚本的原子操作；而跨渠道商的统计走的是**单独的汇总 key**，定期异步计算。（**FlashRoot 多级客户**）

### 3.4 本节高频 Q&A

**Q1：主从复制的全量同步和增量同步分别在什么情况下发生？**

> **参考回答**：**首次建立复制**，或者**断线重连时 `runid` 不匹配 / `offset` 已经不在 `repl_backlog` 范围内**，都会触发全量同步（`PSYNC ? -1` 或 `PSYNC <runid> <offset>` 被拒绝 → `+FULLRESYNC`）。**增量同步**只在"**从库断线时间很短，主库的 `repl_backlog` 还保留着缺的那部分数据**"时才成功。所以 `repl_backlog_size` 的大小直接决定了"能容忍多长的断线"，默认 1MB 在高写入场景下太小了。

**Q2：哨兵模式下，客户端怎么知道当前的主库？**

> **参考回答**：客户端**先连接哨兵集群**（配一组哨兵地址），发 `SENTINEL get-master-addr-by-name mymaster` 拿到当前主库地址，再连主库。同时客户端会**订阅哨兵的 `+switch-master` 频道**（或者定期重新查询哨兵），在故障转移发生后**刷新连接**。go-redis 的 `NewFailoverClient` 就是封装了这个逻辑。
>
> **追问："客户端连哨兵有单点问题吗？"** → 客户端配置的是**哨兵列表**（多个地址），任一哨兵可用即可。**哨兵本身也是集群部署的，单个哨兵挂掉不影响。**

**Q3：Cluster 和"客户端分片"（如 Twemproxy/Codis）有什么区别？**

> **参考回答**：**Cluster 是"服务端分片"**（Redis 官方方案）：节点之间知道彼此的存在，客户端收到 MOVED/ASK 重定向，**扩容只需 reshard，客户端无感知**。**客户端分片（Twemproxy）**是代理层做路由，客户端连代理，代理转发到后端 Redis；**Codis** 则是代理 + 哨兵 + 一个中心化的配置存储（ZooKeeper/etcd）。**对比**：
> - Cluster **不支持多 key 跨槽操作、不支持多 DB**，但**无中心节点、无额外组件、官方支持、运维简单**；
> - Twemproxy/Codis **支持多 key 操作（代理层做归并）**，但**引入了额外的代理层（多一跳、可能成为瓶颈）和中心化组件**。
>
> **现在的趋势是优先用 Cluster**（官方维护、生态完善），除非有强需求（比如必须支持多 key 事务）。**Codis 已经基本停止维护了。**

**Q4：Redis 集群能保证数据一致性吗？**

> **参考回答**：**不能，Redis 集群提供的是一致性和可用性的折中，不是强一致**。具体表现在：
> ① **主从复制是异步的** → 主库故障时，未同步到从库的数据会丢；
> ② **集群模式下，主库故障后从库提升也需要时间** → 这期间的写入会失败（`CLUSTERDOWN`）；
> ③ **网络分区时可能出现脑裂**（旧主库仍在写入）→ 需要 `min-replicas-to-write` 缓解；
> ④ **`WAIT` 命令**能等待 N 个副本确认，但**它不保证"确认后不会丢"**（因为主库可能在确认后立刻挂掉，而从库可能还没完成持久化）。
>
> **结论**：**Redis 适合做缓存和"允许极端情况下少量丢失"的场景；真正不能丢的数据（订单、支付、账本），必须以 MySQL 等支持事务的数据库为权威数据源**。这也是 LoveSpouse 金币计费的架构选择——**Redis 负责实时扣减，MySQL 负责最终账本和对账**。（**LoveSpouse 金币计费**）

### 3.5 本节「结合项目」话术提示

- **被问"你们 Redis 是单机还是集群"** → 诚实但具体："LoveSpouse 的规模用**主从 + 哨兵**就够了（单实例的数据量在几 GB 以内），引入 Cluster 的复杂度不划算；**如果数据量或 QPS 增长到单机撑不住，我会用 Cluster**，并且提前用 hash tag 规划 key 的命名（比如按 `session_id` 分组），避免以后迁移时要改业务代码。**架构要匹配当前规模，同时为下一步预留空间。**"（**这个答法既诚实又体现架构思维，比硬吹"我们用了 Cluster"好得多**）
- **被问"怎么保证 Redis 挂了业务不受影响"** → "三层防护：**① 高可用**（哨兵自动切换）；**② 降级**（Redis 不可用时降级到直接查 MySQL，或者返回兜底数据）；**③ 熔断限流**（保护后端的 MySQL 不被压垮）。**关键是明确"Redis 是缓存还是存储"** —— 如果是缓存，挂了大不了回源慢一点；如果是存储（如金币余额），那必须有降级方案和快速恢复能力。"（**通用，能体现稳定性思维**）

---

## 4. 缓存三大问题与缓存一致性（含 Go 代码）

> **这一节是 Redis 面试的"必问区"**，三个问题的名字（穿透、击穿、雪崩）谁都知道，但**面试官真正想看的是：你能不能给出可落地的代码和清晰的方案边界**。所以下面每个问题都给出 Go 的实现。

### 4.1 缓存穿透（Cache Penetration）

#### 定义

**查询一个"根本不存在"的数据**。因为缓存是"读未命中才回源写缓存"，而**不存在的数据永远写不进缓存**，所以每次请求都会**穿透到数据库**。

```
请求 id = -1（不存在）
   ↓
Redis 未命中（因为数据库里也没有，永远不会被缓存）
   ↓
查 MySQL → 没有
   ↓
下次请求 id = -1 → 重复以上流程 ← 💀 每次都要查库
```

**危害**：如果有恶意攻击者**用大量不存在的 key 刷接口**，每次都会打到数据库，**可能直接打垮数据库**。

#### 方案一：缓存空值（最简单有效）

```go
// 缓存空值：即使查不到也缓存一个特殊值，设置较短的过期时间
const nullPlaceholder = "__NULL__"

func GetUser(ctx context.Context, rdb *redis.Client, db *gorm.DB, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    val, err := rdb.Get(ctx, key).Result()
    switch {
    case err == nil:
        if val == nullPlaceholder {
            return nil, ErrUserNotFound   // 命中空值缓存，直接返回，不打 DB
        }
        var u User
        if err := json.Unmarshal([]byte(val), &u); err != nil {
            return nil, err
        }
        return &u, nil
    case errors.Is(err, redis.Nil):
        // 未命中，继续回源
    default:
        return nil, err
    }

    // 回源查库
    var u User
    if err := db.WithContext(ctx).Where("id = ?", id).First(&u).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            // ★ 缓存空值，TTL 短一些（比如 60 秒），避免大量空 key 长期占用内存
            //   同时能防止"数据刚被创建但缓存还是空值"的不一致窗口过长
            rdb.Set(ctx, key, nullPlaceholder, 60*time.Second)
            return nil, ErrUserNotFound
        }
        return nil, err
    }

    // 正常缓存，TTL 加随机抖动防雪崩（见 4.3）
    b, _ := json.Marshal(u)
    rdb.Set(ctx, key, b, 30*time.Minute+randDuration(5*time.Minute))
    return &u, nil
}
```

**优缺点**：
- ✅ 实现简单、效果立竿见影；
- ❌ **额外的内存开销**（每个不存在的 key 都占一份）；
- ❌ **数据一致性变差**（如果这个 id 后来真的被创建了，在 TTL 到期前会一直返回"不存在"）→ 所以**空值的 TTL 要短**（60 秒以内），并且**数据创建时要主动删掉这个空值缓存**。

#### 方案二：布隆过滤器（Bloom Filter）—— **推荐用于大规模场景**

**原理**：一个**位数组 + N 个哈希函数**。

```
插入元素 "user:100"：
  用 3 个哈希函数算出 3 个位置 → 把位数组的这 3 位都置为 1

查询元素 "user:200"：
  用同样的 3 个哈希函数算出 3 个位置
    - 如果有任何一位是 0 → 【一定不存在】✅ 可以放心拒绝
    - 如果三位都是 1  → 【可能存在】（有可能是别的元素把这个位置成 1 的）
```

**关键特性（面试必答）**：

| 特性 | 说明 |
|---|---|
| **说"不存在"一定准确** | 没有假阴性（False Negative） |
| **说"存在"可能是错的** | 有假阳性（False Positive），概率很小 |
| **不支持删除** | 因为一个位可能被多个元素共享，删掉会影响别人。**变体：Counting Bloom Filter（用计数代替位）** |
| **空间效率极高** | 存储 100 万元素、误判率 1% 只需要约 **1.14MB**（约 9.6 bit/元素） |

**误判率公式**（面试可以列出来，很加分）：

```
误判率 p ≈ (1 - e^(-kn/m))^k
   m = 位数组长度, n = 元素个数, k = 哈希函数个数

最优的哈希函数个数：k = (m/n) × ln2 ≈ 0.693 × (m/n)
给定 n 和期望的 p，所需的位数：m = -n × ln(p) / (ln2)² ≈ 1.44 × n × log₂(1/p)
```

**在 Redis 中怎么用？**

```bash
# 方案 A：RedisBloom 模块（官方模块，推荐）
# 需要单独安装/加载模块，云厂商通常提供了
127.0.0.1:6379> BF.RESERVE user_filter 0.01 1000000    # 误判率 0.01，预计 100 万元素
127.0.0.1:6379> BF.ADD user_filter user:100
(integer) 1
127.0.0.1:6379> BF.EXISTS user_filter user:200
(integer) 0     # 一定不存在
127.0.0.1:6379> BF.MADD user_filter user:101 user:102 user:103
```

```go
// 方案 B：Go 侧实现（用 bits-and-blooms/bloom 库）+ 存到 Redis 的 Bitmap
// 适合不想引入模块的场景；也可以纯本地内存（适合单实例/数据量小）

// 用 Redis Bitmap 做位数组的简化实现（生产建议用成熟的库）
const (
    bloomKey    = "bloom:user"
    bloomBits   = 1 << 24 // 1600 万位 ≈ 2MB，误判率约 1%（可存约 150 万元素）
    hashCount   = 10
)

// 计算第 i 个哈希位置（用双重哈希构造多个独立哈希，Kirsch-Mitzenmacher 优化）
func bloomPositions(data string, hashCount int) []uint64 {
    h1 := fnv.New64a()
    h1.Write([]byte(data))
    a := h1.Sum64()

    h2 := fnv.New64()
    h2.Write([]byte(data))
    b := h2.Sum64()

    pos := make([]uint64, hashCount)
    for i := 0; i < hashCount; i++ {
        pos[i] = (a + uint64(i)*b) % bloomBits
    }
    return pos
}

// 批量写入（用 Pipeline 减少 RTT）
func BloomAdd(ctx context.Context, rdb *redis.Client, items []string) error {
    pipe := rdb.Pipeline()
    for _, item := range items {
        for _, p := range bloomPositions(item, hashCount) {
            pipe.SetBit(ctx, bloomKey, int64(p), 1)
        }
    }
    _, err := pipe.Exec(ctx)
    return err
}

// 判断是否存在（只要有一位是 0，就一定不存在）
func BloomExists(ctx context.Context, rdb *redis.Client, item string) (bool, error) {
    pipe := rdb.Pipeline()
    cmds := make([]*redis.IntCmd, 0, hashCount)
    for _, p := range bloomPositions(item, hashCount) {
        cmds = append(cmds, pipe.GetBit(ctx, bloomKey, int64(p)))
    }
    if _, err := pipe.Exec(ctx); err != nil {
        return false, err
    }
    for _, cmd := range cmds {
        if cmd.Val() == 0 {
            return false, nil // ★ 一定不存在
        }
    }
    return true, nil // 可能存在（需继续查缓存/DB 确认）
}
```

**使用布隆过滤器的完整流程**：

```
① 【写入时】数据创建成功 → 加入布隆过滤器
② 【读取时】
   请求 key
      ↓
   布隆过滤器判断 → "不存在" → 【直接返回空，不查 Redis 也不查 DB】✅
      ↓
                   "可能存在" → 走正常的缓存 + 回源流程
③ 【启动时】需要把已有的全量数据预热进布隆过滤器（或者用 Redis 的持久化 + 定期重建）
```

**优缺点**：
- ✅ **内存占用极小**（100 万元素只要 1MB 多）、**拦截率高**；
- ❌ **实现复杂**（要保证数据写入和布隆过滤器同步）；
- ❌ **不支持删除**（数据删了，布隆过滤器里还在 → 需要定期重建，或者用 Counting Bloom）；
- ❌ **有极小的误判率**（可以调，但代价是内存）。

> **该选哪个？（面试官爱问"你选哪个，为什么"）** → **"看数据是"动态增长"还是"相对固定"。如果是要防"恶意刷不存在的 ID"（ID 空间是已知的、相对固定的），布隆过滤器最合适——内存占用极小，拦截率接近 100%；如果业务本身就允许查不到（比如查询条件组合千变万化，无法事先枚举），那就用空值缓存，因为它不需要维护额外的结构。**实践中我经常**两者结合**：先用布隆过滤器挡掉大部分明显的非法请求，再用空值缓存兜底那些"布隆说可能存在但实际不存在"的请求。"**（**这个"两者结合"的答法很出彩**）

> **结合项目话术**：LoveSpouse 的对话接口会接收 `role_id`、`session_id` 等参数。**对于这类"参数空间可控"的场景，布隆过滤器是很好的防穿透手段**；而对于"用户随便输入的关键词搜索"，我们用**参数校验 + 空值缓存**。**关键是根据参数空间的特征选方案。**（**LoveSpouse 数字人**）

#### 方案三：参数校验（最基础但最重要）

```go
// 最基本的一道防线：在入口就把明显非法的参数拒掉
func validateID(id int64) error {
    if id <= 0 || id > maxValidID {
        return ErrInvalidParam   // 直接返回，不查任何存储
    }
    return nil
}
```

> **面试话术**："其实**参数校验是第一道也是最重要的一道防线**——很多穿透问题本质是接口设计不严谨。**另外还有一个常被忽略的点：权限校验应该在查缓存之前做**。如果一个用户没有权限访问某个资源，直接返回 403 就行，根本不需要查缓存，这既是安全问题，也顺便防了穿透。"

### 4.2 缓存击穿（Cache Breakdown / Hotspot Invalid）

#### 定义

**某个热点 key 在失效的瞬间，大量并发请求同时打过来**，全部未命中，**同时去查数据库并重建缓存**。

```
T0: 热点 key "hot:product:1" 有 10 万 QPS
T1: 这个 key 过期了（被删除）
T2: 10 万个并发请求同时发现缓存未命中
T3: 💀 10 万个请求同时查 MySQL → 数据库瞬间被打爆
```

**和穿透的区别**（面试常混）：
- **穿透**：查的是**不存在**的数据，缓存永远不生效；
- **击穿**：查的是**存在且是热点**的数据，只是**在失效瞬间**出问题。

#### 方案一：互斥锁（Mutex）—— 保证只有一个请求回源

```go
// 用 Redis 分布式锁保证"只有一个请求去重建缓存"
func GetProductWithMutex(ctx context.Context, rdb *redis.Client, db *gorm.DB, id int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", id)
    lockKey := fmt.Sprintf("lock:product:%d", id)

    // ① 先查缓存
    if p, err := getFromCache(ctx, rdb, key); err == nil {
        return p, nil
    }

    // ② 未命中，尝试获取互斥锁（★ 用 SET NX PX，一条原子命令）
    token := uuid.NewString()   // 唯一标识，用于安全释放
    ok, err := rdb.SetNX(ctx, lockKey, token, 10*time.Second).Result()
    if err != nil {
        return nil, err
    }

    if !ok {
        // ③ 没抢到锁：说明有别的请求正在重建缓存
        //    短暂等待后重试（读缓存），避免直接打 DB
        time.Sleep(50 * time.Millisecond)
        if p, err := getFromCache(ctx, rdb, key); err == nil {
            return p, nil
        }
        // 重试仍然没有（可能是重建失败），降级：直接查 DB（但要有数量限制）
        return queryDB(ctx, db, id)
    }

    // ④ 抢到锁：查 DB 并写缓存
    defer releaseLock(ctx, rdb, lockKey, token)   // ★ 用 Lua 脚本安全释放

    // 双重检查：可能别的请求已经重建好了
    if p, err := getFromCache(ctx, rdb, key); err == nil {
        return p, nil
    }

    p, err := queryDB(ctx, db, id)
    if err != nil {
        return nil, err
    }
    b, _ := json.Marshal(p)
    rdb.Set(ctx, key, b, 30*time.Minute)
    return p, nil
}
```

**优缺点**：
- ✅ **强一致性好**（保证同一时刻只有一个请求回源，DB 压力可控）；
- ✅ 实现相对简单；
- ❌ **有额外的锁开销**（多一次 Redis 往返）；
- ❌ **有死锁风险**（必须给锁设置过期时间 + Lua 安全释放）；
- ❌ **吞吐量受限**（其他请求要等待，虽然等待时间很短）。

> **这是最常用的方案**，尤其在"数据一致性要求高 + 热点集中"的场景。

#### 方案二：逻辑过期（Logical Expiration）—— **永不真正的过期**

**思路**：**缓存永不过期（不设 TTL）**，而是**在 value 里存一个"逻辑过期时间"字段**。读取时判断是否逻辑过期：

```
① 读缓存 → 命中 → 检查 value 里的 expire_at
    ├── 未逻辑过期 → 直接返回数据（正常路径，无锁）✅
    └── 已逻辑过期 → 尝试获取互斥锁
            ├── 拿到锁 → 【异步】开一个 goroutine 去重建缓存，当前请求【先返回旧数据】
            └── 没拿到锁 → 也【直接返回旧数据】
```

```go
// 存进 Redis 的结构（把过期时间和数据打包在一起）
type cacheEntry struct {
    Data     json.RawMessage `json:"data"`
    ExpireAt int64           `json:"expire_at"` // 逻辑过期时间戳（秒）
}

func GetProductLogicalExpire(ctx context.Context, rdb *redis.Client, db *gorm.DB, id int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", id)

    val, err := rdb.Get(ctx, key).Result()
    if errors.Is(err, redis.Nil) {
        // 首次访问，缓存里什么都没有 —— 这时候还是得同步加载（或者直接返回空）
        return loadAndCache(ctx, rdb, db, id)
    }
    if err != nil {
        return nil, err
    }

    var entry cacheEntry
    if err := json.Unmarshal([]byte(val), &entry); err != nil {
        return nil, err
    }

    // ★ 关键：没过逻辑过期，直接返回（性能最好，无锁无等待）
    if time.Now().Unix() < entry.ExpireAt {
        var p Product
        json.Unmarshal(entry.Data, &p)
        return &p, nil
    }

    // ★ 逻辑过期了：返回旧数据 + 异步重建
    go func() {
        // 用互斥锁保证只有一个 goroutine 去重建
        lockKey := fmt.Sprintf("lock:rebuild:product:%d", id)
        token := uuid.NewString()
        bgCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()

        ok, err := rdb.SetNX(bgCtx, lockKey, token, 10*time.Second).Result()
        if err != nil || !ok {
            return // 别人在重建，直接放弃
        }
        defer releaseLock(bgCtx, rdb, lockKey, token)

        // 重新查库并写回（带新的逻辑过期时间）
        var p Product
        if err := db.WithContext(bgCtx).Where("id = ?", id).First(&p).Error; err != nil {
            log.Errorf("rebuild cache failed: %v", err)
            return
        }
        data, _ := json.Marshal(p)
        newEntry, _ := json.Marshal(cacheEntry{
            Data:     data,
            ExpireAt: time.Now().Add(30 * time.Minute).Unix(), // 新的逻辑过期时间
        })
        // ★ 注意：不设置 Redis 的 TTL（永不过期），靠逻辑过期控制
        rdb.Set(bgCtx, key, newEntry, 0)
    }()

    // 立即返回旧数据（短暂的不一致是可以接受的）
    var p Product
    json.Unmarshal(entry.Data, &p)
    return &p, nil
}
```

**优缺点**：

| | 互斥锁 | 逻辑过期 |
|---|---|---|
| **一致性** | **强**（重建完成前其他请求等待/查库） | **弱**（会返回旧数据，有一段时间的不一致） |
| **性能** | 中（等待锁 + 缓存重建） | **高（几乎无等待，永远不阻塞）** |
| **实现复杂度** | 低 | 中（要改造 value 结构 + 后台 goroutine） |
| **适用场景** | 数据一致性要求高、热点 key 不多 | **超高并发的热点数据**（秒杀商品、首页配置、榜单） |

> **面试话术**："这两个方案的本质区别是**"一致性"和"可用性"的取舍**。互斥锁是**宁可让请求等一等，也要保证拿到的是最新数据**；逻辑过期是**宁可返回旧数据，也不让任何请求等待**。**我的选择标准是：如果这个数据"过期几秒是错误的"（比如库存、余额），必须用互斥锁；如果"过期几秒只是稍旧"（比如商品标题、榜单），逻辑过期更好，因为它能扛住任何并发。"**（**这个答法把技术选型上升到业务语义，是面试官最想听到的**）

### 4.3 缓存雪崩（Cache Avalanche）

#### 定义（两种）

**① 大批 key 同时失效**：

```
凌晨 0 点，运维批量预热了一批 key，都设置了 1 小时的 TTL
    ↓
凌晨 1 点，这批 key 同时过期
    ↓
💀 所有请求同时打到数据库
```

**② 缓存服务整体不可用**（更严重）：

```
Redis 集群挂了
    ↓
💀 所有请求直接打到数据库 → 数据库瞬间被打爆 → 整个系统雪崩
```

#### 方案一：TTL 加随机抖动（解决"同时失效"）

```go
// ❌ 错误：所有 key 用同一个 TTL，会同时失效
rdb.Set(ctx, key, val, 30*time.Minute)

// ✅ 正确：加随机抖动
func jitterTTL(base time.Duration, jitter time.Duration) time.Duration {
    // rand.Int63n(jitter) 返回 [0, jitter)
    return base + time.Duration(rand.Int63n(int64(jitter)))
}
rdb.Set(ctx, key, val, jitterTTL(30*time.Minute, 5*time.Minute))
// → TTL 落在 [30min, 35min) 之间，避免同时失效
```

**进阶做法：分层 TTL** —— 对同一批数据，**给不同的 key 设置不同的基础 TTL**（比如"热点数据 10 分钟、普通数据 30 分钟、冷数据 2 小时"），进一步打散失效时间。

#### 方案二：多级缓存（解决"缓存服务挂了"）

```
请求
  ↓
① 本地缓存（进程内，如 bigcache / freecache / go-cache）  ← 纳秒级，但容量小、多实例不一致
  ↓ 未命中
② Redis 缓存（分布式，容量大、一致）                      ← 毫秒级
  ↓ 未命中
③ MySQL / 数据库
```

**本地缓存的三个关键问题（面试要能答）**：

| 问题 | 解法 |
|---|---|
| **多实例数据不一致** | 用 Redis 的 **Pub/Sub** 广播失效消息（每个实例订阅，收到就删本地缓存）；或者接受**短 TTL**（如 5~10 秒）的最终一致 |
| **容量受限** | 用 **LRU 淘汰**（bigcache/freecache 都支持）+ 只缓存**极热**的数据（Top 1% 的 key 承载 90% 的请求） |
| **内存占用** | 单独限制本地缓存的大小，避免和业务内存争抢 |

**本地缓存的 Go 选型**：
- `github.com/allegro/bigcache`：**为高并发设计**，分片锁、无 GC 压力（存 `[]byte`），适合大量小对象；
- `github.com/coocood/freecache`：类似，支持过期；
- `github.com/patrickmn/go-cache`：简单易用，但**有 GC 压力**（存 `interface{}`），适合小规模；
- `golang.org/x/sync/singleflight`：**不是缓存，但配合缓使用**（见方案四）。

#### 方案三：熔断、降级、限流（**这是"止血"的关键**）

**核心思想**：**当数据库压力过大时，主动放弃一部分请求，保住数据库**。

```go
// 用 singleflight 合并并发请求（同一个 key 的并发回源只执行一次）
// ★ 这是解决缓存击穿/雪崩的"终极武器"，比互斥锁更优雅
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func GetWithSingleflight(ctx context.Context, key string) (any, error) {
    // 同一个 key 的并发调用，只有一个会真正执行 fn，其他都等待并共享结果
    v, err, _ := g.Do(key, func() (any, error) {
        // 这个函数在同一个 key 上只会被一个 goroutine 执行
        // 其他 goroutine 会阻塞等待结果（不是返回旧值）
        return loadFromDBAndCache(ctx, key)
    })
    return v, err
}
```

> **`singleflight` 是 Go 生态里非常优雅的方案**，它把"缓存击穿"的防护**从分布式锁降级到了进程内的请求合并**——**同一个进程内的并发请求合并成一次回源，多个进程之间各自回源一次**。对于"热点 key + 多实例"的场景，它把回源次数从"QPS × 实例数"降到"实例数"，**效果已经足够好，而且没有任何分布式锁的开销和风险**。
>
> **面试话术**："解决缓存击穿我有一个更轻量的方案：**Go 的 `singleflight`**。它在进程内把同一 key 的并发请求合并成一次回源，实现简单、无锁开销。**它的局限是只在单进程内生效**，所以如果是"一个热点 key 打满整个集群"的场景，还需要配合 Redis 互斥锁或逻辑过期。"（**这个答法能体现你对 Go 生态的熟悉，是 Go 岗位的加分项**）

**降级代码示例**：

```go
// 降级：当 Redis 不可用或回源失败时，返回兜底数据而不是报错
func GetProductWithFallback(ctx context.Context, rdb *redis.Client, db *gorm.DB, id int64) (*Product, error) {
    // 尝试正常路径
    p, err := GetProductWithMutex(ctx, rdb, db, id)
    if err == nil {
        return p, nil
    }

    // Redis 挂了 / 回源失败 → 降级
    log.Warnf("fallback triggered: %v", err)

    // 策略 1：查本地缓存（如果有多级缓存）
    if p := localCache.Get(id); p != nil {
        return p, nil
    }
    // 策略 2：返回兜底数据（比如默认配置、空对象）
    return defaultProduct(), nil
    // 策略 3（谨慎）：直接查 DB —— 但这会加剧 DB 压力，通常需要配合限流
}
```

**限流保护数据库**（见第 5.2 节）：

```go
// 在回源路径上加一个"每秒最多 N 次"的限流，保护数据库
if !dbLimiter.Allow() {
    return nil, ErrTooManyRequests   // 直接拒绝，不给 DB 压力
}
```

#### 方案四：缓存高可用（治本）

- **Redis 主从 + 哨兵**（自动故障转移）；
- **Redis Cluster**（分片 + 高可用）；
- **多级缓存**（Redis 挂了还有本地缓存）；
- **做好容量规划**（`maxmemory` + 淘汰策略，避免 OOM 触发大量淘汰）。

#### 三者的对比总结（**面试可以画这张表**）

| | 穿透 | 击穿 | 雪崩 |
|---|---|---|---|
| **问题数据** | **不存在**的数据 | **存在且是热点**的数据 | **大批**数据 |
| **触发条件** | 恶意/异常查询 | 热点 key 失效的瞬间 | 大批 key 同时失效 / Redis 挂了 |
| **核心方案** | 布隆过滤器、空值缓存、参数校验 | 互斥锁、逻辑过期、singleflight | TTL 抖动、多级缓存、熔断降级、高可用 |
| **一句话记忆** | **查不到** | **一个热点炸了** | **一片全炸了** |

### 4.4 缓存与数据库的一致性（**超高频**）

> 这个问题的标准答案是**分层次的**：先讲清楚"没有绝对强一致，只有最终一致"，再讲具体方案。**面试官最反感的是上来就说"用延迟双删"——因为延迟双删本身有很多问题。**

#### 4.4.1 四种更新策略的对比（**先建立分析框架**）

| 策略 | 操作 | 问题 |
|---|---|---|
| **① 先更新缓存，再更新数据库** | ❌ **强烈不推荐** | 两个请求并发时，缓存和 DB 的更新顺序可能颠倒 → 缓存是旧值 |
| **② 先更新数据库，再更新缓存** | ⚠️ 不推荐 | 同上，且**更新缓存的成本高**（可能要复杂计算）；并发写时缓存可能被旧值覆盖 |
| **③ 先删除缓存，再更新数据库** | ⚠️ 有问题 | 并发读会导致缓存被写入旧值（见下） |
| **④ 先更新数据库，再删除缓存（Cache Aside）** | ✅ **推荐** | 极端并发下也有问题，但概率最低，且可用兜底手段 |

**为什么"删除"而不是"更新"缓存？（必答）**

1. **删除是幂等的，更新不是**：两次删除和一次删除结果相同；但两次更新如果顺序颠倒（缓存更新 B、缓存更新 A），结果就错了；
2. **避免"更新了缓存但数据库回滚了"**：如果事务回滚，缓存里的新值就是脏数据；
3. **懒加载的思想**：删除后下次读会从 DB 重建，**只有真正被访问的数据才会进缓存**（避免把冷数据写进缓存浪费内存）。

#### 4.4.2 "先删缓存再更新数据库"的问题

```
时刻 1  请求 A（写）：删除缓存 user:1
时刻 2  请求 B（读）：缓存未命中 → 查 DB → 读到【旧值】
时刻 3  请求 A（写）：更新 DB 为【新值】
时刻 4  请求 B（读）：把【旧值】写入缓存  ← 💀 缓存里是旧值，长期不一致
```

**问题的本质**：读请求在"删缓存"和"更新 DB"之间的**窗口期**读到了旧值，并把这个旧值写进了缓存。

#### 4.4.3 "先更新数据库再删缓存"的问题（**Cache Aside，推荐但也不完美**）

```
时刻 1  请求 B（读）：缓存刚好【失效】（过期了）
时刻 2  请求 B（读）：查 DB → 读到【旧值】（此时 A 还没写完）
时刻 3  请求 A（写）：更新 DB 为【新值】
时刻 4  请求 A（写）：删除缓存（但缓存本来就是空的，等于没删）
时刻 5  请求 B（读）：把【旧值】写入缓存  ← 💀 缓存里又是旧值
```

**注意**：这个场景要满足两个条件才发生——**① 缓存恰好失效；② 读请求在写请求之前查到 DB，但在写请求之后才写缓存**。**这个窗口极窄（要求读操作耗时 > 写操作耗时，且读在写之前启动）**，所以**发生概率远低于"先删缓存"**。

> **面试话术（非常重要，体现你的判断力）**："**Cache Aside 不是完美的，但它的不一致窗口是几种方案里最小的**。而且实际上还有两个关键的缓解因素：① **读操作通常比写操作快得多**（因为读是主键查询，写可能要更新多个表/索引），所以"读在写之前开始、在写之后结束"的窗口很难出现；② **缓存都有 TTL**，即使真的不一致，也会在 TTL 到期后自愈。**所以工程上通常的做法是：用 Cache Aside + 合理设置 TTL + 关键数据加兜底手段。**"

#### 4.4.4 兜底手段一：延迟双删

```go
// 延迟双删：更新 DB 前后各删一次缓存，第二次延迟执行
func UpdateUserWithDoubleDelete(ctx context.Context, rdb *redis.Client, db *gorm.DB, u *User) error {
    key := fmt.Sprintf("user:%d", u.ID)

    // ① 先删一次缓存（让后续的读请求回源到【旧值】但也有可能读到新值）
    rdb.Del(ctx, key)

    // ② 更新数据库
    if err := db.WithContext(ctx).Save(u).Error; err != nil {
        return err
    }

    // ③ 延迟再删一次（覆盖"读请求把旧值写回缓存"的窗口）
    //    ★ 延迟时间要 > 一次读操作的最长耗时（通常 300ms ~ 1s）
    time.AfterFunc(500*time.Millisecond, func() {
        bgCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        defer cancel()
        rdb.Del(bgCtx, key)
    })

    return nil
}
```

**延迟双删的问题（面试官爱追问）**：
- **延迟时间不好确定**（太短盖不住窗口，太长影响一致性）；
- **第二次删除失败怎么办**（需要重试机制）；
- **`time.AfterFunc` 在进程重启时会丢失** → 生产上应该用**延迟消息队列**（RabbitMQ 延迟队列 / Redis ZSet 延时任务）而不是本地定时器。

> **面试话术**："延迟双删是个**经验性的补丁**，它能降低不一致的概率但**不能消除**（因为延迟时间无法精确确定）。我一般把它作为**过渡方案**，如果业务真的要求很强的缓存一致性，我更倾向于用 **binlog 订阅（Canal）** 的方案。"

#### 4.4.5 兜底手段二：订阅 binlog（Canal）—— **推荐的进阶方案**

**思路**：**业务代码只写数据库，不碰缓存**。有独立的组件订阅 MySQL 的 binlog，解析出数据变更，然后**删除对应的缓存**。

```
应用 → 写 MySQL（只写库，不管缓存）
         ↓
      binlog
         ↓
    Canal / go-mysql（伪装成 MySQL 从库，订阅 binlog）
         ↓
    投递到 MQ（Kafka/RabbitMQ）
         ↓
    消费者 → 删除 / 更新 Redis 缓存
```

**优点**：
- **业务代码零侵入**（不用在每个写操作后面加删缓存的代码）；
- **不会漏**（只要 binlog 有，就一定会触发缓存删除，哪怕是别的系统/人直接改的数据）；
- **天然支持重试**（MQ 的消费失败可以重试，配合幂等）；
- **顺序有保障**（binlog 是顺序的，同一个 key 的多次变更按顺序处理）。

**缺点**：
- **引入额外组件**（Canal + MQ），运维复杂度上升；
- **有延迟**（binlog → Canal → MQ → 消费者的链路，通常几十毫秒到几百毫秒）；
- **需要处理幂等**（同一条 binlog 可能被消费多次）。

> **面试话术**："Canal 方案的**最大价值是"不漏"**。用应用层代码删缓存，最大的风险是**某条更新路径忘了删缓存**（比如有人直接在数据库里改了数据、或者新来的同事写了一个新的更新接口忘了处理缓存）—— 这种"人的失误"在生产里非常常见，而 Canal 从 binlog 出发，**只要数据库变了就一定会触发缓存更新**，这是它不可替代的优势。**代价是引入了组件和延迟，所以要看业务是否值得。**"
>
> **结合项目话术**：POS 数据贯通里的数据是**通过 RabbitMQ 消费多个产品线的数据后写库**的，如果这些数据要缓存，**应用层的写入口非常多（多个产品线的适配器）**，逐个加"删缓存"逻辑既容易漏也不好维护。**这种场景我会倾向于用 binlog 订阅的方式**——因为写库的入口收敛在消费端，而 binlog 是唯一的真相来源。（**POS 数据贯通** —— 这个结合非常自然）

#### 4.4.6 兜底手段三：加锁串行化（最强一致性，性能最差）

```go
// 读写都加分布式锁 → 保证强一致，但性能极差
func UpdateUserStrong(ctx context.Context, rdb *redis.Client, db *gorm.DB, u *User) error {
    lock := NewRedisLock(rdb, fmt.Sprintf("lock:user:%d", u.ID), 10*time.Second)
    if err := lock.Acquire(ctx); err != nil {
        return err
    }
    defer lock.Release(ctx)

    // 更新 DB
    if err := db.WithContext(ctx).Save(u).Error; err != nil {
        return err
    }
    // 删缓存
    rdb.Del(ctx, fmt.Sprintf("user:%d", u.ID))
    return nil
}
// 读操作也要加同一把锁 → 并发度极低
```

**结论**：**只在"绝对不能出现不一致"的场景用**（比如余额、库存的关键计算），且要**限制锁的粒度**。

#### 4.4.7 最终结论（**面试的标准收尾**）

> **"缓存一致性没有银弹，它是一个"根据业务对不一致的容忍度来选择方案"的问题。我的决策框架是：**
> **① 如果数据能容忍最终一致（比如商品描述、用户昵称）** → **Cache Aside（先写库再删缓存）+ TTL 兜底**就够了，不要过度设计；
> **② 如果"不一致会导致业务错误"（比如库存、余额）** → **不要把正确性寄托在缓存上**。正确做法是**让数据库做权威**：扣减操作用数据库的原子更新（`UPDATE ... SET stock = stock - 1 WHERE stock >= 1`）或者 Redis 的 Lua 原子操作 + 异步落账，缓存只做"展示加速"，**关键判断永远查权威数据源**；
> **③ 如果业务真的需要"缓存和数据库强一致"** → 那说明**这个数据不应该被缓存**，或者应该用**读写锁串行化**（接受性能代价）。"
>
> **这个答案的核心思想是："不要用缓存去解决一致性问题——缓存是性能工具，一致性由数据库保证。"** 这句话说出来，面试官会觉得你想清楚了这个问题的本质。

### 4.5 本节高频 Q&A

**Q1：缓存和数据库的双写，你会怎么选方案？**

> **参考回答**：见 4.4.7 的决策框架。**总结一句：默认用 Cache Aside（先写库再删缓存）+ TTL 兜底；数据重要就加 Canal 订阅 binlog 保证不漏；绝对不能不一致就别缓存。**

**Q2：为什么要给缓存设置过期时间？不设行不行？**

> **参考回答**：**必须设，即使一致性要求很高也要设一个较长的 TTL**。原因：**TTL 是"最终一致性的最后兜底"**——不管前面的一致性方案出了什么 bug（代码写错了、MQ 丢消息了、Canal 挂了），只要 TTL 到期，缓存就会从数据库重建，不一致**一定会自愈**。**没有 TTL 的缓存，一旦不一致就是"永久不一致"**，需要人工介入。**所以"所有缓存必须有过期时间"是我认为最重要的一条缓存使用纪律。**

**Q3：缓存预热怎么做？**

> **参考回答**：系统启动/大促前，**主动把热点数据加载进缓存**，避免"冷启动时大量请求穿透到数据库"。做法：
> ① **启动时预热**（应用启动后异步加载热点数据，注意**限速**避免启动瞬间打爆 DB）；
> ② **定时预热**（比如每天凌晨刷一遍日榜数据）；
> ③ **大促前手动预热**（提前几小时或一天，用脚本把数据灌进缓存）；
> ④ **不要把预热放在应用启动的阻塞路径上**（会导致启动超时、健康检查失败）。
>
> **注意**：预热要**控制速率**（比如每秒 1000 个 key），避免"预热本身就打爆数据库"。

**Q4：什么是缓存污染（Cache Pollution）？**

> **参考回答**：指**不常访问的数据挤占了缓存空间，把热点数据淘汰掉了**。典型场景是**缓存穿透攻击**、**批量全表扫描式的查询**（比如运营拉全量数据导报表），这些请求写入大量冷数据到缓存，导致热点数据被 LRU 淘汰。**解决**：① **写缓存前判断"值不值得缓存"**（比如只缓存被访问 2 次以上的数据，或者限制单次批量写缓存的 key 数量）；② 用 **LFU 淘汰策略**（按访问频率淘汰）；③ 大报表类查询**不走缓存**。

**Q5：Redis 挂了对业务有什么影响？怎么设计降级？**

> **参考回答**：取决于 Redis 承担的角色：
> - **纯缓存**：性能下降（全部回源），但功能可用 → **降级方案是"限流 + 直接查库"**，前提是**数据库能扛住（要提前做过容量评估和压测）**；
> - **存了唯一数据**（如会话、余额）：**功能不可用** → 必须有**高可用（哨兵/Cluster）+ 持久化 + 快速恢复预案**；
> - **作为分布式锁**：锁失效 → 可能导致重复执行 → **关键操作要有数据库层的幂等兜底**（唯一索引）。
>
> **我的设计原则**：**"Redis 可以慢，但不能成为单点故障；关键正确性不能依赖 Redis。"** 具体做法是：**所有依赖 Redis 的正确性保证（幂等、锁、去重），都在数据库层有唯一约束或状态机兜底。**

### 4.6 本节「结合项目」话术提示

| 问题 | 钩子 | 话术 |
|---|---|---|
| 你用缓存时怎么防止不一致？ | **LoveSpouse 金币计费** | "LoveSpouse 的金币余额是**不能容忍不一致**的数据，所以我没有把它做成"缓存"，而是**让 Redis 承担实时扣减的职责（权威的实时值），MySQL 存流水作为账本** —— 然后通过**定时对账任务**来校验两者是否一致，不一致就告警并修正。**这里的关键决策是：我明确区分了"实时值"和"账本"两个语义，而不是含糊地叫它"缓存"。**" |
| 怎么防止缓存击穿？ | **LoveSpouse / FlashRoot** | "LoveSpouse 的付费互动推荐配置是一个热点（所有用户都要读），我用的是 **singleflight + 逻辑过期**的组合：进程内合并并发请求，值里带逻辑过期时间，过期后异步重建并返回旧值。**这样即使用户量突增，也不会有一瞬间打爆数据库。**" |
| 缓存穿透怎么处理？ | **通用** | 见 4.1 的三个方案 + 参数校验的补充。 |

---

## 5. 应用场景：分布式锁 / 限流 / 计数与排行榜

### 5.1 分布式锁（**超高频**）

#### 5.1.1 为什么需要分布式锁？

**单机锁（`sync.Mutex`）只能保证"同一个进程内"的互斥**。多实例部署时，同一个业务逻辑会被多个进程并发执行 → 需要**跨进程的互斥**。

**典型场景**（**这些都能和岗位挂钩**）：
- **防止定时任务重复执行**（OA 系统的"超时自动审批"、"每日统计"，多实例部署时每个实例都会触发定时任务）；
- **防止重复提交 / 重复处理**（同一笔审批被点了两次"同意"）；
- **结算批次生成**（**FlashRoot 的结算**，绝对不能生成两个批次）；
- **缓存重建**（见 4.2）。

#### 5.1.2 正确的加锁与解锁实现（Go + go-redis v9）

```go
package lock

import (
    "context"
    "errors"
    "time"

    "github.com/google/uuid"
    "github.com/redis/go-redis/v9"
)

var (
    ErrLockNotAcquired = errors.New("lock not acquired")
    ErrLockLost        = errors.New("lock lost before release")
)

// ★ 释放锁的 Lua 脚本：必须"先比较再删除"，保证原子性
//   为什么需要比较？如果锁已经过期，且别人已经拿到了新锁，
//   直接 DEL 会把【别人的锁】删掉 → 严重 bug
const unlockScript = `
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
`

type RedisLock struct {
    rdb   *redis.Client
    key   string
    token string        // ★ 唯一标识持有者（防止误删别人的锁）
    ttl   time.Duration
}

func NewRedisLock(rdb *redis.Client, key string, ttl time.Duration) *RedisLock {
    return &RedisLock{
        rdb:   rdb,
        key:   key,
        token: uuid.NewString(), // 每个锁实例一个唯一 token
        ttl:   ttl,
    }
}

// 加锁：★ 用一条 SET key value NX PX ttl 原子命令
//   ❌ 不要用 SETNX + EXPIRE 两条命令（中间崩溃会导致死锁）
func (l *RedisLock) Acquire(ctx context.Context) error {
    ok, err := l.rdb.SetNX(ctx, l.key, l.token, l.ttl).Result()
    if err != nil {
        return err
    }
    if !ok {
        return ErrLockNotAcquired
    }
    return nil
}

// 尝试加锁（带重试）
func (l *RedisLock) AcquireWithRetry(ctx context.Context, retries int, interval time.Duration) error {
    for i := 0; i < retries; i++ {
        if err := l.Acquire(ctx); err == nil {
            return nil
        } else if !errors.Is(err, ErrLockNotAcquired) {
            return err // 网络等错误直接返回
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(interval):
        }
    }
    return ErrLockNotAcquired
}

// ★ 解锁：用 Lua 保证"比较 token + 删除"是原子的
func (l *RedisLock) Release(ctx context.Context) error {
    n, err := l.rdb.Eval(ctx, unlockScript, []string{l.key}, l.token).Int64()
    if err != nil {
        return err
    }
    if n == 0 {
        // 说明锁已经过期，或者被别人持有（我们丢失了锁）
        // ★ 必须让调用方知道这一点！否则业务会以为"我还在持锁"
        return ErrLockLost
    }
    return nil
}
```

#### 5.1.3 三个关键问题（**面试官必问，答不上等于没做过**）

**问题一：为什么释放锁要用 Lua 脚本？**

> **参考回答**：因为"**判断锁是不是我的**"和"**删除锁**"必须是原子的。如果分两步：`GET` 判断是我的 → 此时**锁刚好过期**，另一个客户端拿到了新锁 → 我执行 `DEL` → **删掉了别人的锁**！这会导致"锁形同虚设"，两个客户端同时持锁 → 业务并发执行（比如结算跑了两遍）。**Lua 脚本在 Redis 中是原子执行的**（单线程，脚本执行期间不会插入其他命令），所以"GET 比较 + DEL"能保证原子性。
>
> **注意**：严格说 Lua 脚本的原子性**只是"不被其他命令打断"**，它**没有回滚能力**（如果脚本中途报错，前面的操作不会回滚）。但对于"比较 + 删除"这么简单的逻辑，原子性就足够了。

**问题二：锁过期了但业务还没执行完，怎么办？**

这是**分布式锁最大的坑**。**三种解法**：

| 方案 | 说明 | 评价 |
|---|---|---|
| **① 评估并设置合理的 TTL** | 估算业务的最长执行时间，TTL 设置得比它长（比如业务最多 10 秒，TTL 设 30 秒） | **最简单，但不可靠**（业务可能因为 GC/网络抖动/数据量突增而超时） |
| **② Watch Dog（看门狗）自动续期** | 后台起一个 goroutine，每隔 `TTL/3` 检查"任务还在跑吗"，是就把 TTL 续上 | **Redisson 的做法**，效果最好，但要**自己实现**（Go 的 go-redis 没有内置） |
| **③ 让业务本身幂等（★ 最重要）** | **不依赖锁的正确性**——即使锁失效、两个进程同时执行，结果也是对的 | **治本方案**，也是我最推荐的 |

**Go 实现的 Watch Dog（带自动续期）**：

```go
type RedisLockWithWatchdog struct {
    *RedisLock
    stopCh   chan struct{}
    lostCh   chan struct{}   // 通知调用方"锁丢了"
    watchdog *time.Ticker
}

// 续期脚本：只有 token 还是我的，才续期
const renewScript = `
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('PEXPIRE', KEYS[1], ARGV[2])
else
    return 0
end
`

func (l *RedisLockWithWatchdog) AcquireWithWatchdog(ctx context.Context) error {
    if err := l.Acquire(ctx); err != nil {
        return err
    }
    l.stopCh = make(chan struct{})
    l.lostCh = make(chan struct{}, 1)
    l.watchdog = time.NewTicker(l.ttl / 3) // 每 1/3 TTL 续期一次

    go func() {
        for {
            select {
            case <-l.stopCh:
                return
            case <-l.watchdog.C:
                renewCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
                n, err := l.rdb.Eval(renewCtx, renewScript,
                    []string{l.key}, l.token, l.ttl.Milliseconds()).Int64()
                cancel()
                if err != nil || n == 0 {
                    // ★ 续期失败（锁被别人拿了 / Redis 挂了）
                    //   必须通知调用方！否则它以为还在持锁
                    select {
                    case l.lostCh <- struct{}{}:
                    default:
                    }
                    return
                }
            }
        }
    }()
    return nil
}

// ★ 业务代码要监听 Lost() 通道，一旦锁丢失就要停止工作
func (l *RedisLockWithWatchdog) Lost() <-chan struct{} { return l.lostCh }
```

**使用示例（结算任务）**：

```go
func RunSettlement(ctx context.Context) error {
    lk := NewRedisLockWithWatchdog(rdb, "lock:settlement:daily", 30*time.Second)
    if err := lk.AcquireWithWatchdog(ctx); err != nil {
        return fmt.Errorf("another settlement is running: %w", err)
    }
    defer lk.ReleaseWatchdog(ctx)

    // ★ 关键：监听锁丢失信号
    done := make(chan error, 1)
    go func() {
        done <- doSettlement(ctx)   // 业务逻辑
    }()

    select {
    case err := <-done:
        return err
    case <-lk.Lost():
        // 锁丢了，但业务已经在跑 —— 只能让它跑完（不能中途杀），
        // 但要【记录严重告警】，因为可能有两个实例在同时结算
        log.Errorf("CRITICAL: lock lost during settlement, possible concurrent execution")
        return <-done
    }
}
```

> **★ 面试话术（这一段能直接体现你的工程成熟度）**："**我认为最重要的不是把锁做得完美，而是让业务本身幂等**。比如 FlashRoot 的结算，我在**结算明细表上建了 `UNIQUE(biz_no, customer_id)`** —— 即使锁失效、两个实例同时跑结算，**重复的插入会被唯一索引拒绝**，而账户余额的更新用的是 `UPDATE account SET balance = balance + ? WHERE customer_id = ?` 的**原子累加**（不是"先读再算再写"），所以**并发执行也不会算错**。**锁在这里只是"性能优化"（避免无谓的重复计算），而不是"正确性保证"** —— 正确性由数据库约束保证。**这是我做分布式系统的一个基本原则：不要用分布式锁去保证正确性，因为锁本身就会失效。**"

**问题三：单实例 Redis 锁 + 主从切换会怎样？（RedLock 争议）**

```
时刻 1  客户端 A 在【主库】上拿到了锁 lock:order
时刻 2  主库还没把这条 SET 同步到从库，主库【挂了】
时刻 3  哨兵把【从库】提升为新主库（此时新的主库上【没有】这个锁）
时刻 4  客户端 B 来加锁 → 成功了！
        💀 现在 A 和 B 同时持有同一把锁
```

**RedLock（Redis 官方提出的方案）**：**向 N 个（比如 5 个）独立的 Redis 实例**（不是主从，是**完全独立的**）依次申请锁，**只有在多数派（N/2+1）上成功，并且总耗时小于锁的有效时间**，才算加锁成功。

**RedLock 的争议（面试官问"你怎么看 RedLock"，这是道**观点题**，答得有理有据即可）**：

**反对方（Martin Kleppmann，著名的分布式系统专家）**：
1. **依赖系统时钟**：RedLock 的正确性建立在"各个节点的时钟不会有大的漂移"上，但**时钟可能因为 NTP 校时或人为调整而跳变**，导致锁的实际有效期不可控；
2. **没有 fencing token（栅栏令牌）**：即使 RedLock 保证了"多数派上互斥"，**仍然无法阻止"锁过期后两个客户端同时操作"**。真正的解法是**给每次加锁分配一个单调递增的 token，资源端（数据库）拒绝比已见过的更小的 token 的操作**（fencing token）；
3. **GC/网络延迟的不可控性**：客户端可能在"拿锁后、操作前"发生长时间 GC，导致锁过期；
4. **结论**："**如果你需要锁来保证正确性，那 Redis 锁（包括 RedLock）不够用，应该用 ZooKeeper/etcd 这类基于共识的系统的锁；如果只是用来提升效率（防止重复劳动），单实例 Redis 锁就够了，没必要用 RedLock。**"

**支持方（antirez，Redis 作者）**：
- RedLock 的时钟依赖假设是合理的（时钟跳变是罕见且有界的事件）；
- fencing token 需要资源端配合改造，很多场景做不到；
- 实际工程中"锁过期"的概率很低，配合重试和幂等已经足够。

> **面试话术（推荐的中立立场，非常加分）**：
> "**我倾向于 Martin Kleppmann 的观点。** RedLock 的核心问题不是"它实现得不好"，而是"**它试图用一个不保证强一致的系统（Redis）去实现强一致语义的锁**"这一个根本矛盾。**我的实践原则是分层：**
> **① 如果锁是用来"防止重复劳动"（比如避免两个实例同时跑一个耗时的统计任务）** → **单实例 Redis 锁足够了**（即使偶尔失效，最坏情况是任务跑了两遍，没有正确性问题）；
> **② 如果锁是用来"保证正确性"（比如不能重复扣款）** → **不要依赖锁**，而是让**资源端幂等**：数据库唯一索引、状态机的原子流转（`UPDATE ... WHERE status = 'pending'`）、或者用 **fencing token**（每次加锁带一个单调递增的版本号，资源端只接受更大的版本号）。**这样无论锁是否失效，正确性都有保证。**"
>
> **"fencing token" 这个词能说出来，面试官基本会认定你是读过分布式系统资料的。**

#### 5.1.4 Go 生态的分布式锁选型

| 库 | 特点 |
|---|---|
| **`github.com/redis/go-redis/v9` + 自己封装** | **最推荐**——锁的逻辑很简单（20 行 Lua），自己写可控、可调试，避免引入重依赖 |
| `github.com/go-redsync/redsync/v4` | RedLock 的 Go 实现，封装完善 |
| **`github.com/bsm/redislock`** | 轻量的 Go 分布式锁，支持自动续期（Obtain + Refresh），**推荐度仅次于自己封装** |
| **etcd 的 `concurrency` 包** | 基于 Raft 的锁，**最可靠**（适合必须保证正确性的场景）；etcd 的 lease + watch 天然支持会话保活 |
| **ZooKeeper 的临时顺序节点** | 经典方案（Java 生态常用），Go 里用 `go-zookeeper` |

> **结合项目话术**：**"我一般用 go-redis 自己封装，因为锁逻辑只有二十行 Lua，自己写可控、好调试、不引入重依赖。只有在需要自动续期时才用 `bsm/redislock`。"**（**这个答法体现了"不过度依赖第三方库"的工程判断**）

### 5.2 限流

#### 5.2.1 四种限流算法的原理与对比

| 算法 | 原理 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|
| **固定窗口** | 按固定时间窗口计数（`INCR` + `EXPIRE`） | **实现最简单** | **临界问题**：窗口边界可能放过 2 倍流量 | 要求不高的场景 |
| **滑动窗口** | 用 ZSet 存请求时间戳，统计"最近 N 秒"的请求数 | **平滑，无临界问题** | **内存开销大**（每个请求一个 ZSet 成员） | 精确限流（但要用 ZSet 裁剪） |
| **令牌桶** | 恒定速率生成令牌，请求拿令牌 | **允许突发流量**（桶里有存量就能突增） | 需要维护令牌生成 | **最常用**（对突发友好） |
| **漏桶** | 请求进桶，以恒定速率流出 | **输出绝对平滑**（保护下游） | 无法应对突发 | 保护脆弱的下游服务 |

**固定窗口的临界问题（面试常考）**：

```
限流：每分钟最多 100 次

时间:     00:00 ─────────── 01:00 ─────────── 02:00
                                 ↑ 窗口边界
在 00:59 的最后 1 秒发了 100 次（打满第 1 个窗口）
在 01:00 的前 1 秒发了 100 次（打满第 2 个窗口）
→ 在 00:59 ~ 01:01 这 2 秒内，实际通过 200 次 = 2 倍限额 💀
```

#### 5.2.2 实现一：固定窗口（最简单，含 Go 代码）

```go
// 固定窗口：用 INCR + EXPIRE 实现
// ★ 注意：必须用 Pipeline 或者 Lua 保证 INCR 和 EXPIRE 的原子性，
//   否则 INCR 之后崩溃会导致 key 永不过期（计数永远累加）
func FixedWindowAllow(ctx context.Context, rdb *redis.Client, key string, limit int64, window time.Duration) (bool, error) {
    // 用 Lua 保证原子性
    const script = `
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
end
if current > tonumber(ARGV[2]) then
    return 0
end
return 1
`
    res, err := rdb.Eval(ctx, script, []string{key}, window.Milliseconds(), limit).Int64()
    if err != nil {
        return false, err
    }
    return res == 1, nil
}
```

#### 5.2.3 实现二：滑动窗口（ZSet + Lua）

```go
// 滑动窗口：用 ZSet 存请求时间戳，统计最近 window 内的请求数
// ZSet 的 score 用时间戳（毫秒），member 用 "时间戳-随机数"（保证唯一）
const slidingWindowScript = `
local key      = KEYS[1]
local now      = tonumber(ARGV[1])          -- 当前时间戳（毫秒）
local window   = tonumber(ARGV[2])          -- 窗口大小（毫秒）
local limit    = tonumber(ARGV[3])          -- 限流阈值
local member   = ARGV[4]                    -- 本次请求的唯一标识

-- ① 删除窗口之外的旧数据（滑动 = 窗口跟着时间移动）
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- ② 统计当前窗口内的请求数
local count = redis.call('ZCARD', key)

-- ③ 判断是否超限
if count < limit then
    redis.call('ZADD', key, now, member)
    redis.call('PEXPIRE', key, window)      -- 兜底过期，防止冷 key 堆积
    return 1
else
    return 0
end
`

func SlidingWindowAllow(ctx context.Context, rdb *redis.Client, key string, limit int64, window time.Duration) (bool, error) {
    now := time.Now().UnixMilli()
    // member 必须唯一，否则同一毫秒的多个请求会被 ZSet 去重
    member := fmt.Sprintf("%d-%d", now, rand.Int63())
    res, err := rdb.Eval(ctx, slidingWindowScript, []string{key},
        now, window.Milliseconds(), limit, member).Int64()
    if err != nil {
        // ★ 限流器故障时的策略选择：fail-open（放行）还是 fail-close（拒绝）
        //   保护下游 → fail-close；保护可用性 → fail-open
        log.Errorf("rate limiter failed: %v", err)
        return true, nil // 这里选择 fail-open
    }
    return res == 1, nil
}
```

**优缺点**：**精确、平滑，但每个请求都要在 ZSet 里存一个成员**（高 QPS 下内存开销很大）。**优化**：如果 QPS 很高，可以用"**分段计数**"（把 1 秒分成 10 个 100ms 的桶，用 10 个计数器近似滑动窗口）。

#### 5.2.4 实现三：令牌桶（**最推荐，允许突发**）

```go
// 令牌桶：桶里最多 capacity 个令牌，以 rate 个/秒 的速率补充
// 每个令牌代表一次请求的许可
const tokenBucketScript = `
local key       = KEYS[1]
local capacity  = tonumber(ARGV[1])   -- 桶容量
local rate      = tonumber(ARGV[2])   -- 每秒补充的令牌数
local now       = tonumber(ARGV[3])   -- 当前时间（毫秒）
local requested = tonumber(ARGV[4])   -- 本次请求需要的令牌数

-- 读取当前的令牌数和上次补充时间
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens      = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

-- 首次访问：桶是满的
if tokens == nil then
    tokens = capacity
    last_refill = now
end

-- 计算从上次补充到现在应该新增的令牌数
local delta = math.max(0, now - last_refill)
local refill = delta * rate / 1000
tokens = math.min(capacity, tokens + refill)   -- 不能超过容量

-- 判断令牌是否足够
local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- 写回状态
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('PEXPIRE', key, math.ceil(capacity / rate * 1000) * 2)  -- 兜底过期

return {allowed, math.floor(tokens)}
`

func TokenBucketAllow(ctx context.Context, rdb *redis.Client, key string, capacity, rate float64, requested int64) (bool, float64, error) {
    now := time.Now().UnixMilli()
    res, err := rdb.Eval(ctx, tokenBucketScript, []string{key},
        capacity, rate, now, requested).Slice()
    if err != nil {
        return true, 0, err // fail-open
    }
    allowed := res[0].(int64) == 1
    remaining := float64(res[1].(int64))
    return allowed, remaining, nil
}

// 使用示例：接口限流 + 响应限流头（RFC 建议）
func RateLimitMiddleware(rdb *redis.Client, capacity, rate float64) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := fmt.Sprintf("rl:%s:%s", c.ClientIP(), c.FullPath())
        allowed, remaining, _ := TokenBucketAllow(c.Request.Context(), rdb, key, capacity, rate, 1)
        c.Header("X-RateLimit-Limit", strconv.FormatFloat(capacity, 'f', 0, 64))
        c.Header("X-RateLimit-Remaining", strconv.FormatFloat(remaining, 'f', 0, 64))
        if !allowed {
            c.Header("Retry-After", "1")
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "code": 429, "msg": "请求过于频繁，请稍后再试",
            })
            return
        }
        c.Next()
    }
}
```

> **令牌桶 vs 漏桶的关键区别（面试常考）**：
> - **令牌桶**：桶里**攒着令牌**，突发流量来了可以**一次消耗多个令牌**（只要桶里有）→ **允许突发**；
> - **漏桶**：请求排队，**以恒定速率流出** → **输出平滑，不允许突发**。
>
> **怎么选**："**令牌桶适合"允许一定突发"的场景**（比如秒杀开始瞬间的流量洪峰，只要不超过桶容量就放行），**漏桶适合"保护脆弱的下游"**（比如下游的第三方接口有严格的 QPS 限制，超了会被封）。"

#### 5.2.5 限流器的部署位置（**体现全局视野**）

```
客户端
  ↓
① CDN / WAF 层限流（防 DDoS，按 IP）
  ↓
② 网关层限流（Nginx / APISIX / 自研网关，按路由/租户）
  ↓
③ 应用层限流（本节的 Redis 限流，按用户/接口）
  ↓
④ 服务间调用限流（熔断器、连接池限制、gRPC 拦截器）
  ↓
⑤ 数据库层限流（连接池大小、慢查询熔断）
```

> **面试话术**："限流应该**分层做**，而不是只在应用层做。**越靠前拦截成本越低**（在网关拦掉 90% 的恶意流量，应用就不用处理了）。**Redis 限流适合"需要跨实例精确计数"的场景**（比如"每个用户每分钟最多 10 次下单"）；而"保护服务自身不被压垮"更适合用**进程内的本地限流**（比如 Go 的 `golang.org/x/time/rate`，零网络开销，虽然多实例下不精确，但保护本进程足够了）。**注意 Redis 限流本身会给 Redis 带来压力**——每个请求都要访问 Redis，所以要多做一步：先用本地限流挡一层，再对通过的请求做精确的 Redis 限流。"

### 5.3 计数器与排行榜（**结合金币计费项目**）

#### 5.3.1 计数器：为什么必须用 Redis 的原子操作？

**核心问题**：`INCR`/`DECR` 是**原子**的，而"先 GET 再 SET"**不是**。

```go
// ❌ 危险：读-改-写，并发下会丢更新
val, _ := rdb.Get(ctx, "counter").Int64()
rdb.Set(ctx, "counter", val+1, 0)
// 两个 goroutine 同时读到 100，都写回 101 → 实际应该 102，丢了 1 次

// ✅ 正确：用原子的 INCR
rdb.Incr(ctx, "counter")
```

#### 5.3.2 金币计费：完整的扣减方案（**LoveSpouse 项目的核心话术**）

**场景**：用户和数字人对话，按消息/时长消耗金币。**要求**：① 不能超扣（余额不能为负）；② 不能重复扣（同一条消息只扣一次）；③ 高并发下不能出现竞态。

**方案一：用 Lua 脚本做"检查 + 扣减"的原子操作（推荐）**

```go
// 金币扣减的 Lua 脚本
// 返回值：>= 0 表示扣减后的余额；-1 表示余额不足；-2 表示重复请求
const deductCoinScript = `
local balanceKey = KEYS[1]          -- 余额 key：coin:balance:{userID}
local dedupKey   = KEYS[2]          -- 幂等 key：coin:dedup:{requestID}
local amount     = tonumber(ARGV[1])-- 扣减数量
local dedupTTL   = tonumber(ARGV[2])-- 幂等记录保留时间（秒）

-- ① 幂等检查：如果这个请求已经处理过，直接返回
if redis.call('EXISTS', dedupKey) == 1 then
    return -2
end

-- ② 读取余额
local balance = tonumber(redis.call('GET', balanceKey) or '-1')
if balance < 0 then
    return -3                       -- 余额 key 不存在（用户未初始化）
end

-- ③ 余额检查（★ 关键：不能扣成负数）
if balance < amount then
    return -1
end

-- ④ 原子扣减 + 记录幂等标记
redis.call('DECRBY', balanceKey, amount)
redis.call('SET', dedupKey, '1', 'EX', dedupTTL)

return balance - amount
`

func DeductCoin(ctx context.Context, rdb *redis.Client, userID, requestID string, amount int64) (int64, error) {
    balanceKey := fmt.Sprintf("coin:balance:%s", userID)
    dedupKey := fmt.Sprintf("coin:dedup:%s", requestID)

    // dedupTTL：同一条消息的重复请求窗口，设置为远大于重试间隔即可
    res, err := rdb.Eval(ctx, deductCoinScript,
        []string{balanceKey, dedupKey}, amount, 600).Int64()
    if err != nil {
        return 0, err
    }
    switch res {
    case -1:
        return 0, ErrInsufficientBalance
    case -2:
        // ★ 重复请求：幂等返回成功（不要报错！调用方可能只是在重试）
        //   这里可以再查一次余额返回给调用方
        cur, _ := rdb.Get(ctx, balanceKey).Int64()
        return cur, nil
    case -3:
        return 0, ErrBalanceNotInitialized
    default:
        return res, nil
    }
}
```

**方案二：用 Hash 存多个字段（余额 + 累计消费），一次原子操作**

```go
// 如果余额和累计消费要一起更新，可以用 Hash + Lua
// 好处：一次网络往返完成多个字段的原子更新
const deductCoinHashScript = `
local key    = KEYS[1]
local amount = tonumber(ARGV[1])

local balance = tonumber(redis.call('HGET', key, 'balance') or '-1')
if balance < 0 then return -3 end
if balance < amount then return -1 end

redis.call('HINCRBY', key, 'balance', -amount)
redis.call('HINCRBY', key, 'total_consumed', amount)
return balance - amount
`
```

**为什么不用 MySQL 直接扣？（面试一定会追问，这是关键的价值点）**

| 维度 | MySQL 方案 | Redis 方案 |
|---|---|---|
| **性能** | 单行热点更新，**串行化**（行锁），TPS 有限（几千到几万） | **10 万+ QPS**，内存操作 |
| **并发** | 高并发下大量行锁等待，**容易死锁**（如果涉及多行/多表） | 无锁（单线程天然原子） |
| **持久性** | **强**（事务 + redo log） | **弱**（可能丢 1 秒） |
| **正确性** | 强一致 | 需要幂等设计 + 对账兜底 |

**LoveSpouse 的实际架构（**面试话术**）**：

> **"数字人的金币计费我采用的是「Redis 做实时扣减（权威的实时余额）+ MySQL 做账本流水（异步落账）+ 定时对账」的三层架构。**
> **① 实时扣减走 Redis 的 Lua 脚本**：因为扣减是"检查余额 + 扣减 + 记录幂等"的三步操作，必须原子，而 Lua 脚本能保证这一点；同时 Redis 的单线程模型天然避免了行锁竞争和死锁，抗住了高并发。
> **② 流水异步落 MySQL**：每次扣减成功，发一条消息（或者写本地消息表）到 MySQL 记录流水。**流水表用了 `UNIQUE(request_id)` 做幂等**，重复消费不会产生重复流水。
> **③ 定时对账**：每天/每小时跑一次对账任务，用 MySQL 的流水汇总（`SUM`，DECIMAL 精度）和 Redis 的余额比对，不一致就告警并人工介入。
>
> **这个架构的关键决策是：我明确区分了"实时值"（Redis）和"账本"（MySQL）两个语义**。**在不能丢数据的场景下，直接让 Redis 做唯一存储是危险的，所以我用 MySQL 的流水作为"不可丢失的凭证"，Redis 的余额作为"可快速重建的实时状态"—— 即使 Redis 全丢了，也能从 MySQL 的流水重新算出余额。** "（**LoveSpouse 数字人 —— 这是整个 Redis 模块最有力的一段话术，一定要背熟**）

**追问：如果 Redis 的余额丢了怎么恢复？**

> "**用 MySQL 的流水重放**。流水表记录了每一次变更（`request_id`、`user_id`、`amount`、`created_at`），所以可以按用户汇总算出余额（`SUM(amount)`）。**这就是为什么流水表必须用 DECIMAL 精确记录、必须有幂等键、必须只增不改。** 恢复是一个离线任务，不需要实时。"
>
> **注意**："**这也意味着 Redis 不需要开很重的持久化**——因为余额是可重建的。我把 Redis 配成 AOF everysec 主要是为了减少恢复时的重放量，而不是为了让 Redis 成为唯一的数据源。**判断一个 Redis 数据该不该持久化，标准是"它能不能被重建"。**"

#### 5.3.3 排行榜（ZSet 的经典应用）

```go
// Redis 排行榜的常用操作
func LeaderboardDemo(ctx context.Context, rdb *redis.Client) {
    key := "rank:flashroot:rebate:202609"   // 按月的返点排行榜

    // ① 更新分数（ZINCRBY 是原子的，适合累加型榜单）
    rdb.ZIncrBy(ctx, key, 100.5, "customer:1001")
    rdb.ZIncrBy(ctx, key, 250.0, "customer:1002")

    // ② 批量更新（用 Pipeline 减少 RTT）
    pipe := rdb.Pipeline()
    for _, item := range items {
        pipe.ZIncrBy(ctx, key, item.Score, item.Member)
    }
    pipe.Exec(ctx)

    // ③ 取 Top 10（含分数）—— ★ 一定要限制范围，不要 ZRANGE 0 -1
    top10, _ := rdb.ZRevRangeWithScores(ctx, key, 0, 9).Result()
    for i, z := range top10 {
        fmt.Printf("%d. %s: %.2f\n", i+1, z.Member, z.Score)
    }

    // ④ 查某个成员的排名（从 0 开始，ZREVRANK 是从高到低）
    rank, _ := rdb.ZRevRank(ctx, key, "customer:1001").Result()
    fmt.Printf("customer:1001 排名: %d\n", rank+1)

    // ⑤ 查某个成员的分数
    score, _ := rdb.ZScore(ctx, key, "customer:1001").Result()

    // ⑥ 查分数区间内的成员（比如"返点超过 100 的渠道商"）
    rdb.ZRangeByScoreWithScores(ctx, key, &redis.ZRangeBy{
        Min: "100", Max: "+inf", Offset: 0, Count: 100,
    })

    // ⑦ 只保留 Top 1000（定期裁剪，控制内存）
    rdb.ZRemRangeByRank(ctx, key, 0, -1001)

    // ⑧ 设置过期时间（月度榜单，保留 2 个月）
    rdb.Expire(ctx, key, 60*24*time.Hour)
}
```

**排行榜的坑与技巧（面试加分）**：

| 问题 | 解法 |
|---|---|
| **分数相同的排序不稳定** | ZSet 在分数相同时**按 member 的字典序排列**。如果需要"分数相同时按时间先后"，可以把时间戳编码进 member（如 `customer:1001:1726000000`），或者在分数里加一个极小的时间因子（**不推荐**，会有精度问题） |
| **大 ZSet 的内存开销** | 一个百万元素的 ZSet 用跳表 + dict，**内存可能上百 MB**。控制手段：定期裁剪（`ZREMRANGEBYRANK`）、分片（按用户 ID 哈希拆成多个榜单）、只保留 Top N |
| **榜单要"实时"还是"定时"** | 高频更新的榜单（如实时打赏榜）用 `ZINCRBY` 直接更新；**低频但量大的榜单用定时任务批量更新**（避免每次业务操作都要写 Redis） |
| **需要多维度的榜单** | 一个维度一个 ZSet（如"日榜/周榜/月榜"三个 key），或者用不同的 key 前缀 |
| **分页取榜单** | `ZREVRANGE key start stop` 是 O(log(N)+M)，**深分页同样有性能问题**，但比 MySQL 好得多（M 是返回的数量） |

> **结合项目话术**：**"FlashRoot 的渠道商返点排行我用 ZSet 实现——按渠道商累加返点金额（`ZINCRBY`），运营后台取 Top 100 展示（`ZREVRANGE 0 99 WITHSCORES`），并且定期用 `ZREMRANGEBYRANK` 只保留前 1000 名来控制内存。** 这里有一个和业务结合的点：**返点金额用 ZSet 的 double 存是有精度风险的**，所以**ZSet 只用来做排名展示**，真正的金额还是以 DECIMAL 精度的 MySQL 为准——**排行榜是"展示层"，不能作为"数据源"**。"（**FlashRoot 返点结算 + 04-MySQL 的金额精度知识串联 —— 这个答法非常漂亮，体现跨模块的知识融合**）

#### 5.3.4 其他常用场景速查

| 场景 | Redis 方案 | 说明 |
|---|---|---|
| **签到 / 打卡** | **Bitmap**（`SETBIT sign:202609:{uid} {day} 1`） | 一个用户一个月只要 **4 字节**；`BITCOUNT` 统计签到天数；`BITOP` 可以算"连续签到"（与运算） |
| **UV 统计（独立访客）** | **HyperLogLog**（`PFADD`/`PFCOUNT`） | **标准误差 0.81%**，12KB 就能统计 2^64 个元素的基数；**不能取回元素，只能计数**；`PFMERGE` 可以合并多个 HLL |
| **共同好友 / 关注** | **Set**（`SINTER`） | 交并差都是 O(n) |
| **附近的人** | **GEO**（`GEOADD`/`GEOSEARCH`，底层是 ZSet + GeoHash） | 用 ZSet 的 score 存 GeoHash 编码，范围查询转成 score 范围查询 |
| **消息队列** | **Stream**（5.0+，`XADD`/`XREADGROUP`） | **支持消费者组、ACK、消息回溯**，比 List 更完善的队列语义；比专业的 MQ 简单，适合轻量场景 |
| **延迟任务** | **ZSet**（score 存执行时间戳），定时 `ZRANGEBYSCORE 0 now` 捞取 | 精度依赖扫描频率；**LightSpouse 的数字人超时未支付可以用这个**（但生产上更推荐 RabbitMQ 的延迟队列） |
| **简单的发布订阅** | **Pub/Sub** | **不持久化，消息可能丢**（订阅者掉线就收不到）；需要可靠性就用 Stream |

### 5.4 本节高频 Q&A

**Q1：分布式锁的 TTL 设置多长合适？**

> **参考回答**：**TTL 要大于"业务的最长执行时间"，但也不能设得太长**（否则锁失效后要等很久才能被别人拿到，影响可用性）。我的做法是：① **先评估**——在业务代码里打点统计 P99.9 的执行时间；② **TTL 设为 P99.9 的 2~3 倍**（比如 P99.9 是 5 秒，TTL 设 15 秒）；③ **加上 Watch Dog 续期**（每 TTL/3 续一次）作为兜底；④ **最重要的：让业务幂等**（这样即使锁失效也不会出错）。

**Q2：Redis 分布式锁和 ZooKeeper/etcd 分布式锁怎么选？**

> **参考回答**：
>
> | | Redis | ZooKeeper/etcd |
> |---|---|---|
> | 一致性模型 | 最终一致（主从异步） | **强一致（Zab/Raft 共识）** |
> | 性能 | **高**（内存、无共识开销） | 中（每次操作要过共识） |
> | 锁释放 | 依赖 TTL（可能提前释放） | **会话断开自动释放**（临时节点/lease），更可靠 |
> | 实现复杂度 | 低（20 行 Lua） | 中 |
> | 适用 | **防止重复劳动、提升效率** | **对正确性要求高、并发不极端的场景** |
>
> **我的选择**："**如果团队已经有 Redis，且锁只用于"防重复"（不是"保正确"），用 Redis 就够了**——性能好、运维简单。**如果必须保证正确性**（比如不能重复扣款），**我不会依赖任何分布式锁，而是用"资源端幂等"**（数据库唯一索引 + 原子状态流转），因为这才是真正可靠的方案。**ZooKeeper/etcd 的锁更可靠，但仍然无法解决"客户端 GC 导致锁过期"这个问题**，所以"幂等兜底"永远是必须的。"

**Q3：Redis 实现延迟队列怎么做？**

> **参考回答**：**用 ZSet，score 存"应该执行的时间戳"**：
> ```bash
> ZADD delay_queue 1758100000000 "task:1"     # 添加一个 2 秒后执行的任务
> ZRANGEBYSCORE delay_queue 0 <当前时间戳> LIMIT 0 10   # 捞取到期的任务
> ZREM delay_queue "task:1"                   # 删除（★ 要用 Lua 保证"捞取 + 删除"的原子性）
> ```
> **注意**：**必须是 Lua 脚本原子地"读 + 删"**，否则多实例部署时同一个任务会被多个实例捞到（虽然可以用唯一锁去重，但原子性更简单）。
> **更可靠的方案是 RabbitMQ 的延迟队列**：RabbitMQ 支持 **TTL + 死信交换机（DLX）** 实现延迟消息，或者用 **`rabbitmq_delayed_message_exchange` 插件**（更精准，不用 TTL 的"队头阻塞"问题）。
> **结合项目**："我们在 POS 数据贯通里用 RabbitMQ 的延迟队列处理**超时未支付的订单**，比 Redis ZSet 轮询更精准（到点触发，没有轮询间隔的误差），而且**消息本身有持久化和 ACK 机制，不会丢**。**Redis ZSet 适合"能容忍秒级误差、且丢失不致命"的场景。**"（**POS 数据贯通**）

**Q4：Redis 的 `INCR` 和数据库的自增哪个好？**

> **参考回答**：**取决于用途**。
> - **业务主键** → 数据库自增或雪花 ID（**要有持久性、要能追溯**）；
> - **统计数据**（PV/UV/点赞数）→ **Redis 的 `INCR`**（性能高、可容忍最终一致）；
> - **秒杀库存** → Redis 的 `DECR` 做预扣减 + 数据库做最终扣减（**Redis 挡流量，数据库保正确**）；
> - **需要在数据库里做唯一约束的编号**（如订单号）→ 用**号段模式**（Redis 一次 `INCRBY 1000` 拿一段，应用内部分配）—— **这样既有 Redis 的高性能，又减少了 Redis 的压力**（一次网络往返给 1000 个 ID）。这正是**美团 Leaf** 的方案。**"号段模式"这个词说出来很加分。**

**Q5：Redis 的原子性够用吗？`MULTI/EXEC` 和 Lua 到底什么时候用？**

> **参考回答**：**"原子性"要区分两个层面**：
> ① **单命令的原子性**（`INCR`、`SETNX`、`ZINCRBY`）：Redis 单线程执行命令，单命令天然原子，**大部分计数/累加场景够用**；
> ② **多命令的原子性**（"读-判断-写"）：`MULTI/EXEC` **不能根据中间结果做判断**（命令在 `EXEC` 时按顺序执行，但你无法用前一条的结果决定后一条），**所以"检查余额再扣减"必须用 Lua 脚本**。
>
> **结论**："**只要涉及"读-判断-写"的逻辑，就用 Lua**。不涉及判断的批量累加，用 Pipeline（省 RTT）就够了。"

### 5.5 本节「结合项目」话术提示

| 岗位相关场景 | 你的经验 | 话术 |
|---|---|---|
| **防止定时任务重复执行**（OA 的自动审批、日报） | **FlashRoot 结算 / POS 消费** | "多实例部署下的定时任务必须加锁。我用的是 **Redis 锁 + 数据库幂等**的双保险：锁避免重复劳动，而数据库的唯一索引保证即使锁失效也不会重复结算。"（**FlashRoot 结算**） |
| **接口限流保护系统** | **通用** | "Our 网关和服务层都有限流。服务层用的是**令牌桶 + Lua**（允许突发，适配真实流量形态），并且**先过滤掉明显的刷子请求**再进 Redis，避免限流器本身成为瓶颈。" |
| **金币/积分扣减** | **LoveSpouse 金币计费** | **见 5.3.2 的完整话术。** |
| **排行榜** | **FlashRoot 返点排行** | **见 5.3.3 的话术。** |

---

## 6. 精选 18 道高频面试题与参考回答

> 前面各节已经覆盖了知识点级别的问答，这一节是**综合题和场景题**，建议每题口述 60~90 秒。

---

**Q1：Redis 为什么这么快？（**开场必问**）**

> **参考回答**：见 1.7 的完整结构。**一定要按层次答**：① **纯内存操作**（最根本，比磁盘快 10 万倍）；② **命令执行单线程**（无锁、无上下文切换）；③ **IO 多路复用 + 事件驱动**（epoll，单线程处理上万连接）；④ **高效的数据结构**（SDS、跳表、listpack）；⑤ **RESP 协议简单**；⑥ **jemalloc 内存分配器 + 共享整数对象**。**最后补一句 6.0 的多线程 IO**："6.0 起把网络读写和协议解析多线程化了，但命令执行仍是单线程——**瓶颈从命令执行转移到了网络 IO**。"（**说出这一层，直接区分于背八股的人**）

---

**Q2：Redis 和 Memcached 的区别？**

> **参考回答**：
> | | Redis | Memcached |
> |---|---|---|
> | 数据结构 | **丰富**（String/List/Hash/Set/ZSet/Bitmap/HLL/Stream/GEO） | **只有 String（KV）** |
> | 持久化 | **支持**（RDB/AOF） | 不支持 |
> | 高可用 | **主从 + 哨兵 + Cluster** | 需要客户端分片，无官方集群方案 |
> | 线程模型 | 单线程（6.0 起多线程 IO） | **多线程**（能利用多核） |
> | 内存管理 | jemalloc（有内存碎片问题） | **Slab Allocation**（预分配，碎片少） |
> | 单机容量 | 受限于内存，但功能多 | **纯内存，简单高效** |
>
> **结论**："**Memcached 的优势是"简单 + 多线程 + 内存利用率高"，适合纯粹的 KV 缓存（比如缓存 HTML 片段、Session）；Redis 的优势是"数据结构丰富 + 功能全（持久化、集群、Lua、Pub/Sub）"。** 现在 Redis 基本是默认选择，Memcached 只在极端的"纯 KV + 要榨干多核"的场景还有优势。"

---

**Q3：Redis 的 key 和 value 的大小限制？**

> **参考回答**：**key 最大 512MB**（`redis-cli` 里的实际限制来自 SDS 的 `sdshdr64`），**value 最大也是 512MB**（String 类型）。**但这是理论上限，不是建议值**。
> **实践上的经验值**：**单个 value 不要超过 10KB**（String），**集合类元素的个数不要超过 5000**（这些就是"大 key"的经验阈值，见第 7 节）。**key 要短**（因为 key 要存在内存里，100 万个 100 字节的 key 就是 100MB），但**要可读**（便于排障）—— 平衡点是用**有意义的短前缀**（如 `u:1001:nick` 而不是 `user_profile_nickname_of_user_1001`）。

---

**Q4：Redis 的内存碎片是怎么回事？怎么处理？**

> **参考回答**：**内存碎片率 = `used_memory_rss / used_memory`**（操作系统分配的内存 / Redis 实际使用的内存）。**正常情况下应该在 1.0~1.5 之间**。
> **碎片产生的原因**：① **频繁的修改操作**（`APPEND`、`SETRANGE`、大量删除）导致内存分配器难以完全复用空闲块；② **数据增长/收缩**（大量删除后，释放的内存块不连续，无法满足新的大分配）；③ jemalloc 的"内存块档位"（8/16/32/48/64...）导致内部碎片。
> **处理**：
> - **`INFO memory` 看 `mem_fragmentation_ratio`**；**如果 > 1.5 且 `used_memory` 很大**，考虑整理；
> - **`MEMORY PURGE`**（4.0+，让 jemalloc 归还空闲内存给操作系统）；
> - **开启 activedefrag**（4.0+）：`activedefrag yes` + `active-defrag-threshold-lower 10`（碎片率超过 10% 就开始整理）+ `active-defrag-cycle-min/max`（控制 CPU 占用）—— **这是"在线整理碎片"，不用重启**；
> - **重启**（最彻底，但需要主从切换或接受停机）。
> **注意**："**整理碎片是有 CPU 成本的**，所以要用 `active-defrag-cycle-max` 限制它占用的 CPU 比例，避免影响正常请求。"

---

**Q5：Redis 的 `Pipeline` 有什么用？和批量命令（MSET）的区别？**

> **参考回答**：**Pipeline 的目的是减少网络往返（RTT）**。假设 RTT 是 1ms，执行 1000 条命令：不用 Pipeline 需要 1000ms，用了只需要 1ms + 服务端处理时间。
> **和 `MSET` 的区别**：
> - **`MSET`（批量命令）**：**服务端支持的多 key 操作**，一条命令完成，**在集群模式下要求所有 key 同槽**；
> - **`Pipeline`**：**纯客户端的优化**，把多条独立命令一次性发出去，**与服务端无关**，**在集群模式下可以针对不同节点分别 Pipeline**（但客户端库需要支持，go-redis 的 `ClusterClient.Pipeline()` 会自动按节点分组）。
>
> **注意**：**Pipeline 不是原子的**（中间可能插入其他客户端的命令）。**另外要控制 Pipeline 的批大小**（比如每批 100~1000 条），因为**服务端会把所有的回复都缓存在内存里**（`client-output-buffer-limit`），超大 Pipeline 可能导致**输出缓冲区溢出，连接被强制关闭**。

---

**Q6：Redis 的主从复制是同步还是异步？会有数据丢失吗？**

> **参考回答**：**异步**。主库执行完写命令就立即返回给客户端，**不等从库确认**。所以：
> - **主库宕机且未同步的数据会丢**（最多丢失"最后一次同步之后"的所有写入，量可能很大）；
> - **`WAIT numreplicas timeout`** 可以**阻塞等待 N 个从库确认**，能显著减少丢数据的窗口，但**不能完全消除**（主库可能在"确认之后、从库持久化之前"宕机）；
> - **`min-replicas-to-write`** 可以**在从库都联系不上时拒绝写入**（防脑裂）。
>
> **结论**："**Redis 的复制是"尽力而为"的最终一致，不是强一致。** 要强一致的场景必须用别的方案（如 MySQL 半同步/组复制、etcd 的 Raft）。"

---

**Q7：缓存和数据库的一致性怎么保证？（**超高频，完整答法见 4.4**）**

> **参考回答**：见 4.4.7 的决策框架。**核心三句话**：① **默认用 Cache Aside（先写库，再删缓存）+ TTL 兜底**；② **数据重要就上 Canal 订阅 binlog 保证"不漏"**；③ **绝对不能不一致的数据就不要缓存，或者用数据库的原子操作保证正确性。**

---

**Q8：什么是大 key？怎么发现和处理？（**见第 7 节**）**

> **参考回答**：见 7.1~7.3。

---

**Q9：Redis 的事务为什么不支持回滚？**

> **参考回答**：**这是设计选择，不是缺陷**。原因有三：
> ① **回滚需要 undo log**（记录每个操作的反向操作），这会**破坏 Redis "简单快速" 的设计理念**，增加内存和性能开销；
> ② **Redis 的错误分为两类**：**语法错误/命令不存在**在**入队时**就能检测到（`MULTI` 之后的命令会返回错误，此时整个事务会被标记为失败，`EXEC` 时全部不执行）；**运行期错误**（如对 String 执行 `LPUSH`）**只影响那一条命令**，其他命令继续执行。**Redis 认为"运行期错误是编程错误，应该在开发阶段发现，而不是靠回滚掩盖"**；
> ③ **Redis 的使用场景是缓存和高性能 KV**，大部分操作是单 key 的**天然原子操作**，需要事务回滚的场景很少。
>
> **如果真的需要"要么全成功要么全失败"，应该用 Lua 脚本**（把逻辑写在脚本里，用 `redis.pcall` 捕获错误，自己在脚本里实现"补偿"逻辑）。**注意**：Lua 脚本也**没有自动回滚**，只是提供了"能写判断逻辑"的能力。

---

**Q10：假如 Redis 内存满了，会发生什么？**

> **参考回答**：取决于 `maxmemory-policy`：
> - **`noeviction`（默认）**：**新的写命令直接返回错误**（`OOM command not allowed when used memory > 'maxmemory'`），**读命令仍然正常**。注意：**`DEL` 这类删除命令也是写命令**，所以理论上连删除都会被拒绝（实际 Redis 允许 `DEL` 执行，因为它能释放内存）；
> - **`allkeys-lru` / `volatile-lru` 等**：按策略**淘汰 key**，写命令正常执行；
> - **如果连淘汰都找不到可淘汰的 key**（比如 `volatile-*` 策略但所有 key 都没设 TTL）→ **退化成 `noeviction`**，写入报错。
>
> **生产建议**：① **一定要设置 `maxmemory`**（否则 Redis 会一直申请内存直到被 OOM Killer 杀掉或拖垮整台机器）；② **留出余量**（`maxmemory` 设为物理内存的 60% 左右，给 fork/COW 和内存碎片留空间）；③ **做好监控告警**（`used_memory` 接近 `maxmemory` 时告警）；④ **淘汰策略的选择**：**纯缓存用 `allkeys-lru`**，**混合存储（既有缓存又有关键数据）用 `volatile-lru`**（保证没设 TTL 的关键数据不会被淘汰）。

---

**Q11：`Redis` 的 `Hash` 适合存什么？和 `String` 存 JSON 比有什么优势？**

> **参考回答**：
> **Hash 的优势**：
> ① **部分更新/部分读取**（`HSET user:1 name tom` 只更新一个字段），而 String 存 JSON 必须**整体读-改-写**，在高并发下有**丢更新**的风险（两个请求同时读到旧的 JSON，各自改一个字段再写回，后写的会覆盖先写的）；
> ② **内存效率**（字段少时用 listpack 编码，比存 JSON 字符串更省）；
> ③ **支持原子操作**（`HINCRBY` 可以做字段级的原子累加）。
> **Hash 的劣势**：
> ① **不能对整个对象做过期**（Redis 的 TTL 只能作用于 key 整体）；
> ② **不支持嵌套结构**（Hash 的值只能是字符串，不能是另一个 Hash）；
> ③ key 的数量多时（每个对象一个 key），**key 本身的开销大**（每个 key 都要一个 dictEntry + redisObject + SDS）。
> **选择建议**："**需要部分更新或用 `HINCRBY` 做计数 → Hash；需要整体读写、结构复杂（有嵌套）→ String 存 JSON；需要 RediSearch 之类的检索能力 → JSON 模块（RedisJSON）**。"

---

**Q12：`ZSet` 的 `ZRANGEBYSCORE` 和 `ZRANGEBYLEX` 有什么区别？**

> **参考回答**：
> - **`ZRANGEBYSCORE`**：按 **score（分数）范围**查询，如"分数在 100~200 之间的成员"；
> - **`ZRANGEBYLEX`**：按 **member（成员）的字典序范围**查询，如"member 在 `[a` 到 `[c` 之间的成员"。
>
> **关键前提**：**`ZRANGEBYLEX` 只在"所有成员的 score 都相同"时才有意义**（因为 ZSet 是先按 score 排序，score 相同才按 member 字典序）。**典型用途**：**用 score 全设成 0 的 ZSet 实现"按字典序排序的自动补全/前缀匹配"**（比如 `ZRANGEBYLEX key [prefix (prefix\xff` 就能拿到所有以 prefix 开头的成员）。
>
> **语法细节**：`-` 表示最小，`+` 表示最大，`[` 表示包含，`(` 表示不包含。**举例**：`ZRANGEBYLEX key [a (c` 取所有 ≥ a 且 < c 的成员。

---

**Q13：Redis 6.0 的多线程是怎么回事？为什么不干脆做成完全多线程？**

> **参考回答**：见 1.7 的 ④。**核心要点**：**多线程化的只有"网络数据的读取 + 协议解析 + 回复的写回"，命令的执行仍然是单线程**。**为什么不做成完全多线程？** 因为一旦命令执行多线程化，就必须给所有数据结构加锁（细粒度锁），这会带来：① **复杂度的爆炸**（参考 Java 的 `ConcurrentHashMap`）；② **锁竞争的开销抵消多核收益**（尤其在小数据量、高并发的场景）；③ **丧失"单命令原子性"这个宝贵特性**（现在的 `INCR`、`SETNX` 之所以原子，就是因为单线程）。**所以 Redis 的选择是"只把真正瓶颈的部分（网络 IO）并行化，保留核心的简单性"。**

---

**Q14：Redis 的 `SCAN` 为什么可能返回重复的元素？**

> **参考回答**：因为 **`SCAN` 是"渐进式遍历"**，遍历期间如果发生了 **rehash**（扩容或缩容），**元素在桶之间的位置会重新分布**。假设：
> - 你遍历到了桶 5（此时表大小是 8，桶 5 里的元素都被返回了）；
> - 此时发生了**扩容**（表大小变成 16），原来桶 5 里的元素**一部分会被重新哈希到桶 5，一部分到桶 13**；
> - 你继续遍历到桶 13 → **又会返回一遍那些被重哈希到 13 的元素**。
>
> **所以 `SCAN` 的保证是"弱保证"**：① **遍历期间一直存在的 key，一定会被返回至少一次**；② **可能返回重复的 key**；③ **遍历期间被删除的 key 可能返回也可能不返回**。**应用层要自己去重**。
>
> **延伸**："这也是为什么 **SCAN 不能用于"精确的一次性快照"场景**（比如导出全量数据，要自己去重 + 处理并发修改）。"

---

**Q15：Redis 的 Rehash 为什么是渐进的？**

> **参考回答**：见 1.6。**核心**：**避免一次性 rehash 阻塞单线程**。要点包括：`ht[0]`/`ht[1]` 双表、`rehashidx` 进度指针、**查找先查 ht[0] 再查 ht[1]、新增直接写 ht[1]**、扩容条件（负载因子 ≥ 1，有子进程时 ≥ 5）、缩容条件（< 0.1）、以及**定时任务兜底推进**（`serverCron` → `incrementallyRehash`，1ms 时间片）。

---

**Q16：如何用 Redis 实现"同一用户 5 分钟内只能提交一次"？**

> **参考回答**：**最简单的方案是 `SET key value NX EX 300`**：

```go
func SubmitOnce(ctx context.Context, rdb *redis.Client, userID string) (bool, error) {
    key := fmt.Sprintf("submit:limit:%s", userID)
    // ★ 一条原子命令：不存在则设置，并带 300 秒过期
    ok, err := rdb.SetNX(ctx, key, time.Now().Unix(), 5*time.Minute).Result()
    if err != nil {
        // ★ 故障时 fail-open 还是 fail-close？防重复提交场景建议 fail-open
        //   （宁可放过一次重复，也不要因为 Redis 故障导致所有人都不能提交）
        log.Errorf("setnx failed: %v", err)
        return true, nil
    }
    return ok, nil
}
```

> **注意几个点**：
> ① **如果业务失败了要允许重试**（比如提交失败后要能立即再提交）→ 需要在失败时 `DEL` 掉这个 key（**但要注意：DEL 掉之后就可能被刷，所以更严谨的做法是"成功才保留，失败就删"**）；
> ② **`NX` + `EX` 必须一条命令**（不要 `SETNX` 后再 `EXPIRE`）；
> ③ **如果要做"5 分钟内最多 3 次"这种带计数的**，就要用 `INCR` + 首次设置 `EXPIRE` 的 Lua 脚本（见 5.2.2）；
> ④ **分布式环境下这是跨实例生效的**（比 `sync.Map` 做的本地限流强）。
>
> **结合项目**：**"LoveSpouse 的数字人消息处理、POS 数据贯通的消息消费都有类似的"防重复处理"需求。** 我用的是**更强的方式：用消息 ID 或者业务单号做幂等键，配合数据库的唯一索引** —— 因为 Redis 的 `SETNX` 一旦 Redis 重启或数据过期，防重就失效了，**而数据库的唯一索引是永久的、可靠的**。**Redis 的 `SETNX` 适合"短时间窗口的防刷"（性能优化），数据库唯一索引适合"永久性的幂等保证"（正确性保证）。**"（**LoveSpouse / POS 数据贯通 —— 这个对比非常关键，体现你分得清"性能手段"和"正确性手段"**）

---

**Q17：缓存的数据结构与 MySQL 表结构如何对应？ORM 怎么处理？**

> **参考回答**：**没有强制对应关系，但有几个实践原则**：
> ① **缓存的是"查询结果"而不是"表"**（比如"用户详情页"的数据可能来自 5 张表，缓存的是一个聚合对象）；
> ② **用 Go 的 struct 直接序列化**（JSON / MessagePack / Protobuf）。**JSON 可读性好但体积大、序列化慢；MessagePack/Protobuf 体积小、快，但不可读**。**缓存 value 大的时候（> 1KB）用 MessagePack/Protobuf 收益明显**；
> ③ **字段增删的兼容性**：**JSON 天然兼容**（新增字段不影响旧数据反序列化）；**Protobuf 需要预留字段号**（用 `reserved`）；
> ④ **不要缓存 GORM 的 model 结构**（因为 model 里可能有 `gorm.Model` 的嵌入字段、关联字段、`DeletedAt` 等，序列化出来很冗余）→ **定义一个专门的 DTO 用于缓存**；
> ⑤ **null 值的处理**：`time.Time` 的零值会序列化成 `"0001-01-01T00:00:00Z"`，比 `null` 大很多；用 `*time.Time` 或者自定义序列化。

```go
// 缓存 DTO（不要用 GORM model 直接序列化）
type UserCacheDTO struct {
    ID        int64   `json:"id"`
    Nickname  string  `json:"nickname"`
    Avatar    string  `json:"avatar"`
    Level     int     `json:"level"`
    UpdatedAt int64   `json:"updated_at"`  // ★ 用 int64 时间戳，比 RFC3339 字符串省一半空间
    // 敏感字段不入缓存（手机号、身份证）
}
```

---

**Q18：如果让你设计一个"客服系统的会话缓存"，你会怎么设计缓存结构？**

> **参考回答**（**这是岗位相关的场景题，能展示综合能力**）：我会分四类数据设计，**因为它们的访问模式和一致性要求完全不同**：
>
> | 数据 | 访问模式 | 存储结构 | 过期策略 | 一致性要求 |
> |---|---|---|---|---|
> | **坐席在线状态** | 写频繁（心跳）、读频繁 | **String + TTL**（`online:{agentID}` = 最后心跳时间，TTL 60 秒） | 心跳续期，不续就自动下线 | **最终一致**（状态可容忍秒级误差） |
> | **用户会话上下文**（最近 N 条消息） | 读多写多 | **List**（`LPUSH` + `LTRIM 0 99` 保留最近 100 条）或 **Stream** | TTL 1 小时 | 最终一致 |
> | **会话详情**（客服/用户信息、标签） | 读多写少 | **Hash**（部分字段更新） | TTL 30 分钟 + 主动失效 | Cache Aside |
> | **排队队列 / 待分配工单** | 写多读多、**必须精确** | **List**（`LPUSH`/`RPOPLPUSH`）或 **ZSet**（按优先级 + 等待时间排序） | 不过期 | **强一致**（不能丢单） |
>
> **几个关键设计点**：
> ① **坐席在线状态用"心跳 + TTL"而不是显式上下线**——因为客户端可能异常退出（断网、崩溃），显式的"下线"消息可能发不出来；用 TTL 过期天然处理了"失联"的情况（**这是"失效即下线"的经典设计**）；
> ② **排队队列不能用"缓存"语义**（不能设 TTL、不能因为内存满被淘汰）—— 所以要么用 Redis 但**单独一个实例 + `noeviction` 策略**，要么用专业的消息队列。**不能把"缓存"和"队列"混在一个实例里**，因为 `maxmemory-policy` 是全实例生效的，淘汰缓存 key 的同时可能会淘汰队列数据（**这是生产事故的经典来源，面试说出来非常有说服力**）；
> ③ **会话消息列表做长度限制**（`LTRIM`），否则一个长期会话会把 List 撑成大 key；
> ④ **消息的持久化在 MySQL**，Redis 只是"热数据"——**最近 100 条在 Redis，全量在 MySQL**，翻历史消息走 MySQL 游标分页（**和 04-MySQL 的深分页优化呼应**）。
>
> **这个答法的核心是"按数据的访问模式和一致性要求分层"，而不是"所有东西都往 Redis 塞"。**（**可结合 LoveSpouse 的对话流程 + 04-MySQL 的游标分页**）

---

## 7. 大 key 与热 key 专题

> **这是生产上最常出问题、也最能体现实战经验的部分。** 面试官如果问"你线上遇到过什么 Redis 问题"，**答大 key 或热 key 是最有说服力的**。

### 7.1 大 key（Big Key）

#### 定义（经验值）

| 类型 | 大 key 阈值 |
|---|---|
| **String** | value > **10KB** |
| **Hash / List / Set / ZSet** | 元素个数 > **5000** |
| **总体** | 单个 key 占用内存 > **1MB**（有些公司定义为 > 10MB） |

#### 危害（**面试要能说全**）

1. **阻塞主线程（最严重）**：`DEL` 一个百万成员的 Set 是 **O(n)** 的，**会阻塞 Redis 好几秒**，期间**所有请求都卡住**；
2. **网络拥塞**：`HGETALL` 一个大 Hash 会返回几百 MB 数据，**占满带宽 + 客户端内存**（客户端可能直接 OOM）；
3. **数据倾斜**：在 Cluster 模式下，一个大 key 会把**整个节点的内存和 QPS 拉高**（槽的分布再均匀也没用）；
4. **持久化开销**：RDB/AOF 重写时要一次性写入大 key，**加剧 fork 的 COW 开销**；
5. **过期删除阻塞**：大 key 过期时，**删除操作会阻塞主线程**（Redis 4.0 之后有 `lazyfree-lazy-expire` 可以异步删除）；
6. **主从同步延迟**：大 key 的传输会占用复制带宽，**导致从库延迟**。

> **注意一个反直觉的点**：**大 key 的危害不只是"慢"，更是"阻塞"** —— 因为 Redis 单线程，一个 O(n) 的删除操作会**让所有其他请求排队等待**。**这就是为什么"大 key 是单线程模型的死敌"。**

#### 排查方法（**面试要能说出具体命令**）

```bash
# ① redis-cli --bigkeys（★ 最常用，基于 SCAN，不阻塞）
redis-cli --bigkeys
# 输出示例：
# -------- summary -------
# Sampled 1000000 keys in the keyspace!
# Total key length in bytes is 12345678 (avg len 12.34)
#
# Biggest string found 'user:profile:10086' has 1024000 bytes
# Biggest list   found 'feed:1001' has 50000 items
# Biggest hash   found 'user:attributes:2000' has 20000 fields
#
# ★ 注意：它是【采样】的（每种类型只报告最大的那个），不是精确的 TOP N

# ② redis-cli --memkeys（按内存占用排序，更准确）
redis-cli --memkeys

# ③ MEMORY USAGE（★ 精确查看单个 key 的内存占用，4.0+）
127.0.0.1:6379> MEMORY USAGE user:profile:10086
(integer) 1024000
# 加上 SAMPLES 参数可以牺牲精度换速度（对于大集合，采样更快）
127.0.0.1:6379> MEMORY USAGE big:hash SAMPLES 0    # 0 = 精确计算

# ④ RDB 离线分析（最全面，不影响线上）
#    用 rdb-tools（Python）分析 dump.rdb，导出所有 key 的大小排行
pip install rdbtools python-lzf
rdb -c memory /var/lib/redis/dump.rdb --bytes 10240 -f memory.csv
#    然后按 size 排序找大 key

# ⑤ Redis 4.0+ 的慢日志会记录"删除大 key"的耗时
SLOWLOG GET 10
```

#### 处理方案

**① 删除大 key 要用异步删除（`UNLINK`）**

```bash
# ❌ DEL 是同步删除，大 key 会阻塞主线程
DEL big:hash

# ✅ UNLINK 是异步删除（4.0+）：先把 key 从 keyspace 摘除，真正的内存释放交给后台线程
UNLINK big:hash

# 对应的配置（让"过期删除"和"淘汰删除"也异步化）
lazyfree-lazy-expire yes          # 过期 key 的删除异步化
lazyfree-lazy-eviction yes        # 内存淘汰时的删除异步化
lazyfree-lazy-server-del yes      # 隐式删除（如 RENAME 覆盖）异步化
replica-lazy-flush yes            # 从库全量同步时清空数据异步化
```

**② 大 key 拆分（治本）**

```go
// 场景：一个大 Hash 存了 10 万个用户的属性
// ❌ 单 key：user:attributes（10 万 field）→ 大 key
// ✅ 拆分：按哈希分成 100 个子 key，每个 1000 个 field
func hashShardKey(userID string, shards int) string {
    h := fnv.New32a()
    h.Write([]byte(userID))
    shard := h.Sum32() % uint32(shards)
    return fmt.Sprintf("user:attributes:%d", shard)
}
// 优点：单个 key 变小，删除/读取都快
// 缺点：失去了"整体操作"的能力（比如 HLEN 要遍历所有分片求和）
//      → 解决方案：维护一个"总数字段"或者用单独的计数器
```

```go
// 场景：一个大 List 存了 100 万条消息
// ❌ 单 key：chat:1001:messages
// ✅ 拆分：
//    chat:1001:recent  (List, LTRIM 保留最近 100 条)  ← 高频读取的"热区"
//    MySQL 存全量，翻历史走游标分页               ← 冷数据
// 这是"冷热分离"的思想：Redis 只放热数据

// 场景：一个大 ZSet 存百万成员的排行榜
// ✅ 拆分：按分数区间拆（top 榜 + 长尾榜），或者定期 ZREMRANGEBYRANK 裁剪
rdb.ZRemRangeByRank(ctx, key, 0, -1001)   // 只保留前 1000 名
```

**③ 读取大 key 的替代方式（不要一次全取）**

```bash
# ❌ HGETALL 一个大 Hash → 几百 MB 的响应
HGETALL big:hash

# ✅ 用 HSCAN 分批取
HSCAN big:hash 0 COUNT 100

# ❌ SMEMBERS 一个大 Set
SMEMBERS big:set

# ✅ SSCAN 分批
SSCAN big:set 0 COUNT 100

# ✅ 只取需要的字段
HMGET big:hash field1 field2 field3
```

```go
// Go 中遍历大 Hash 的正确姿势（HSCAN 分批）
func IterateBigHash(ctx context.Context, rdb *redis.Client, key string, batch int64) error {
    var cursor uint64
    for {
        keys, nextCursor, err := rdb.HScan(ctx, key, cursor, "", batch).Result()
        if err != nil {
            return err
        }
        // HScan 返回的是 [field1, value1, field2, value2, ...]
        for i := 0; i < len(keys); i += 2 {
            field, value := keys[i], keys[i+1]
            _ = field
            _ = value
        }
        cursor = nextCursor
        if cursor == 0 {
            return nil
        }
    }
}
```

> **面试话术**："大 key 的处理我分三步：**① 发现**（`--bigkeys` 采样扫描 + `MEMORY USAGE` 精确定位 + RDB 离线分析做全量排行）；**② 止血**（用 `UNLINK` 异步删除，开启 `lazyfree-*` 系列配置，避免删除操作阻塞主线程）；**③ 治本**（拆分大 key：按哈希分片拆 Hash、冷热分离拆 List、定期裁剪 ZSet）。**另外最重要的是预防** —— 在**写入侧就限制集合的大小**（比如 List 用 `LTRIM` 限制长度），而不是等到变成了大 key 再处理。"

### 7.2 热 key（Hot Key）

#### 定义

**某个 key 被极端高频地访问**（比如 QPS 是其他 key 的几百倍）。比如"首页的全局配置"、"秒杀商品"、"热门角色的配置"。

#### 危害

1. **单节点 CPU 打满**：Cluster 模式下，热 key 落在某个槽上，**那个节点的 CPU 被打满**，而其他节点很闲（**分片解决不了热 key 的问题**！）；
2. **网络带宽打满**：这个节点成为带宽瓶颈；
3. **缓存击穿风险放大**：热 key 一旦失效，**瞬间的并发全部打到 DB**（见 4.2）。

> **这是面试官很爱问的点**："**Redis 集群能解决热 key 问题吗？**" → **不能！集群解决的是"数据量大"和"整体 QPS 高"的问题，但热 key 的 QPS 集中在单个 key 上，而单个 key 只能在一个槽（一个节点）上，分片对它无能为力。** 这个回答能直接区分"背过集群原理"和"真的想过这个问题"。

#### 排查

```bash
# ① redis-cli --hotkeys（★ 需要 maxmemory-policy 是 LFU 系列）
redis-cli --hotkeys
# 输出示例：
# [00.00%] Hot key 'hot:product:1' found so far with counter 9999
# 原理：利用 LFU 的访问计数器（redisObject 的 24 位 lru 字段在 LFU 模式下存的是计数器）

# ② monitor 命令（★ 慎用！会严重影响性能，只适合短时间调试）
redis-cli MONITOR | head -10000 | awk '{print $4}' | sort | uniq -c | sort -rn | head

# ③ 客户端埋点统计（★ 最推荐的方案，生产可用）
#    在业务代码里统计每个 key 的访问次数，定期上报到监控系统
#    优点：不影响 Redis，可以关联到具体的业务场景

# ④ 代理层统计（如果有 Twemproxy/Codis/自研代理）
# ⑤ 网络抓包分析（tcpdump + 分析）
```

**Go 侧的客户端埋点（推荐方案）**：

```go
// 用一个 Hook 统计 key 的访问频率（go-redis 支持 Hook 机制）
type HotKeyHook struct {
    mu     sync.Mutex
    counts map[string]int64
}

func (h *HotKeyHook) DialHook(next redis.DialHook) redis.DialHook { return next }

func (h *HotKeyHook) ProcessHook(next redis.ProcessHook) redis.ProcessHook {
    return func(ctx context.Context, cmd redis.Cmder) error {
        // 记录读命令的第一个参数（近似当作 key）
        if len(cmd.Args()) > 1 {
            if k, ok := cmd.Args()[1].(string); ok {
                h.mu.Lock()
                h.counts[k]++
                h.mu.Unlock()
            }
        }
        return next(ctx, cmd)
    }
}

func (h *HotKeyHook) ProcessPipelineHook(next redis.ProcessPipelineHook) redis.ProcessPipelineHook {
    return next
}

// 定期把统计结果上报（比如每分钟 Top 100），注意要重置计数器
func (h *HotKeyHook) Report() []KeyCount {
    h.mu.Lock()
    defer h.mu.Unlock()
    // 转成 slice 排序，取 Top N，然后清空
    ...
}
```

#### 解决方案（**四个层次**）

**① 本地缓存（最有效）**

```go
// 把热 key 缓存在进程内存里，请求直接命中本地缓存，完全不访问 Redis
// ★ 这是解决热 key 最有效的手段，因为它把"一次网络往返"变成了"一次内存访问"
var hotLocalCache = bigcache.NewBigCache(...)

func GetHotData(ctx context.Context, rdb *redis.Client, key string) ([]byte, error) {
    // ① 先查本地缓存
    if data, err := hotLocalCache.Get(key); err == nil {
        return data, nil
    }
    // ② 本地未命中，查 Redis
    data, err := rdb.Get(ctx, key).Bytes()
    if err != nil {
        return nil, err
    }
    // ③ 写回本地缓存（TTL 要短，比如 5~10 秒，避免多实例间不一致太久）
    hotLocalCache.Set(key, data)
    return data, nil
}
```

**注意**：**本地缓存的 TTL 要短**（5~10 秒），并且**多实例之间会有短暂不一致**。**对于"能容忍几秒不一致"的热数据（配置、榜单、商品详情），这是完美的方案。**

**② key 拆分（读写分散）**

```go
// 把热 key 复制成 N 份，随机读取，把压力分散到 N 个节点
const hotKeyCopies = 10

// 写：要写所有副本（或者用异步同步）
func setHotKey(ctx context.Context, rdb *redis.Client, baseKey, value string) error {
    pipe := rdb.Pipeline()
    for i := 0; i < hotKeyCopies; i++ {
        pipe.Set(ctx, fmt.Sprintf("%s:%d", baseKey, i), value, 0)
    }
    _, err := pipe.Exec(ctx)
    return err
}

// 读：随机选一个副本
func getHotKey(ctx context.Context, rdb *redis.Client, baseKey string) (string, error) {
    idx := rand.Intn(hotKeyCopies)
    return rdb.Get(ctx, fmt.Sprintf("%s:%d", baseKey, idx)).Result()
}
```

> **注意**：**这个方案只适用于"读多写极少"的数据**（比如配置、静态资源）。如果数据频繁更新，维护 N 个副本的一致性会非常麻烦。**另外要注意：如果用了分片，副本要落在不同的节点上**（用 hash tag 反而会把它们挤到一个槽，适得其反 —— **这是很细节的坑，面试说出来很加分**）。

**③ 读写分离（用从库分担读压力）**

热 key 的读请求打到从库，**用多个从库分摊读压力**。**注意**：这只能分担"读"，写还是打到主库；而且**从库也要防大 key**。

**④ 限流 + 熔断（兜底）**

如果热 key 的访问量超过了系统承载能力，**必须在入口限流**（见 5.2），否则整个系统会被拖垮。

### 7.3 大 key / 热 key 速查表

| | 大 key | 热 key |
|---|---|---|
| **问题本质** | **单 key 体积大**（O(n) 操作阻塞） | **单 key 访问量极高**（单节点 CPU/带宽打满） |
| **影响** | 阻塞主线程、网络拥塞、数据倾斜、持久化慢 | 单节点过载、集群的分片对它无效 |
| **排查** | `--bigkeys`、`MEMORY USAGE`、`rdb -c memory`、`SLOWLOG` | `--hotkeys`（LFU）、客户端埋点、代理层统计 |
| **解决** | `UNLINK` 异步删、拆分、冷热分离、定期裁剪、写入侧限制 | **本地缓存**、key 拆分多副本、读写分离、限流 |
| **预防** | 设计时就限制集合大小（`LTRIM`、`ZREMRANGEBYRANK`） | 设计时就识别热点（配置类、榜单类数据） |
| **一句话** | **"别让一个 key 太大"** | **"别让一个 key 太热"** |

### 7.4 本节「结合项目」话术提示

> **面试官问："你线上遇到过 Redis 的什么问题？"** → **这是一个绝佳的展示机会**，用下面的故事：

**S（背景）**：LoveSpouse 的数字人上线后，某个热门角色的对话入口在晚上 8~10 点的高峰期，Redis 的**单节点 CPU 会飙到 80% 以上**，而其他节点很闲。

**T（任务）**：定位原因并解决，同时保证高峰期服务不降级。

**A（行动）**：
1. **定位**：用客户端 Hook 埋点统计 key 的访问频次，发现**这个角色的配置 key 的 QPS 是其他 key 的 200 倍**——典型的热 key（因为角色被推荐到了首页，所有用户进入时都要读它的配置）；
2. **分析**：热 key 的问题用集群分片解决不了（单个 key 只能落在一个槽上）；
3. **解决**：① **加本地缓存**（`bigcache`，TTL 5 秒）—— 这一层就挡掉了 95%+ 的请求；② **配置变更时通过 Redis Pub/Sub 广播失效**（保证多实例的本地缓存能在秒级内同步）；③ **本地缓存作为降级手段**（Redis 抖动时至少还能读本地）；
4. **预防**：把"识别热点数据"作为**缓存设计的一个固定步骤**——配置类、榜单类、首页展示类的数据，默认就要考虑用本地缓存。

**R（结果）**：Redis 单节点 CPU 从 80% 降到 20% 以下，高峰期 P99 从 120ms 降到 15ms。

> **（可结合 LoveSpouse 数字人 / FlashRoot 排行榜作答）**
>
> **注意**：讲这个故事时，**要突出"我怎么发现问题的"（埋点统计）、"为什么集群解决不了"（原理）、"为什么选本地缓存"（权衡）**，而不是只讲"我加了缓存就好了"。**过程比结论重要。**

---

## 8. 速记表与自测清单

### 8.1 数字与配置速记

| 数字 / 配置 | 含义 |
|---|---|
| **44 字节** | `embstr` 与 `raw` 的分界（64 字节内存块 - 20 字节结构开销） |
| **16 字节** | `redisObject` 的大小 |
| **128 / 64 字节** | listpack/ziplist 编码的默认阈值（元素个数 / 单个元素大小） |
| **512** | `set-max-intset-entries` 默认值 |
| **32 / 0.25** | 跳表的最大层数 `ZSKIPLIST_MAXLEVEL` / 晋升概率 `ZSKIPLIST_P` |
| **512MB** | key 和 value 的最大长度 |
| **16384（2¹⁴）** | Cluster 的哈希槽数量（2KB 位图） |
| **+10000** | 集群总线端口 = 服务端口 + 10000 |
| **15000 ms** | `cluster-node-timeout` 默认值 |
| **30000 ms** | 哨兵的 `down-after-milliseconds` 默认值 |
| **1MB** | `repl-backlog-size` 默认值（**生产要调大**） |
| **10 次/秒** | 定期删除的扫描频率（`hz 10`） |
| **25ms** | 定期删除的单次时间上限 |
| **20 / 25%** | 定期删除每次抽样的 key 数 / 触发继续扫描的过期比例 |
| **everysec** | AOF 推荐的 fsync 策略（最多丢 1 秒） |
| **50%~60%** | Redis 内存占物理内存的建议上限（给 fork/COW 留空间） |
| **10KB / 5000** | 大 key 的经验阈值（String 的 value / 集合的元素数） |
| **0.81%** | HyperLogLog 的标准误差 |

### 8.2 一句话结论速记（面试"抢答"用）

- **为什么快**：纯内存 + 单线程（无锁）+ IO 多路复用 + 高效数据结构 + RESP。
- **6.0 多线程**：只多线程化了网络 IO，**命令执行仍单线程**。
- **SDS**：O(1) 取长度、二进制安全、预分配（<1MB 翻倍，≥1MB 加 1MB）。
- **empstr 44 字节**：64（jemalloc 块）- 16（redisObject）- 3（sdshdr8）- 1（'\0'）。
- **listpack vs ziplist**：listpack 存自己的长度，**消除了级联更新**。
- **ZSet = 跳表 + dict**：dict 做 O(1) 查分，跳表做 O(log n) 范围查询和排名。
- **跳表 vs 红黑树**：跳表范围查询友好、实现简单；**B+ 树是为磁盘设计的，内存场景不需要**。
- **渐进式 rehash**：`ht[0]`/`ht[1]` + `rehashidx`；查找查两个表，**新增只写 ht[1]**。
- **RDB vs AOF**：RDB 是快照（快、可能丢），AOF 是命令日志（全、慢）。**AOF 优先级更高**。
- **AOF 重写**：读内存生成最少命令集；**双缓冲区**（aof_buf + aof_rewrite_buf）。
- **混合持久化**：RDB 前导 + AOF 增量，**4.0 起默认**。
- **repl_backlog**：决定"能容忍多长断线"；默认 1MB **太小**。
- **SDOWN vs ODOWN**：主观下线（单个哨兵）/ 客观下线（quorum 个哨兵）。
- **quorum vs majority**：判定下线用 quorum，**选 leader 要多数派**。
- **16384**：Cluster 槽数（心跳包 2KB）；**MOVED 永久、ASK 临时且要先发 ASKING**。
- **缓存穿透**：查不存在的数据 → 布隆过滤器 / 空值缓存。
- **缓存击穿**：热点 key 失效瞬间 → 互斥锁 / 逻辑过期 / `singleflight`。
- **缓存雪崩**：大批失效或 Redis 挂 → TTL 抖动 / 多级缓存 / 熔断降级。
- **一致性**：Cache Aside（先写库再删缓存）+ TTL 兜底；**别用缓存保证正确性**。
- **分布式锁**：`SET NX PX` 加锁，**Lua 比较 token 再删**；**正确性靠幂等，不靠锁**。
- **RedLock 争议**：时钟依赖 + 无 fencing token → **正确性场景不要用 Redis 锁**。
- **大 key**：`UNLINK` 异步删 + 拆分；**热 key**：本地缓存 + 副本；**集群解决不了热 key**。

### 8.3 面试前自测（对着镜子口述，每题 60~90 秒）

1. 按 6 个层次讲"Redis 为什么快"，并解释"单线程"和"6.0 多线程"为什么不矛盾。
2. 画出一个 ZSet 的跳表结构（含 span 和 backward 指针），解释为什么不用红黑树。
3. 讲清 `embstr` 为什么以 44 字节为界（现场算一遍）。
4. 讲清渐进式 rehash 期间"查找/新增/删除"分别怎么处理。
5. 讲清 AOF 重写的完整流程，重点解释"双缓冲区"为什么必要。
6. 讲清 RDB 的 fork 与 COW，以及"为什么内存不能超过物理内存的一半"。
7. 讲清哨兵的"主观下线 → 客观下线 → 选 leader → 故障转移"完整流程，以及 quorum 和 majority 的区别。
8. 讲清 Cluster 的 16384 和 MOVED/ASK 的区别（**包括 ASKING 命令的作用**）。
9. 讲清缓存穿透/击穿/雪崩的区别和各自方案（**能写出 Go 代码**）。
10. 讲清缓存一致性的决策框架（**"默认 Cache Aside + TTL 兜底；重要就上 Canal；不能不一致就别缓存"**）。
11. 写出一个正确的 Redis 分布式锁（加锁、Lua 解锁、Watch Dog 续期），并说明"锁过期了怎么办"。
12. 讲清"令牌桶和漏桶的区别"，并说明限流该放在架构的哪一层。
13. 讲清 LoveSpouse 金币计费的完整架构（**Redis 实时扣减 + MySQL 账本 + 对账**），并回答"Redis 的余额丢了怎么恢复"。
14. 讲清"大 key 的危害、排查和处理"，以及"**为什么集群解决不了热 key**"。
15. 讲清"Redis 事务为什么不支持回滚"，以及"什么时候该用 Lua 而不是 MULTI/EXEC"。

### 8.4 反问环节（问面试官的问题）

- "咱们 Redis 的部署形态是什么（单机/主从+哨兵/Cluster）？有没有遇到过热 key 或者大 key 的问题，是怎么治理的？"
- "客服/OA 系统里 Redis 主要承担哪些职责（纯缓存、会话、分布式锁、队列）？有没有明确的'哪些数据不许放 Redis'的规范？"
- "团队有没有 Redis 的监控告警体系（大 key 扫描、慢日志、内存增长趋势）？"
- "在缓存和数据库的一致性上，团队有没有统一的原则或者工具（比如 Canal 这类 binlog 订阅）？"

> **反问注意事项**：**不要问"Redis 怎么用"这种基础问题**。上面这几个问题的共同点是**暗示你知道这些坑的存在**（大 key、热 key、一致性、监控），**面试官会觉得你是有实战经验、能立刻上手干活的人。**

---

*（本模块完，配套模块：[03-计算机网络](./03-计算机网络.md) | [04-MySQL](./04-MySQL.md) | [06-系统架构与高并发](./06-系统架构与高并发.md) | [07-OA与客服系统业务深度](./07-OA与客服系统业务深度.md)）*
