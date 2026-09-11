# PostgreSQL 与 SQL｜45 分钟

用时：概念 10 分钟，手写 SQL 20 分钟，慢查询复述 15 分钟。与 BI 模块合计 70 分钟。

## 1. 聚合与窗口函数

`GROUP BY` 将多行收成聚合结果；窗口函数在保留输入行的基础上进行排名、累计或前后比较。窗口结果需要外层查询筛选。[PostgreSQL 窗口函数文档](https://www.postgresql.org/docs/current/tutorial-window.html)

| 工具 | 面试记法 |
|---|---|
| ROW_NUMBER | 连续编号，需稳定排序来打破并列 |
| RANK | 并列占名次，如 1、1、3 |
| DENSE_RANK | 并列不跳号，如 1、1、2 |
| LAG | 取排序后的上一行，不一定是昨天 |
| CTE（WITH） | 分步骤组织 SQL，不保证性能提升 |

## 2. 唯一需要手写的 SQL

题目：每天支付毛收入最高的三个渠道。

假设 `orders(tenant_id, order_id, channel_id, paid_at timestamptz, amount numeric, status)` 一行一笔订单，同币种，不处理退款。`$1` 为租户，`$2/$3` 为明确时区的起止时间点，区间左闭右开。

```sql
WITH daily AS (
  SELECT (paid_at AT TIME ZONE 'Asia/Shanghai')::date AS day,
         channel_id, SUM(amount) AS revenue
  FROM orders
  WHERE tenant_id = $1 AND status = 'paid'
    AND paid_at >= $2 AND paid_at < $3
  GROUP BY 1, 2
), ranked AS (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY day ORDER BY revenue DESC, channel_id
  ) AS rn
  FROM daily
)
SELECT day, channel_id, revenue
FROM ranked
WHERE rn <= 3
ORDER BY day, rn;
```

解释顺序：过滤租户和时间 → 按天/渠道汇总 → 每天独立排名 → 取前三行。

纸面验算：同一天 A=100、B=80、C=80、D=50，返回 A/B/C。若改为前三个收入档位且保留并列，用仅按收入排序的 DENSE_RANK，则四个渠道都返回。

环比追问：补齐日期后用 LAG；用 `(本期-上期)/NULLIF(上期,0)`，避免整数除法。数据未到齐不能补成零。

## 3. 慢查询的排查顺序

1. 固定 SQL、参数、时间范围和数据量，先排除锁等待或连接池排队。
2. 先看 EXPLAIN；在可控环境用 `EXPLAIN (ANALYZE, BUFFERS)` 获取实际信息，它会执行语句。
3. 找主要扫描、关联和排序成本，比较估算/实际行数，观察 loops、缓冲读取及排序落盘。
4. 修改 SQL、索引或统计信息后，在相近条件下对比正确性与耗时。[执行计划文档](https://www.postgresql.org/docs/current/using-explain.html)

不要看到 Seq Scan 就认定有问题，大比例读取时顺序扫描可能更便宜；也不要只看一次缓存命中后的最快结果。

### 实操：如何检查连接池排队与锁等待

两处等待不同：**连接池排队发生在应用拿连接时；锁等待发生在 SQL 已到数据库之后。** 以下按 Go `database/sql` + PostgreSQL，排障时按需阅读。

**第一步：在 Go 服务中看连接池指标。** `db` 是业务实际使用的 `*sql.DB`，不要新建一个池来检查。

```go
s := db.Stats()
log.Printf("max=%d open=%d inUse=%d idle=%d waits=%d waitTime=%s",
    s.MaxOpenConnections, s.OpenConnections, s.InUse, s.Idle,
    s.WaitCount, s.WaitDuration)
```

在问题发生期间每隔几秒采样：当 `InUse` 持续达到非零上限、`Idle=0`，且 `WaitCount/WaitDuration` 持续增加，就有连接池排队证据。后两者是累计值，要看采样差值；`WaitCount` 不是当前排队人数。[Go DBStats](https://pkg.go.dev/database/sql#DBStats)

检查 Rows 是否及时 Close、事务是否 Commit/Rollback、是否有慢 SQL/长事务长期占用连接。先修复占用原因，再结合数据库容量调整池大小。若使用原生 pgxpool，应查看该池的 Stat；若中间还有 PgBouncer，也要查代理池。

**第二步：在 psql 或数据库客户端查询当前会话。** 应在卡顿期间执行，使用能查看目标会话的监控账号。

```sql
SELECT pid, application_name, state,
       wait_event_type, wait_event,
       clock_timestamp() - query_start AS query_age,
       clock_timestamp() - xact_start AS transaction_age,
       pg_blocking_pids(pid) AS blocking_pids,
       LEFT(query, 200) AS query
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid <> pg_backend_pid()
  AND state <> 'idle'
ORDER BY query_start;
```

`wait_event_type='Lock'` 表示当前等待锁；`state='active'` 也可能在等待。`idle in transaction` 表示事务未结束、当前等待客户端，可能持锁。`query_age` 是语句开始以来的时间，不是锁等待时长；非 active 会话的 query 是上一条语句。[会话状态说明](https://www.postgresql.org/docs/current/monitoring-stats.html)

**第三步：找到谁阻塞了它。** 如 blocking_pids 返回 `{12345}`：

```sql
SELECT pid, application_name, usename, state,
       xact_start, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE pid = 12345; -- 替换为实际阻塞者 PID
```

`pg_blocking_pids` 返回阻塞该会话获取锁的进程 ID。[函数说明](https://www.postgresql.org/docs/current/functions-info.html)

结合 application_name 和应用日志定位事务，检查是否更新后未提交、事务中等待外部接口，或 DDL 与业务冲突。优先让所属业务正常结束事务，再修复事务范围；不要根据一张截图直接终止会话。

**判断边界**：一次采样没有锁等待，不代表之前没等过；池指标也只覆盖当前进程的当前池。应结合故障时间连续采样。锁等待可能长期占住连接，继而导致连接池排队，两者可以同时发生。

## 4. 索引、预聚合与分页

常用“租户等值 + 支付时间范围”时，候选索引为 `(tenant_id, paid_at)`。如果渠道等值筛选也很常见，再评估 `(tenant_id, channel_id, paid_at)`。选择要结合真实查询，索引也增加写入与存储成本。

大量历史聚合可考虑日汇总表，但去重人数不一定能跨天相加，退款和迟到数据也需要重算机制。

明细深分页可使用稳定排序的游标，例如时间加唯一 ID；大导出改异步分批，并约定数据截止时间或一致性策略。

## 自测

不看文档写出上述 SQL；再用一分钟解释：为什么先聚合？同额渠道怎么办？索引为什么这样排？慢查询首先看什么？

SQL 为学习示例，本次未连接数据库执行。面试短答见[面试问答](面试问答.md)第 1—6 题，无需重复通读。
