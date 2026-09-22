# 09 · MySQL 与 Redis 高频问答

> **本文件用法**：半天准备，别通读——先用 20 分钟把每个知识点下面的 `>` 引用块（**能直接说出口的一句话答案**）和表格扫一遍背下来，展开要点只当追问时的素材；标 **【理解层】** 的小节是简历里没有实战的，只答原理、**不要硬编项目经历**。「话术」是答题口径，里面的数字和事故请换成你真实经历，没有的就改成设计口吻（"我是这么设计的"），别改成回忆（"当时出了事故"）。

> **岗位锚点**：JD 第 3 条「熟练使用 MySQL 数据库，熟悉 Redis 等缓存/NoSQL 组件的使用场景」。面试官真正想确认的是——**LoveSpouse 为什么消息存 MySQL、金币余额放 Redis？POS 上亿流水怎么扛？** 每道八股都要能落到一个真实项目上。MySQL 占 60%，Redis 占 40%。

---

# 第一部分 · MySQL（60%）

## 1. 索引

### 1.1 B+ 树为什么适合做索引

> **一句话**：B+ 树是「矮胖 + 有序 + 叶子成链」——非叶子节点只存键不存数据，一页 16KB 能装上千个键，**3 层就能存约 2000 万行，查任意一行最多 3 次磁盘 IO**。

展开要点：
- 出发点是磁盘 IO：内存随机访问 ~100ns，机械盘随机读 ~10ms，差 10 万倍；InnoDB 以 **16KB 页**为最小 IO 单位，所以索引设计的唯一目标是「一次 IO 排除尽可能多的数据」。
- **vs B 树**：B 树非叶子节点也存数据行，一页装的键少 → 扇出小 → 树更高 → IO 更多；B+ 树数据全在叶子，非叶子是纯索引。
- **vs 哈希**：等值 O(1) 最快，但**不支持范围查询、排序、最左前缀**，还有哈希冲突；用在 Memory 引擎和 InnoDB 自适应哈希索引（AHI）。
- **vs 跳表**：跳表是 Redis ZSet 的选择（内存结构、实现简单、范围查询友好），但它是链表、随机层高、**磁盘随机 IO 太差**；B+ 树一个节点就是一页，天然对齐磁盘。
- 叶子节点用**双向链表**串联 → 范围查询、`order by`、分页都是顺序扫描。
- 扇出怎么算（加分点）：bigint 主键 8B + 页指针 6B ≈ 14B，一页 16KB 约 1170 个键；3 层 ≈ 1170 × 1170 × 16 ≈ **2000 万行**。

常见追问：为什么页是 16KB？→ 折中，太小 IO 次数多、太大一次读入大量无用数据。随机主键（UUID）会频繁页分裂、页利用率低，自增主键几乎不分裂。

项目话术：「POS 流水表我坚持用自增 bigint 主键而不是 UUID，就是因为 UUID 随机插入会让 B+ 树频繁页分裂；对外暴露的全局单号我用雪花 ID 另存一列，主键仍然自增。」

### 1.2 聚簇索引 / 二级索引 / 回表 / 覆盖索引

> **一句话**：InnoDB 主键索引就是聚簇索引，叶子直接存整行；二级索引叶子只存「索引列 + 主键」，所以查非索引列要**回表**再走一次聚簇索引——**覆盖索引就是让这次回表不发生**。

展开要点：
- 一张表只有一个聚簇索引。没主键 → 用第一个非空唯一索引；还没有 → 隐式生成 6 字节 `row_id`。
- 回表 = 两次 B+ 树查找：二级索引定位主键 → 拿主键去聚簇索引取整行。
- 覆盖索引：查询的列全在索引中，`EXPLAIN` 的 `Extra` 显示 `Using index`；因为二级索引自带主键，`select id, name from user where name=?` 建 `idx_name(name)` 就已经是覆盖索引。
- 主键要短：二级索引叶子要存主键，主键越长所有二级索引越大（UUID 36B vs bigint 8B）。
- **索引下推 ICP**（5.6+）：把 where 中能用索引判定的条件下推到引擎层过滤，减少回表次数，`Extra` 显示 `Using index condition`。

项目话术：「LoveSpouse 的角色列表接口我做成覆盖索引 `idx_status_sort(status, sort, id)`，列表页只查 id 和排序字段，`Extra` 从 `Using filesort` 变成 `Using index`，省掉回表那次随机 IO。」

### 1.3 最左前缀、索引失效、联合索引顺序

> **一句话**：联合索引 `(a,b,c)` 在 B+ 树里按 a→b→c 排序，所以查询必须从 a 开始连续匹配；**索引失效的本质是「条件让 B+ 树没法利用有序性去定位」**。

索引失效清单（背这张表就够）：

| 场景 | 例子 | 说明 |
| --- | --- | --- |
| 跳过最左列 | `where b=1`（索引 `(a,b)`） | 完全用不上 |
| 范围列后面的列 | `where a=1 and b>2 and c=3` | 只能用 a、b |
| 对索引列做函数/运算 | `where date(created_at)='2026-01-01'`、`id+1=5` | 改成 `>= .. and < ..` |
| 隐式类型转换 | `where phone=13800000000`（phone 是 varchar） | 等价于 `CAST(phone AS int)` |
| 前导模糊 | `like '%abc'` | 后模糊 `like 'abc%'` 可以走 |
| `or` 连接非索引列 | `where a=1 or d=2` | 退化为全表 |
| `!=` / `not in` / `is not null` | — | 不一定失效，看优化器成本判断 |
| 区分度太低 | 性别、状态只有几个值 | 优化器主动放弃索引 |

展开要点：
- 联合索引顺序口诀：**等值在前、范围在后、排序列跟上、区分度高的优先**。
- `order by` 也走最左前缀：`(a,b)` 下 `order by a,b` 免排序，`order by b` 会 `Using filesort`。
- 8.0 的**索引跳跃扫描（Skip Scan）**：a 区分度很低时，即使不写 a 也能用上索引——加分项。

项目话术：「POS 流水的主要查询是『某门店 + 某时间段』，我建的是 `(shop_id, biz_date, id)`：等值的 shop_id 在最左、范围的 biz_date 其次，最后带上 id 让排序分页也能走索引，避免 filesort。」

## 2. 执行计划与慢 SQL 治理

> **一句话**：`EXPLAIN` 我只看四个字段——`type` 看访问方式、`key` 看实际用了哪个索引、`rows` 看预估扫多少行、`Extra` 看有没有 filesort / temporary / 回表。

`type` 从好到坏：`system`/`const`（主键或唯一索引等值，最多一行）> `eq_ref`（join 被驱动表走主键) > `ref`（普通二级索引等值）> `range`（between/in/范围）> `index`（扫整棵索引树）> `ALL`（全表扫，**必须优化**）。

`Extra` 关键字：`Using index`（覆盖索引，好）、`Using index condition`（ICP 生效）、`Using where`（Server 层过滤）、`Using filesort`（额外排序，要优化）、`Using temporary`（临时表，group by / distinct / union，要优化）、`Using join buffer`（被驱动表连接列没索引）。

展开要点：
- `rows` 是**预估值**不是真实值；要看真实行数用 `EXPLAIN ANALYZE`（8.0.18+）或 `optimizer_trace`。
- 慢查询开启：`slow_query_log=ON`、`long_query_time=0.5~1`、`log_queries_not_using_indexes`；分析用 `mysqldumpslow` / `pt-query-digest`。
- **慢 SQL 治理流程**（按顺序说，别跳步）：① 慢日志捞出 TOP N（按「总耗时 = 次数 × 单次」排，高频小慢 SQL 更值得治）② `EXPLAIN` 定位瓶颈 ③ **优先加/改索引或改写 SQL**（成本最低、可回滚）④ 仍不行就减少数据量（归档、汇总表、游标分页）⑤ 再不行才动架构（读写分离、分区、分表、缓存）。

项目话术：「POS 中台的统计接口就是按这个顺序治的：慢日志里发现 `type=ALL`、`rows` 几百万，加联合索引后变成 range、扫几百行，P99 从秒级降到百毫秒内。」

## 3. 事务与 MVCC

> **一句话**：A 靠 undo log、D 靠 redo log、I 靠 MVCC + 锁，C 是应用层要保证的目标；InnoDB 默认 RR，靠 **MVCC 解决快照读一致性、靠间隙锁在很大程度上解决幻读**。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
| --- | --- | --- | --- | --- |
| 读未提交 RU | 会 | 会 | 会 | 直接读最新数据，不加锁 |
| 读已提交 RC | 不会 | 会 | 会 | **每次快照读都生成 ReadView** |
| 可重复读 RR（默认） | 不会 | 不会 | 基本不会 | **只在第一次快照读生成 ReadView** + Next-Key Lock |
| 串行化 | 不会 | 不会 | 不会 | 读写都加锁，并发度最低 |

- 三个现象：**脏读** = 读到别人未提交的数据；**不可重复读** = 同一事务两次读同一行结果不同（别人 update 并提交）；**幻读** = 同一事务两次**范围**查询行数变了（别人 insert/delete 并提交）。
- **MVCC 原理**（要能连贯讲）：① 每行有隐藏字段 `DB_TRX_ID`（最后改它的事务 ID）和 `DB_ROLL_PTR`（指向 undo log）② 每次 update/delete 把旧版本写进 undo log，串成**版本链**，链头是最新版本 ③ 快照读时生成 **ReadView**：`m_ids`（活跃事务列表）、`min_trx_id`、`max_trx_id`、`creator_trx_id` ④ 沿版本链找第一个可见版本：`trx_id < min_trx_id` 可见，`>= max_trx_id` 不可见，在 `m_ids` 里不可见，等于 `creator_trx_id` 可见。
- **RC 每次快照读都生成 ReadView，RR 只在第一次生成**——这是两个级别差异的根本。
- **当前读 vs 快照读**：普通 `select` 是快照读，读 MVCC 版本、不加锁；`select ... for update`、`lock in share mode`、`insert/update/delete` 是当前读，读最新版本并加锁。RR 下快照读靠 ReadView 天然看不到别人的插入，当前读靠 Next-Key Lock 阻止插入。

常见追问：为什么很多公司改 RC？→ RR 的间隙锁范围大、更容易死锁，RC 加锁更少并发更好；但 RC 必须配 `binlog_format=ROW`。undo log 什么时候删？→ 没有更早的 ReadView 需要它时，由 purge 线程清理。

项目话术：「LoveSpouse 的会话列表我用快照读 + 覆盖索引，完全不加锁；只有扣金币、更新未读数这种要防并发的地方才用原子 update 或当前读。」

## 4. 锁

> **一句话**：InnoDB 的锁是「表级意向锁 + 行级记录锁/间隙锁/临键锁」，而且**行锁是加在索引上的**；RR 下等值查一个不存在的记录也会加间隙锁，这是死锁最常见的来源。

展开要点：
- **全局锁**：`flush tables with read only`，全库只读，用于全库备份（现在多用 `mysqldump --single-transaction` 或 XtraBackup 避开它）。
- **表锁**与 MDL：`lock tables`；**MDL 元数据锁**在 DDL 时自动加，**长事务会让 DDL 拿不到 MDL，进而阻塞后面所有查询**，这是经典生产事故，所以「DDL 前先看有没有长事务」。
- **行锁三种**：Record Lock（锁单条记录）、Gap Lock（锁记录之间的**间隙**，只在 RR 存在，防插入）、Next-Key Lock（前两者组合，RR 默认）；唯一索引等值命中时退化为 Record Lock。
- **意向锁 IS/IX**：表级、InnoDB 自动加，作用是让表锁不用逐行扫描就能判断表里有没有行锁；IS/IX 互相兼容。
- **关键结论**：`update` 的 where 没走索引 → 全表扫描 → 锁住所有记录和间隙，**等于表锁**。这是「update 必须走索引」的根本原因。

死锁：

> **一句话**：InnoDB 检测到死锁会**直接回滚代价小的那个事务**；排查靠 `show engine innodb status` 的 `LATEST DETECTED DEADLOCK` 段。

- 排查手段：`show engine innodb status\G` 看 `LATEST DETECTED DEADLOCK`（含两个事务的 SQL、持有/等待的锁）；8.0 看 `performance_schema.data_locks` / `data_lock_waits` 实时锁等待；`innodb_print_all_deadlocks=ON` 把全部死锁打进 error log。
- 四类典型成因：① 两事务以相反顺序更新多行 → **统一加锁顺序** ② 间隙锁 + 插入意向锁冲突（RR 特有）→ 改 RC，或唯一键场景用 `insert ... on duplicate key update` ③ 大事务持锁太久 → 拆小事务 ④ 无索引 update 锁范围过大 → 补索引。
- 避免死锁的实操：事务短小、按固定顺序访问、批量更新先 `order by id`；并发扣减用原子 `update ... where balance >= x` 代替「先 `for update` 再 update」；`innodb_lock_wait_timeout` 从 50s 调到 5~10s 快速失败；应用层捕获死锁错误码 **1213** 后重试。

项目话术：「FlashRoot 的批量返点结算是这么设计的：按客户 ID 排序后统一顺序加锁、每 100 单一个事务、结算单上加唯一索引——三层一起把死锁和重复结算都挡掉。」

## 5. 日志：redo / undo / binlog

> **一句话**：redo log 保持久性（崩溃恢复，物理日志、循环写），undo log 保原子性和 MVCC（逻辑日志、回滚 + 版本链），binlog 是 Server 层的归档日志（逻辑日志、追加写，用于主从复制和数据恢复）。

| | redo log | undo log | binlog |
| --- | --- | --- | --- |
| 层级 | InnoDB 引擎 | InnoDB 引擎 | **Server 层**（所有引擎共用） |
| 类型 | 物理日志（页的修改） | 逻辑日志（反向操作） | 逻辑日志（SQL 或行变更） |
| 写入 | 循环写，固定大小 | 追加，可被 purge 清理 | 追加写，按文件滚动 |
| 作用 | 崩溃恢复、持久性 | 回滚、MVCC 版本链 | 主从复制、按时间点恢复 |
| 格式 | — | — | STATEMENT / ROW / MIXED |

展开要点：
- **WAL（Write-Ahead Logging）**：先写日志再写数据页——把「随机写 16KB 数据页」变成「顺序追加小记录」。事务提交只要 redo 落盘就算成功，脏页由后台线程慢慢刷。
- `innodb_flush_log_at_trx_commit=1`（每次提交 fsync，不丢）+ `sync_binlog=1` ＝ **「双 1 配置」**，金融级不丢数据的标准。
- **两阶段提交**（为什么需要）：让 redo log 和 binlog 逻辑一致。① **prepare**：写 redo log 标记 prepare ② 写 binlog ③ **commit**：redo log 标记 commit。崩溃恢复判断：redo 处于 prepare 且 binlog 完整 → 提交；binlog 不完整 → 回滚。
- **组提交（group commit）**：多个事务的 fsync 合并成一次，显著提升并发写吞吐；相关参数 `innodb_log_file_size`（大一些减少 checkpoint 频率）、`innodb_log_buffer_size`。

常见追问：为什么不直接刷数据页？→ 数据页是 16KB 随机写，一个事务可能改多个页，直接刷盘 IO 量和并发都不可控；redo 是顺序追加的小记录。redo 写满了会怎样？→ 触发 checkpoint 刷脏页腾空间，刷盘跟不上写入就会被阻塞，表现为性能抖动。

## 6. 优化实践

> **一句话**：优先改索引和 SQL（成本最低），其次减少数据量（归档、汇总表、游标分页），最后才动架构（读写分离、分区、分表）。

**① 大表分页优化**（必背）：慢的原因是 `limit 1000000, 20` 要先扫 100 万行再丢掉。三种改法：`where id > #{last_id} order by id limit 20`（游标，最快，适合「下一页」）；**延迟关联** `select t.* from t join (select id from t order by id limit 1000000, 20) x using(id)`（先在索引上分页拿 id，回表次数从 100 万降到 20）；业务上禁止深分页（超过 N 页必须收窄条件）。

**② 批量插入**：一条 insert 多组 values（500~1000 行/批，注意 `max_allowed_packet`）；用事务包住避免每行一次自动提交 fsync；幂等用 `insert ignore` 或 `insert ... on duplicate key update`；导入场景可临时 `set unique_checks=0, foreign_key_checks=0`，更大的量上 `load data infile`。

**③ Go 连接池（`database/sql` / GORM）**——问「GORM 怎么配连接池」就是问这四个：

| 参数 | 默认值 | 建议 |
| --- | --- | --- |
| `SetMaxOpenConns` | 0（**无限制**） | 必须设！≈ DB 最大连接数 / 实例数，或 CPU 核数 × 2~4 |
| `SetMaxIdleConns` | 2 | 建议 = MaxOpenConns，避免频繁建连 |
| `SetConnMaxLifetime` | 0（永不过期） | 30min~1h，避开 MySQL `wait_timeout` 与云 LB 断连 |
| `SetConnMaxIdleTime` | 0 | 5~15min |

Go 代码：`sqlDB, _ := gormDB.DB()` 然后调上面四个方法。两个高频坑：① `MaxOpenConns` 不设（=0 无限制）在高并发下会打爆 MySQL 的 `max_connections`（默认 151），报 `Too many connections`；② `ConnMaxLifetime` 太大时连接会被 MySQL 单方面关闭 → `invalid connection`，所以要配 lifetime + 重试。

**④ 其它零碎**：`count(*)` 和 `count(1)` 在 InnoDB 下性能一样（都扫索引），别用 `count(字段)`（要判 NULL）；`join` 要小表驱动大表且被驱动表连接列必须有索引，否则出现 `Using join buffer`；大事务要拆（undo 膨胀、锁持有久、主从延迟放大、回滚代价高）；冷热分离，POS 流水 3 个月内留 MySQL、更老的归档到历史表或对象存储。

## 7. 分库分表【理解层】

> **一句话**：**先想能不能不做**——加索引、加缓存、建汇总表、归档冷数据、读写分离能扛很久；真到单表千万级以上、单库写入到瓶颈，再「先垂直拆业务，再水平拆数据」。

展开要点：
- 触发信号（说判断依据，别只背数字）：单表 > 1000 万~2000 万行且 B+ 树变 4 层；单库 QPS/连接数/磁盘 IO 到顶，加从库也扛不住写；单表大到备份和 DDL 都做不了。
- 怎么分：**垂直分库**（按业务拆，微服务化的自然结果）、**垂直分表**（text/json 大字段拆到扩展表，主表变窄一页装更多行）、**水平分表**（`user_id % N` 哈希均匀但扩容难，或按时间范围好扩容但可能热点）。
- 中间件：ShardingSphere、Vitess、MyCat。**【理解层】我理解它的原理是 SQL 解析 → 路由 → 改写 → 执行 → 结果归并，但我们项目量级没到，没上过分表中间件。**
- 分表带来的四个问题（面试官真正想听的）：① **分布式 ID** → 雪花算法（1 符号位 + 41 时间戳 + 10 机器位 + 12 序列）、号段模式（Leaf）、Redis INCR、多步长自增 ② **跨库 join** → 拆成多次查询在应用层拼，或加冗余字段 ③ **分布式事务** → 能不用就不用，用本地消息表 + MQ 重试做最终一致，或 TCC ④ **跨库分页/排序/聚合** → 各分片取前 N 再内存归并，深分页必须禁止。
- 附加：扩容迁移用「双写 + 校验 + 灰度切换」；全局唯一索引要靠去重表或布隆过滤器。

项目话术：「POS 中台的流水表我是**先加索引 + 按门店/日期建预聚合汇总表 + 冷数据归档**扛住的；如果哪天单表过亿，我会优先按 `shop_id` 哈希分表，因为门店是最主要的查询入口，同时用雪花 ID 做全局单号。」

## 8. 主从复制与读写分离【理解层】

> **一句话**：主库写 binlog → 从库 IO 线程拉取写 relay log → 从库 SQL 线程回放；读写分离就是写走主、读走从，用中间件或客户端路由（**go-zero 的 MySQL 配置原生支持主从**）。

展开要点：
- 三种复制模式：**异步**（默认，主库提交即返回，主挂可能丢数据）、**半同步**（至少一个从库 ack 才返回）、**组复制 MGR**（Paxos 变体，一致性最强性能最差）。binlog 格式推荐 **ROW**（STATEMENT 遇到 `now()`、`uuid()` 会导致主从不一致）。
- **主从延迟的原因**：从库 SQL 线程默认**单线程回放**（5.7+ 可开并行复制 `slave_parallel_workers`）；主库大事务/大批量 DML 要回放很久；从库承担大量读抢不到资源；网络延迟、从库配置差。
- **主从延迟的处理**（必须有方案）：**写后读一致性**——刚写完的一段时间内该用户的读也走主库（按用户 ID 粘滞）；关键读显式走主（支付结果、金币余额、结算数据）；监控 `seconds_behind_master`，延迟超阈值自动把读切回主库；治本是拆大事务、开并行复制、提升从库配置；兜底是写入时同步塞一份到 Redis（见 Redis 第 4 节）。

项目话术：「我们用 go-zero 的 MySQL 配置把统计类查询路由到从库，事务和余额类查询强制走主库；遇到刚写完立刻读的场景，我会让这条读请求也走主库，或者写的时候往 Redis 塞一份兜底。」

## 9. MySQL 场景题

### 9.1 POS 数据中台：海量销售流水表的设计与优化

> **一句话**：流水表是**只追加、按「门店 + 时间」查、写多读少**的形态，核心四件事——**写入快、幂等、查询走索引、老数据归档**。

1. **表结构**：主键自增 bigint（顺序写避免页分裂）；业务单号（门店 + POS 机号 + 单号）建**唯一索引**做幂等，RabbitMQ 重复消费时 `insert ignore` 直接挡掉；金额用 `decimal(12,2)` 或 `bigint` 存分，**绝不用 float/double**；高频过滤列是 `shop_id`、`biz_date`、`pos_id`、`order_no`、`status`；大字段（明细 JSON、发票原文）拆到扩展表或对象存储；时间用 `datetime` 而非 `timestamp`（没有 2038 问题、无隐式时区转换）。
2. **索引设计**：`uk_order(shop_id, pos_id, order_no)` 管幂等和按单号查；`idx_shop_date(shop_id, biz_date, id)` 管列表 + 排序 + 分页；`idx_date_status(biz_date, status)` 给对账/日结任务用。
3. **写入**：RabbitMQ 消费端**攒批**（满 500 条或 200ms 触发），用多 values 一条 SQL 写，把高频小事务换成低频大事务；批不能太大，注意 `max_allowed_packet` 和主从延迟；幂等双保险 = 唯一索引 + 消费记录表（`msg_id` 唯一）。
4. **查询**：列表接口**强制带门店 + 时间范围**，不给全表扫的机会，深分页改游标；报表/对账不做实时聚合，走**按天预聚合的汇总表**（消费时累加或定时任务算）。
5. **数据量**：按 `biz_date` 做 RANGE 分区，或按年归档到历史库；更彻底是按 `shop_id` 哈希分表；冷热分离——3 个月内热数据在 MySQL，更老的落 ClickHouse / 对象存储。

### 9.2 LoveSpouse：会话与消息表设计

> **一句话**：消息表是**按会话维度分页查、只追加、量最大**；会话表是**按用户维度查列表、更新频繁**——两张表分开设计，索引完全不同。

1. **拆表**：`conversation`（会话：user_id、character_id、最后一条消息摘要、未读数、更新时间）与 `message`（消息明细）分开。
2. **message 关键设计**：主键自增 id，**同时当消息序号用**，翻历史消息直接 `where conversation_id=? and id < #{last_id} order by id desc limit 20`，**不用 offset**；`msg_id`（腾讯 IM 的消息 ID）建**唯一索引**做幂等——IM 会重复投递/乱序，`insert ignore` + 唯一键是最简单可靠的方案；大内容（语音 URL、长 JSON）只存元数据 + URL，正文落对象存储；带上消息类型、发送方（user/character）、token 消耗、已读标记。
3. **conversation 关键设计**：未读数用**原子 update** 累加/清零，不要「先 select 再 update」；列表查询走 `idx_user_update(user_id, updated_at desc)`，一条索引同时满足过滤和排序；「最近联系人列表」这种高频接口用 **Redis ZSet**（score = 最后消息时间）兜一层，MySQL 只做持久化。
4. **与分层记忆的配合**：近期对话和会话摘要落 MySQL；稳定事实、热态记忆放 Redis；**摘要生成是异步的（走 MQ），不阻塞对话主链路**。
5. **归档**：老会话消息按月归档，或按 `user_id` 分表。

### 9.3 FlashRoot：返点结算的金额精度与并发扣减

> **一句话**：金额一律用 `decimal` 或整数分，**绝不用 float**；并发扣减靠**数据库原子 update + 条件判断**（或乐观锁版本号），绝不「先查再改」。

1. **精度**：float/double 是二进制浮点，`0.1+0.2 != 0.3`，金额必须 `decimal(12,2)` 或 `bigint` 存分；Go 侧用 `shopspring/decimal` 或直接 `int64` 存分，**不要用 float64**；返点比例（如 3.5%）也走 decimal，先乘后除、最后统一 `Round`，并把舍入规则（四舍五入还是银行家舍入）写进结算文档。
2. **并发扣减三选一**（按推荐度，要讲出为什么）：
   - ① **原子 update（推荐）**：`update account set balance = balance - ? where id = ? and balance >= ?`，再看 `RowsAffected` 是否为 1。这一条 SQL 同时完成「检查 + 扣减」，靠 InnoDB 行锁保证原子性，**没有 TOCTOU 问题**。
   - ② **乐观锁**：`update account set balance = ?, version = version + 1 where id = ? and version = ?`，失败重试。
   - ③ **悲观锁**：`select ... for update` → 计算 → update。必须走主键/索引，否则锁全表；事务必须短。
   - **不推荐**：先 `select` 出来、在 Go 里算完再 `update`——并发下必然超扣。
3. **幂等与对账**：结算单表建 `uk_settle(period, customer_id, type)` 唯一索引，重复触发直接失败，配合状态机 `待结算 → 结算中 → 已结算`；每笔金额变动都写**流水账（ledger）**，余额 = 流水累加，余额字段只是冗余、可对账可追溯；结算任务用 Redis 分布式锁保证同一批次只有一个实例在跑；每日对账 `余额` vs `流水累计`，不一致就告警。
4. **多级客户返点**：返点是层级递归的（一级/二级客户层层分佣），用**路径枚举或闭包表**存层级关系，一次查出链路，避免递归查库。

---

# 第二部分 · Redis（40%）

## 1. 数据结构与选型

> **一句话**：选型只看两点——**数据形态 + 访问模式**；单值用 String、对象字段级更新用 Hash、队列/时间线用 List、去重与集合运算用 Set、排行与时间序用 ZSet、海量去重计数用 Bitmap/HLL、LBS 用 GEO、专业消息队列用 Stream。

| 结构 | 底层实现 | 常用命令 | 结合我们项目的场景 |
| --- | --- | --- | --- |
| **String** | SDS 动态字符串 | `GET/SET/INCR/SETNX/SETEX` | **金币余额**、分布式锁、验证码、角色配置 JSON 缓存、接口计数器 |
| **Hash** | listpack → hashtable | `HSET/HGET/HINCRBY` | 角色人设多字段配置、用户会话状态（字段级更新、省内存） |
| **List** | quicklist（链表 + listpack） | `LPUSH/RPOP/LRANGE` | IM 离线消息兜底、最新 N 条消息 |
| **Set** | intset → hashtable | `SADD/SISMEMBER/SINTER` | 用户标签、当日已推送用户去重、共同角色 |
| **ZSet** | listpack + **skiplist** | `ZADD/ZRANGE/ZSCORE` | **最近会话列表**（score = 最后消息时间）、金币消耗排行榜、限流滑动窗口 |
| **Bitmap** | String 的位操作 | `SETBIT/BITCOUNT` | 用户签到、日活统计（1 亿用户仅 12MB）、布隆过滤器底层 |
| **HyperLogLog** | 稀疏/稠密编码 | `PFADD/PFCOUNT` | UV 统计、广告曝光去重人数（**路由器广告**），误差 0.81% |
| **GEO** | ZSet + geohash | `GEOADD/GEOSEARCH` | 附近的人/附近商户（本项目用不上，知道即可） |
| **Stream** | rax + listpack | `XADD/XREADGROUP` | 带消费者组和 ACK 的消息队列；我们消息用的是 RabbitMQ，这个是了解 |

常见追问：ZSet 为什么用跳表不用红黑树？→ ① 范围查询天然有序，实现比平衡树简单得多 ② 不用旋转，实现/并发复杂度低 ③ 层高概率可调、内存可控 ④ 平衡树做范围查询也要中序遍历，代码更复杂。Hash 什么时候转 hashtable？→ 元素数 > 128 或单 value > 64 字节（阈值可配 `hash-max-listpack-entries`）。

## 2. Redis 为什么快

> **一句话**：**纯内存操作 + 命令执行单线程（免锁免上下文切换）+ epoll IO 多路复用 + 精挑的数据结构（跳表、渐进式 rehash）**，四条缺一不可。

展开要点：
- **内存**：命令路径完全不碰磁盘（RDB/AOF 是持久化，不在命令路径上）。
- **单线程**：指**命令执行**单线程——无锁、无 CPU 上下文切换、无竞争，所有命令天然原子。
- **IO 多路复用**：epoll + 自研事件循环，单线程扛上万连接。
- **6.0 的多线程**：网络 IO 读写是多线程的（`io-threads`），**命令执行仍然是单线程**——这个追问很容易被问，一定要说清。
- **高效数据结构**：SDS（O(1) 取长度、二进制安全、预分配减少拷贝）、跳表（范围查询 O(logN)）、listpack，以及**渐进式 rehash**——字典维护 `ht[0]/ht[1]` 两张表 + `rehashidx`，把 rehash 代价摊到每次操作上，避免一次性卡顿。
- 附加：`redisObject` 整数共享（0~9999）、RESP 协议简单。追问「为什么不用多线程执行命令」→ 加锁 + 上下文切换的开销可能比单线程还大，瓶颈通常在网络 IO 和内存；真要用多核就多实例 + Cluster。

## 3. 缓存三大问题

> **一句话**：**穿透是查不存在的数据（缓存和 DB 都没有）、击穿是一个热 key 过期、雪崩是大量 key 同时过期**——现象像，解法完全不同，先分清再答。

| 问题 | 现象 | 解法 |
| --- | --- | --- |
| **穿透** | 查 DB 里也不存在的数据，缓存永不命中，请求全打到 DB | ① 缓存空值（`SET key "" EX 60`）② 布隆过滤器前置拦截（有误判，只能挡不存在的）③ 入口参数校验 + 限流 |
| **击穿** | **单个热 key 过期瞬间**，大量并发同时打到 DB | ① 互斥锁 / singleflight：只放一个请求查 DB，其余等待或返回旧值（Go 用 `golang.org/x/sync/singleflight`）② 逻辑过期：key 不设 TTL，value 里带过期时间，读到过期先返回旧值 + 异步刷新 ③ 热点常驻 + 后台定时更新 |
| **雪崩** | **大量 key 同一时刻过期**，或 Redis 整体宕机，DB 被打满 | ① 过期时间加随机抖动（`base + rand(0,300)`）② 多级缓存（本地 + Redis）③ 哨兵/Cluster 高可用 ④ DB 层限流 + 熔断降级 ⑤ 缓存预热 |

项目话术：
- 击穿（LoveSpouse）：「角色配置是人人要读的热点，我用**逻辑过期 + 本地缓存**：配置在服务启动时加载进进程内存，Redis 存一份带逻辑过期时间的 JSON，过期后由一个 goroutine 异步刷新——用户永远读到可用数据，不存在击穿窗口。」
- 穿透：「按 ID 查消息/流水的接口，我在入口做参数校验 + 空值缓存 60 秒，防止有人用随机 ID 刷穿到 MySQL。」
- 雪崩：「批量写的 key 我都加了随机 TTL 抖动，避免同一时刻集体失效。」

## 4. 缓存与数据库一致性

> **一句话**：**没有并发就没有一致性问题**——标准答案是 Cache Aside（**先更新 DB，再删除缓存**），接受短暂不一致，用「延迟双删 + TTL 兜底 + 订阅 binlog」逼近最终一致；要强一致就只能加锁或不用缓存。

展开要点：
- 为什么**删缓存**而不是更新缓存：① 更新缓存有并发写覆盖问题 ② 缓存可能是聚合结果，不一定能直接算出来 ③ 懒加载更省资源（不读就不写）。
- 为什么**先更新 DB 再删缓存**（顺序不能反）：先删缓存再更新 DB 的话，删除到更新完成之间读请求会把**旧值**加载回缓存，脏数据可能长期存在；反过来只有「读请求恰好在更新前读到旧值、又在删除后才写回」这个极小窗口才脏，下次读会自然修复。
- **延迟双删**：更新 DB → 删缓存 → 异步延迟 500ms~1s 再删一次；缺点是该延迟多久难定、只能异步。
- 更强方案：**订阅 binlog**（Canal / go-mysql）异步删缓存，业务代码解耦、可靠性最高；MQ 补偿删除 + 重试；读写共用一把分布式锁（强一致，代价大，只用在高价值数据）。
- 一致性级别：**强一致**只能靠锁/事务，或者干脆不缓存；**最终一致**是缓存场景的默认目标——**「TTL 是最后的兜底」这句话面试官很爱听**，即使删缓存失败，过期后也会自动修正。

项目话术：
- 「角色配置、剧情配置是**读多写极少**的，我直接用『更新 DB 后删除缓存 + 5 分钟 TTL 兜底』，几乎没有并发写，不值得为它上 binlog 订阅。」
- 「金币余额我不做妥协：**扣减在 Redis 用 Lua 原子完成（快、防超扣），MySQL 只做账本**；后台把 Redis 流水异步落 MySQL，定时对账校验 Redis 余额和 MySQL 流水累计的差额并告警——不一致必须是可发现、可修复的。」

## 5. 分布式锁

> **一句话**：正确姿势是 `SET key value NX PX 30000`，value 存**唯一请求标识**，释放用 **Lua 比对 value 再删**，业务没跑完由后台协程**续期**；单实例 Redis 上这套就够了，跨机房强一致才需要讨论 Redlock。

1. **加锁**：`SET lock:xxx {唯一标识} NX PX 30000`。**NX 和 PX 必须是一条原子命令**——老写法 `SETNX` + `EXPIRE` 两条不是原子的，中间进程挂掉就死锁。
2. **value 为什么必须唯一**（如 `实例ID:协程ID:UUID`）：防止**误删别人的锁**——A 业务超时（锁自动过期）→ B 拿到锁 → A 执行完直接 `DEL` 把 B 的锁删了。
3. **释放必须用 Lua**（保证「判断 + 删除」原子）：
   ```lua
   if redis.call('get', KEYS[1]) == ARGV[1] then
     return redis.call('del', KEYS[1])
   else
     return 0
   end
   ```
4. **锁续期（看门狗）**：起一个后台 goroutine，每隔 `TTL/3` 检查业务是否还在跑，是就 `PEXPIRE` 续回 30s，业务结束就取消续期。Java 的 Redisson 内置了看门狗，**Go 生态没有 Redisson**，一般自研续期协程，或直接用 `redsync`（Redlock 的 Go 实现）。
5. **Redlock 争议**（能说出正反两方就是加分）：做法是向 N（通常 5）个**独立** Redis 实例依次加锁，多数（N/2+1）成功且总耗时小于锁有效期才算成功。质疑方（Martin Kleppmann）：依赖各实例时钟不跳变，GC/网络停顿会让锁过期后仍继续写，**它解决不了「锁失效之后的正确性」**，不如用 fencing token（每次加锁带单调递增版本号，写资源时校验）。支持方（antirez）：Redlock 的目标是「高效、容错的互斥锁」，不是「绝对正确」。
   **怎么答**：「我们的场景是防重复执行（互斥），不是防并发写坏数据，所以单实例 Redis 锁 + 唯一 value + Lua 释放就够；真要强正确性，我会用带 fencing token 的方案，或者干脆用数据库唯一键——**能用唯一索引解决的幂等问题就不要用分布式锁**。」

**跨实例通话控制（LoveSpouse 真实场景）**
- 为什么需要锁：实时语音通话（Gemini Live）是有状态长连接，用户可能同时从 App、重连、多设备发起多路；不加互斥会出现**两个实例同时往同一会话推语音、金币被双重计费**。
- 做法：`SET call:lock:{userId} {instanceId:sessionId} NX PX 60000`，拿到锁的实例才能建立通话；通话中由持有者定期心跳续期；结束时 Lua 校验 value 后释放；实例崩了锁会自动过期，用户重连能重新拿到锁。**一定要说出「TTL 是分钟级 + 心跳续期」**，证明你想过实例崩溃的场景。
- 其它要点：锁粒度要小（按 userId / 业务 ID），不要一把大锁；拿不到锁不要死等 → 自旋 + 退避 + 超时失败，或直接返回「操作进行中」；可重入要么用 Hash 存重入次数（`{holder: count}`），要么在设计上避免——**更推荐避免**。

## 6. 持久化

> **一句话**：RDB 是**某一时刻的全量快照**（恢复快、可能丢数据），AOF 是**追加的写命令日志**（丢得少、恢复慢、文件大）；生产用**混合持久化**，既不丢太多数据也能快速恢复。

| | RDB | AOF |
| --- | --- | --- |
| 内容 | 内存数据的二进制快照 | 写命令追加日志（RESP 格式） |
| 触发 | `save`（阻塞）/ `bgsave`（fork 子进程 + 写时复制 COW）/ 定时策略 | `appendonly yes` + `appendfsync` |
| 恢复速度 | 快（直接载入内存） | 慢（逐条回放命令） |
| 数据安全 | 差（两次快照间宕机全丢） | 好（取决于 fsync 策略） |
| 文件体积 | 小（压缩过） | 大（需 `BGREWRITEAOF` 压缩） |
| 阻塞风险 | fork 时内存越大阻塞越久，COW 占额外内存 | `always` 每次 fsync 影响性能；rewrite 的 fork 也会阻塞 |

展开要点：
- `appendfsync` 三档：**`everysec`（默认、生产推荐，最多丢 1 秒）**、`always`（最安全，QPS 掉到几百，几乎不用）、`no`（交给 OS，可能丢几十秒）。
- **混合持久化**（4.0+，`aof-use-rdb-preamble yes` 默认开）：AOF rewrite 时先把当前内存以 RDB 格式写进 AOF 文件头部，后续增量命令追加在后面，重启时先加载 RDB 部分再回放增量——**兼具两者优点，这就是标准答案**。
- 主从 + 持久化的一个坑：主库为了性能不持久化时，主库重启后数据为空会把空数据全量同步给从库，造成整体丢数据；要靠从库持久化 + 哨兵，并配 `min-replicas-to-write` 之类的参数防。
- 怎么选：纯缓存（数据能从 DB 重建）→ 只开 RDB 就够；存关键数据 → 混合持久化 + 主从。

## 7. 过期与淘汰

> **一句话**：过期是「**惰性删除 + 定期删除**」两者配合，淘汰是内存达到 `maxmemory` 后按 8 种策略之一踢数据；**生产必须显式设 maxmemory 和淘汰策略，并区分「过期」与「淘汰」**。

展开要点：
- **惰性删除**：访问 key 时才检查是否过期——CPU 友好、内存不友好（不访问的 key 不释放）。
- **定期删除**：每 100ms 随机抽 20 个设了 TTL 的 key，删掉过期的；若过期比例 > 25% 就再抽一轮。**采样 + 自适应**，避免全表扫描卡住主线程。
- 8 种淘汰策略：

| 策略 | 范围 | 算法 |
| --- | --- | --- |
| `noeviction`（默认） | — | 不淘汰，写命令直接报 OOM 错 |
| `volatile-lru` / `volatile-lfu` / `volatile-random` / `volatile-ttl` | 只淘汰**设了 TTL** 的 key | LRU / LFU / 随机 / 剩余 TTL 最小优先 |
| `allkeys-lru` / `allkeys-lfu` / `allkeys-random` | 所有 key | LRU / LFU / 随机 |

- 怎么选：**缓存场景用 `allkeys-lfu`（次选 `allkeys-lru`）**——LFU 按访问频率淘汰，能防「偶发批量扫描把热数据挤掉」；同一实例混了持久化数据就用 `volatile-*`；**绝不能留默认 noeviction**（写满直接报错）。
- Redis 的 LRU 是**近似 LRU**：随机采样 `maxmemory-samples`（默认 5）个 key 淘汰最久未用的，不是精确 LRU（精确 LRU 要额外链表，内存不划算）。
- **内存告警处理流程**：① `info memory` 看 used_memory / maxmemory / `mem_fragmentation_ratio`（碎片率 > 1.5 考虑 `activedefrag` 或重启）② `redis-cli --bigkeys` 找大 key，`memory usage <key>` 看单个 key 大小 ③ 处理大 key：拆分、只存必要字段、设合理 TTL、**用 `unlink` 代替 `del` 异步删除** ④ 查有没有「忘了设 TTL」的 key 泄漏 → 补 TTL + 加监控 ⑤ 扩容（加内存 / 上 Cluster 分片）。

## 8. 高可用【理解层】

> **一句话**：演进顺序是「**主从解决读扩展和备份 → 哨兵解决主库故障自动切换 → Cluster 解决单机内存与写入瓶颈**」，能把这个顺序和各自职责说清就够了。

- **主从**：从库 `replicaof` 主库，全量同步（RDB）+ 增量同步（`repl_backlog` 环形缓冲）；从库默认只读。**它本身不是高可用方案**（主库挂了要人工切）。
- **哨兵 Sentinel**：独立进程集群（通常 3 个奇数个），负责监控、通知、**自动故障转移**、配置中心。流程是：**主观下线**（单个哨兵认为挂了）→ **客观下线**（quorum 个哨兵都认为挂了）→ 选举 leader 哨兵 → 选新主库（优先级 + offset 最大 + runid 最小）→ 通知客户端新地址；客户端要支持哨兵（go-redis 的 `NewFailoverClient`）。两个坑：切换期间有秒级不可用；**脑裂**（旧主库没死透仍在接受写）→ 用 `min-replicas-to-write` 限制。
- **Cluster 分片**：**16384 个槽（slot）**，`CRC16(key) mod 16384` 决定 key 落在哪个节点；客户端重定向 **`MOVED`**（槽已永久迁移）/ **`ASK`**（槽正在迁移中的临时重定向）；多 key 操作要用 **hash tag** 强制落到同一槽，否则报 `CROSSSLOT`——**结合项目**：LoveSpouse 金币相关的 key 用 `{coin}:{userId}` 这种 tag；故障转移靠 gossip + 主从 + 半数以上主节点判定；槽迁移用 `CLUSTER SETSLOT ... MIGRATING/IMPORTING` 在线迁移。限制：不支持跨槽多 key 和跨槽事务、只能有 db0、批量操作要拆或加 hash tag。
- 一句话演进：「量小 → 单机 + 持久化；读多 → 主从；要自动切换 → 哨兵；单机内存/QPS 到顶 → Cluster。」

## 9. 实战问题：大 key / 热 key / Pipeline / 阻塞

> **一句话**：**大 key 是「单次操作耗时过长阻塞单线程」，热 key 是「单个分片被打爆」**；前者靠拆，后者靠多级缓存 + 分散。

- **大 key 的危害**：`del` / `hgetall` / `lrange` 一个大 key 会阻塞单线程几百毫秒 → 后面所有请求排队；主从复制、RDB fork 都受影响；Cluster 下还会造成数据倾斜。判定经验值是 String > 10KB、集合元素 > 5000；治理靠拆分（Hash 分桶、List 分片）、`unlink` 异步删除、用 `hscan/sscan/zscan` 分批读、**永远不要 `keys *`**；发现靠 `redis-cli --bigkeys`、`memory usage <key>`、`redis-rdb-tools` 离线分析 RDB。
- **热 key 的危害**：单个 key 的 QPS 占集群大半 → 它所在的分片被打满，**扩容也没用**（一个 key 只能在一个分片）。治理：① **本地缓存**（进程内 `sync.Map` / `bigcache` / `ristretto`，几秒过期）② key 加随机后缀分散到多个分片（读时随机选一个，写时广播）③ 热 key 读从库。发现：`redis-cli --hotkeys`（依赖 LFU）、客户端埋点统计；`monitor` 慎用（会降性能）。
- **Pipeline**：把 N 条命令一次发出、一次收结果，**省的是 RTT，不是服务端工作量**；适合批量 get/set，一次别塞太多（几万条会占内存），建议每批几百~一千；**Pipeline 不保证原子性**（要原子用 Lua 或 MULTI 事务）。`MGET/MSET` 也是减少 RTT 的手段，且是单条命令。
- **Redis 阻塞的常见原因**（「Redis 为什么变慢」）：① 慢命令——`keys *`、大 key 的 `hgetall`、大集合的 `smembers`、`sort`、大范围 `zrange` ② 大 key 删除 → 用 `unlink` ③ **fork 阻塞**——bgsave / AOF rewrite 时 fork 要拷贝页表，内存越大越久（十 GB 可能几百 ms）④ **AOF fsync 阻塞**——`appendfsync always`，或 everysec 遇到慢磁盘 ⑤ **内存 swap**——物理内存不足被换到磁盘（灾难级，禁用 swap 或 `vm.swappiness=1`）⑥ 客户端连接数过多 / 输出缓冲区爆（`client-output-buffer-limit`）。排查手段：`slowlog get`（`slowlog-log-slower-than 10000`，单位微秒）、`info` 看 `latest_fork_usec`、`latency monitor`。

## 10. Redis 场景题

### 10.1 LoveSpouse：金币余额的并发扣减

> **一句话**：**Redis 扛并发扣减、MySQL 做账本**——Lua 脚本在 Redis 里原子完成「校验余额 + 幂等 + 扣减 + 写流水」，后台异步落 MySQL，靠唯一请求号双保险幂等，定时对账兜底。

1. **为什么余额放 Redis**：对话消耗金币是**每次交互都要扣**的超高频写，直接打 MySQL 会把行锁变成瓶颈；Redis 单线程 + Lua 天然原子。
2. **扣减必须用 Lua**（单独 `DECRBY` 不行，它既不校验余额也不写流水）：
   ```lua
   -- KEYS[1]=coin:balance:{userId}  ARGV[1]=amount  ARGV[2]=requestId
   local bal = tonumber(redis.call('get', KEYS[1]) or '0')
   if bal < tonumber(ARGV[1]) then return -1 end                              -- 余额不足
   if redis.call('setnx', 'coin:req:'..ARGV[2], 1) == 0 then return -2 end    -- 幂等：已处理
   redis.call('decrby', KEYS[1], ARGV[1])
   redis.call('rpush', 'coin:ledger:queue', ARGV[2]..':'..ARGV[1])
   return bal - tonumber(ARGV[1])
   ```
   ——「校验 + 幂等 + 扣减 + 写流水」四件事在**一个 Lua 里完成**，这是这道题的核心答案。
3. **幂等**：`requestId`（来自腾讯 IM 消息 ID 或客户端业务号）既做 Redis 幂等键，MySQL 流水表也建 `uk_request_id` 唯一索引——**双保险，Redis 丢数据时 MySQL 还能挡重复**。
4. **落库**：后台 goroutine / 消费者把 Redis 里的流水队列**批量**写 MySQL；写成功后让幂等键自然过期或显式删除。
5. **对账与兜底**：余额不足返回明确错误码、前端引导充值，不静默失败；定时对账 `MySQL 流水累计` vs `Redis 余额`，不一致就告警并**以 MySQL 账本为准回写 Redis**；充值路径相反——走 MySQL 事务（写充值流水 + 更新余额），成功后再删/更新 Redis 缓存。**即「扣减以 Redis 为准、入账以 MySQL 为准，两边靠流水对齐」。**
6. **冷启动**：Redis 重启后余额从 MySQL 账本重算并预热；**绝不接受「余额只存在于 Redis」**。
7. **追问「Redis 挂了会不会超扣/少扣」**→ 会**少扣**（余额偏高），不会超扣（Lua 里校验了余额）；恢复后按账本重建余额，少扣的部分靠对账补齐。**这个回答能证明你真的想过故障场景。**

### 10.2 路由器广告：高并发 Portal 认证的缓存设计

> **一句话**：Portal 认证是典型的「**一次认证、多次放行**」——认证结果放 Redis 带 TTL，放行判断只查 Redis（加本地缓存二级），认证接口本身用限流 + 幂等挡住刷量。

1. **流程**：连 WiFi → 跳 Portal 页 → 手机号/微信授权 → 服务端校验 → 写认证态 → 网关/AC 放行。
2. **缓存设计**：认证态 `auth:{deviceMac}` → `{userId, expireAt}`，TTL = 会话时长（如 2 小时），**续期用 `EXPIRE` 而不是重新写入**；商户/广告位配置读多写少 → Redis + 本地缓存，配置变更时用 Pub/Sub 通知各实例失效本地缓存；广告素材（图片/视频）走 CDN，不进 Redis。
3. **高并发手段**：认证接口用 `SETNX auth:lock:{mac}` 防同一设备并发重复提交；按设备/手机号/IP 限流（Redis + Lua 滑动窗口，或 `INCR` + `EXPIRE` 固定窗口）；同一设备的 key 用 hash tag 让它们落在同一分片避免跨槽；网关放行查询走**本地缓存 + Redis 两级**，Redis 挂了降级为「已认证设备继续放行、拒绝新认证」——**保护数据库，而不是把 DB 打挂**。
4. **为什么这层不用 MySQL**：放行判断是**每次上网请求都要查**的超高频读，量级和认证请求差好几个数量级；MySQL 只存认证流水（审计/计费），异步写。

---

# 第三部分 · MySQL vs Redis 与选型

> **一句话**：**MySQL 存「真相」，Redis 存「热度和速度」**——判断标准是「这份数据丢了，会不会造成不可接受的损失」。

| 判断维度 | 放 MySQL | 放 Redis |
| --- | --- | --- |
| 数据角色 | **唯一真相源**（账本、订单、消息、配置） | 缓存 / 派生数据（能重新算出来） |
| 一致性要求 | 强一致、要事务 | 可接受短暂不一致、最终一致 |
| 数据量 | 大（可落盘、可分片） | 小（内存贵，只放热点） |
| 访问模式 | 复杂查询、范围、join、聚合 | 按 key 点查、计数、排行、原子操作、锁 |
| 持久化 | 必须 | 可丢，能重建 |

- **LoveSpouse 的落位**：会话、消息、角色配置、金币流水账本 → MySQL；角色配置缓存、最近会话列表、金币余额、通话锁、限流计数 → Redis。
- **POS 的落位**：流水、商品档案、电子发票 → MySQL；统计汇总、幂等键、热点商品、分布式锁 → Redis。
- **商云前台**：本地 SQLite 是「端侧真相」（离线也要能收银），云端 MySQL 做汇总；用 MQ 同步，冲突以云端为准。

**用 Redis 做计数器的取舍**

> **一句话**：计数能重建就放 Redis，不能重建就必须落 MySQL；关键计数用「Redis 实时 + MySQL 定期落盘 + 启动预热」，并靠对账兜底。

- 好处：`INCR` 原子、O(1)，天然适合计数（未读数、限流、播放量、库存）。
- 风险：① Redis 重启/主从切换会丢最后一段计数（主从是**异步复制**）② 计数是派生数据，必须能和真相源对账。
- 实践原则：**可重建**（未读数可由消息表 count、播放量可接受偏差）→ 放心放 Redis；**不可重建**（余额、库存、订单）→ 必须落 MySQL；库存这类必须准确的 → **Redis 预扣减挡流量 + MySQL 事务扣减守底线**，Redis 扣成功而 MySQL 失败要回滚 Redis（补偿），反之则拒绝请求。

---

# 第四部分 · 高频追问 TOP 20 速答表

| # | 问题 | 一句话答案 |
| --- | --- | --- |
| 1 | B+ 树为什么比 B 树适合做索引 | 非叶子不存数据 → 扇出大 → 树矮（3 层约 2000 万行），叶子成链表，范围查询快 |
| 2 | 什么是回表，怎么避免 | 二级索引只存主键，查非索引列要再走一次聚簇索引；用**覆盖索引**避免 |
| 3 | 最左前缀与联合索引顺序 | 必须从最左列连续匹配；顺序口诀：等值在前、范围在后、排序列跟上、区分度高优先 |
| 4 | 索引失效的常见场景 | 函数/运算、隐式类型转换、前导 `%`、`or` 非索引列、跳过最左列、区分度太低 |
| 5 | EXPLAIN 看哪几个字段 | `type`、`key`、`rows`、`Extra`（filesort / temporary / Using index） |
| 6 | RR 和 RC 的本质区别 | ReadView 生成时机：RC 每次快照读都生成，RR 只在第一次生成；RR 还有间隙锁 |
| 7 | MVCC 怎么实现的 | 隐藏 `trx_id` + undo log 版本链 + ReadView 可见性判断 |
| 8 | 当前读和快照读的区别 | 快照读读 MVCC 版本、不加锁；当前读（`for update` / update / delete）读最新版本并加锁 |
| 9 | 间隙锁与临键锁 | RR 下锁住索引记录之间的间隙防插入；**等值查不存在的记录也会加间隙锁** |
| 10 | 死锁怎么排查、怎么避免 | `show engine innodb status` 看 LATEST DETECTED DEADLOCK；统一加锁顺序、拆小事务、update 走索引 |
| 11 | redo / undo / binlog 的区别 | redo 崩溃恢复（物理、循环写）、undo 回滚 + MVCC（逻辑）、binlog 归档 + 主从（Server 层） |
| 12 | 为什么需要两阶段提交 | 让 redo log 和 binlog 逻辑一致，崩溃恢复时能判断该提交还是回滚 |
| 13 | 深分页怎么优化 | 游标 `where id > last_id`、延迟关联（先拿 id 再回表）、业务禁止深分页 |
| 14 | Go 连接池怎么配 | `MaxOpenConns` 必设（CPU×2~4）、`MaxIdleConns = MaxOpen`、`ConnMaxLifetime` 30min~1h |
| 15 | 分库分表后会带来什么问题 | 分布式 ID、跨库 join、分布式事务、跨库分页排序；**能不分就不分** |
| 16 | 主从延迟怎么处理 | 写后读走主库、关键读强制走主、延迟监控自动切回主库、拆大事务 + 并行复制 |
| 17 | Redis 为什么快 | 内存 + 命令执行单线程免锁 + epoll 多路复用 + 高效数据结构与渐进式 rehash |
| 18 | 缓存穿透 / 击穿 / 雪崩 | 空值 + 布隆 / 互斥锁 + 逻辑过期 / TTL 打散 + 多级缓存 + 限流降级 |
| 19 | 缓存与 DB 一致性怎么做 | 先更新 DB 再删缓存，延迟双删或订阅 binlog，**TTL 兜底**，接受最终一致 |
| 20 | 大 key 和热 key 怎么治 | 大 key 拆 + `unlink` + `scan` 分批；热 key 上本地缓存 + 加随机后缀分散到多分片 |

---

**最后 3 分钟自检**：B+ 树 3 层 2000 万行 / 回表与覆盖索引 / 最左前缀口诀 / type 从好到坏 → RC 与 RR 的差异根源是 ReadView 生成时机 → redo·undo·binlog 三句话 + 两阶段提交三步 → Go 连接池四个参数两个坑 → 缓存三大问题各三个解法 +「TTL 是最后的兜底」→ 分布式锁四要素（NX+PX 原子、value 唯一、Lua 释放、看门狗续期）→ 金币扣减的 Lua 四件事 +「扣减以 Redis 为准、入账以 MySQL 为准」+「不会超扣、只会少扣」。
