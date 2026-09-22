# 《一条 SQL 的一生》技术解读稿（W1 内核 · 准、可核、附官方来源）

稿件目标：占「技术解读 30%」。**每条关键论断附官方来源 URL**；不确定的数字不写。
演示语句（原型同步）：`SELECT name FROM users WHERE id = 42;`

---

## 0. 总览：一次查询要穿过四层

一条 SQL 从客户端发出到拿回结果，要经过 **客户端 → TiDB Server（计算层）→ PD（调度/授时）→ TiKV（存储层）→ 回到客户端**。

TiDB Server 这一层是**无状态**的：

> "The nodes at this layer are stateless. These nodes themselves do not store data and are completely equivalent."
> — https://docs.pingcap.com/tidb/stable/tidb-computing

它"receives SQL requests, performs SQL parsing and optimization, and ultimately generates a distributed execution plan."
— https://docs.pingcap.com/tidb/stable/tidb-architecture

---

## 1. 解析（Parser）：把文本变成语法树

TiDB Server 收到 **MySQL 协议包**，做词法、语法、语义解析，然后生成并优化执行计划：

> "TiDB Server will parse MySQL Protocol Packet … parse the SQL request syntactically and semantically, develop and optimize query plans, execute a query plan, get and process the data."
> — https://docs.pingcap.com/tidb/stable/tidb-computing

阶段命名：`parser` → 逻辑优化 → 物理优化。
— https://docs.pingcap.com/tidb/stable/sql-optimization-concepts

---

## 2. 优化（Optimizer）：基于代价选计划（CBO）

优化器用**统计信息**估每一步的行数，并算出各候选计划的**代价**，取总体代价最低者：

> "TiDB uses statistics as input to the optimizer to estimate the number of rows processed in each plan step … produces a cost for each available plan. The optimizer then picks the execution plan with the lowest overall cost."
> — https://docs.pingcap.com/tidb/stable/statistics

统计信息构成：**等深直方图（histogram）**、**Count-Min Sketch**、**Top-N**；默认 Top-N 为 `20`（最大 1024）、CMSketch depth `5`、width `2048`。由 `ANALYZE TABLE` 收集；当变更行比例超过 `tidb_auto_analyze_ratio`（默认 `0.5`）时自动更新。
— https://docs.pingcap.com/tidb/stable/statistics

`EXPLAIN` 里的 `estRows` 就是这份估计；若出现 `stats:pseudo` 则估计可能不准。
— https://docs.pingcap.com/tidb/stable/explain-overview

> 原型里我给 `id = 42` 走的是**点查（Point_Get）**，因为 `id` 是主键——优化器能直接定位，无需全表扫。

---

## 3. 下推（Coprocessor）：把计算送到数据身边

> "Coprocessor is a coprocessing mechanism that shares the computation workload with TiDB. It is located in the storage layer (TiKV or TiFlash) and collaboratively processes computations pushed down from TiDB on a per-Region basis."
> — https://docs.pingcap.com/tidb/stable/glossary

举例：谓词 `name = "TiDB"` 会被下推到存储层；聚合 `Count(*)` 也能下推做**预聚合**：
> "the SQL predicate condition `name = "TiDB"` should be pushed down to the storage node … the aggregation function `Count(*)` can also be pushed down to the storage nodes for pre-aggregation."
> — https://docs.pingcap.com/tidb/stable/tidb-computing

`EXPLAIN` 里 **`cop[tikv]`** = 在 TiKV coprocessor 里算；**`root`** = 回 TiDB 里算：
> "A `cop[tikv]` task indicates that the operator is performed inside the TiKV coprocessor. A `root` task indicates that it will be completed inside of TiDB."
> — https://docs.pingcap.com/tidb/stable/explain-overview

优化目标之一：**尽量把计算下推到 TiKV**，减少传输、卸载单点压力。
— https://docs.pingcap.com/tidb/stable/expressions-pushed-down

---

## 4. 数据怎么放：Region、Raft 与 RocksDB

### 4.1 Region 是调度的最小单元
> "Each segment is called a Region. Each Region can be described by `[StartKey, EndKey)`, a left-closed and right-open interval. The default size limit for each Region is 256 MiB and the size can be configured."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

> "Region is the minimal piece of data storage in TiKV, each representing a range of data (256 MiB by default). Each Region has three replicas by default."
> — https://docs.pingcap.com/tidb/stable/glossary

Region 不是一开始就切好的，而是**随写入逐步分裂**：
> "A region in a TiKV cluster is not divided at the beginning but is gradually split as data is written to it … generate new Regions through splitting existing ones every time the size of the Region or the number of keys has reached a threshold."
> — https://docs.pingcap.com/tidb/stable/glossary

> ⚠️ 注意单位：官方写的是 **MiB（256）**，不是 MB。

### 4.2 三副本 + Raft：读写都走 Leader
> "Multiple Replicas of a Region are stored on different nodes to form a Raft Group … One of the Replicas serves as the Leader of the Group and other as the Follower. By default, all reads and writes are processed through the Leader."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

> "In all replicas, a leader is responsible for reading and writing, and followers are responsible for replicating Raft logs from the leader."
> — https://docs.pingcap.com/tidb/stable/tidb-scheduling

写只需复制到多数派：
> "successful writes only need that data is replicated to the majority of nodes."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

### 4.3 行数据在 TiKV 里长这样
TiKV 不直接写盘，先写 **RocksDB**（可理解为一个持久化有序 KV Map）：
> "TiKV does not write data directly on the disk, but stores data in RocksDB … simply consider RocksDB as a single persistent Key-Value Map."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

行编码：`Key: tablePrefix{TableID}_recordPrefixSep{RowID}`，`Value: [col1, col2, col3, col4]`；常量 `tablePrefix='t'`、`recordPrefixSep='r'`、`indexPrefixSep='i'`。唯一索引 `Key: …_indexPrefixSep{IndexID}_indexedColumnsValue`。全表按 `RowID` 有序排列。
— https://docs.pingcap.com/tidb/stable/tidb-computing

---

## 5. PD：大脑与授时

> "The PD server is the metadata managing component of the entire cluster … allocates transaction IDs to distributed transactions … 'the brain' of the entire TiDB cluster."
> — https://docs.pingcap.com/tidb/stable/tidb-architecture

> "it requires a global timing service, Timestamp Oracle (TSO), to assign a monotonically increasing timestamp. In TiKV, such a feature is provided by PD."
> — https://docs.pingcap.com/tidb/stable/glossary

TSO = 物理时间戳（Unix 毫秒）+ 逻辑计数器，拼成一个**单调递增**的全局时间戳。
— https://docs.pingcap.com/tidb/stable/tso

---

## 6. 事务：Percolator 与两阶段提交（2PC）

TiKV 事务模型来自 Google Percolator：
> "Transaction of TiKV adopts the model used by Google in BigTable: Percolator."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

乐观事务用 2PC：
> "To support distributed transactions, TiDB adopts two-phase commit (2PC) in optimistic transactions."
> — https://docs.pingcap.com/tidb/stable/optimistic-transaction

2PC 里 `start_ts` 从 PD 取，"monotonically increasing in time and globally unique … also serves as the version of the database snapshot"；先 **prewrite**（TiKV 查冲突/过期版本，给合规数据**加锁**），再取 `commit_ts`，向主键所在 TiKV 发起 **commit** 并清理 prewrite 阶段留下的锁。
— https://docs.pingcap.com/tidb/stable/optimistic-transaction

悲观事务在 2PC 前多一个 **Acquire Pessimistic Lock** 阶段：
— https://docs.pingcap.com/tidb/stable/pessimistic-transaction

> "Starting from TiDB 3.0.8, TiDB uses the pessimistic transaction mode by default."
> — https://docs.pingcap.com/tidb/stable/transaction-overview

默认隔离级别为 **Repeatable Read**（与 MySQL 相同）：
— https://docs.pingcap.com/tidb/stable/pessimistic-transaction

---

## 7. MVCC：同一把 Key，多个版本

> "TiKV supports multi-version concurrency control (MVCC) … TiKV MVCC is implemented by appending a version number to the key."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

> "for multiple versions of the same Key, versions with larger numbers are placed first … you can directly locate the first position greater than or equal to this `Key_Version` through RocksDB's `SeekPrefix(Key_Version)` API."
> — https://docs.pingcap.com/tidb/stable/tidb-storage

快照读 / 当前读：
> "Snapshot read: it is an unlocked read that reads a version committed before the transaction starts … Current read: it is a locked read that reads the latest committed version."
> — https://docs.pingcap.com/tidb/stable/pessimistic-transaction

---

## 8. 落到本次演示语句

`SELECT name FROM users WHERE id = 42;` 的一生：

1. **客户端** 发 MySQL 协议包。
2. **TiDB Server** 解析 → CBO 选计划。`id` 是主键 ⇒ **Point_Get**，`estRows=1`。
3. **PD** 授 `start_ts`（快照读时间点）。
4. 按 `id=42` 计算 key，**按 `[StartKey, EndKey)` 定位 Region**，找到该 Region 的 **Leader**（Raft group 三副本之一）。
5. **Coprocessor 下推**：点查/谓词在 TiKV 侧完成（`cop[tikv]`），只回传结果。
6. **TiKV** 在 RocksDB 里 `SeekPrefix` 到 `<key, start_ts>`，做 **MVCC** 版本匹配，取值。
7. **返回**，TiDB 侧（`root`）合并/组装结果集，回客户端。

> 若换成写语句（`UPDATE`），则在第 6 步前后插入 **prewrite → commit** 的 2PC，并涉及 **Raft 日志复制到多数派**。

---

## 附：来源清单（全部为 PingCAP 官方文档）
- https://docs.pingcap.com/tidb/stable/tidb-computing
- https://docs.pingcap.com/tidb/stable/tidb-architecture
- https://docs.pingcap.com/tidb/stable/tidb-storage
- https://docs.pingcap.com/tidb/stable/tidb-scheduling
- https://docs.pingcap.com/tidb/stable/glossary
- https://docs.pingcap.com/tidb/stable/sql-optimization-concepts
- https://docs.pingcap.com/tidb/stable/statistics
- https://docs.pingcap.com/tidb/stable/explain-overview
- https://docs.pingcap.com/tidb/stable/expressions-pushed-down
- https://docs.pingcap.com/tidb/stable/transaction-overview
- https://docs.pingcap.com/tidb/stable/optimistic-transaction
- https://docs.pingcap.com/tidb/stable/pessimistic-transaction
- https://docs.pingcap.com/tidb/stable/tso

> 未核实项注明为 ⚠️；未采信的路径（如 `/tidb-transaction`、`/optimizer-overview` 返回 404）不引用。
