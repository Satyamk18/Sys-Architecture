# PostgreSQL Performance Engineering: MVCC, Concurrency, Query Execution, and Scaling

> Structured study notes from our PostgreSQL deep dive. These notes capture the concepts covered so far, the questions and misconceptions we resolved, practical SQL/diagnostic commands, and the current learning boundary. The next modules will deepen memory, storage/WAL, and lock internals before the final high-throughput incident exercise.

## 1. Learning map

We are approaching PostgreSQL performance through four resource dimensions:

| Dimension | What we investigate | Study status |
|---|---|---|
| CPU | Query plans, scans, joins, cardinality estimates, aggregation, parallel execution | **Started and covered at a foundational level** |
| Memory | `shared_buffers`, OS page cache, `work_mem`, memory pressure, sort/hash spills | Roadmap established; detailed module pending |
| Storage | WAL, commit durability, `fsync`, checkpoints, NVMe latency/IOPS, write amplification | Roadmap established; detailed module pending |
| Locks and concurrency | MVCC, row locks, deadlocks, hot rows, isolation, contention | MVCC and isolation foundations covered; deeper lock diagnosis pending |

The guiding principle is:

**A database is not slow because it lacks hardware by default. It is slow because some part of its execution path, resource budget, concurrency model, or workload design has become the limiting factor.**

We should identify the bottleneck with evidence before changing configuration or adding infrastructure.

---

## 2. PostgreSQL MVCC: how concurrent reads and writes work

### 2.1 What MVCC means

MVCC stands for **Multi-Version Concurrency Control**. PostgreSQL does not generally overwrite a row in place every time it is updated. An update creates a new row version (tuple), while the previous version remains until it is no longer needed by any active transaction snapshot and can be reclaimed.

This lets readers work with a consistent view of data without taking ordinary read locks that block writers.

Conceptual example:

Initial row:

| id | balance | version |
|---:|---:|---|
| 7 | 500 | V1 |

Transaction T1 updates the balance to 450:

| id | balance | visibility |
|---:|---:|---|
| 7 | 500 | Old tuple version |
| 7 | 450 | New tuple version |

A transaction whose snapshot predates the update may continue to see 500. A transaction whose snapshot includes the update sees 450.

The database decides tuple visibility using transaction IDs and the snapshot, not merely by choosing the physically newest row.

### 2.2 The role of `xmin` and `xmax`

PostgreSQL tuple headers contain transaction metadata. Two important fields are:

- `xmin`: transaction ID associated with the insertion of this tuple version.
- `xmax`: transaction ID associated with deleting the tuple or superseding it through an update (with additional meanings in some lock-related cases).

A simplified mental model:

- A tuple is potentially visible if its inserting transaction committed and is visible to the current snapshot.
- A tuple may be invisible if its deleting/updating transaction committed and is visible to the current snapshot.
- In-progress or aborted transactions require additional visibility checks.

Inspecting these fields can help learning and debugging, but they are implementation details. Do not build application logic around raw transaction IDs.

Example inspection:

```sql
SELECT xmin, xmax, *
FROM accounts
WHERE id = 7;
```

### 2.3 Why old row versions remain

An old tuple cannot be removed while an active transaction may still need to see it. For example, a long-running transaction with an older snapshot can prevent cleanup of tuples that would otherwise be reclaimable.

When obsolete tuple versions accumulate, they are commonly called **dead tuples**.

Consequences can include:

- More table and index pages to scan.
- Increased disk usage and cache pressure.
- More work for vacuum.
- Potentially worse query latency and throughput.

### 2.4 VACUUM and dead-tuple cleanup

PostgreSQL's VACUUM process identifies tuple versions that are no longer visible to any relevant transaction and makes their space reusable.

Important distinctions:

- Ordinary `VACUUM` makes space reusable inside the relation; it generally does not return all that space to the operating system.
- `VACUUM FULL` rewrites the table and can return space to the OS, but requires a stronger lock and additional I/O.
- Autovacuum runs maintenance automatically based on thresholds and settings.
- Long-running transactions and old snapshots can delay cleanup.

Useful checks:

```sql
-- Approximate dead-tuple and live-tuple counts
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

These statistics are estimates, not exact real-time counts.

### 2.5 HOT updates

A **Heap-Only Tuple (HOT) update** can avoid creating new index entries when the update does not modify columns referenced by indexes and the new tuple version can fit on the same heap page.

Why HOT matters:

- Fewer index modifications.
- Less index bloat and write amplification.
- Lower update cost.

HOT is not guaranteed. Indexing frequently updated columns can make HOT updates less likely. Page free space also matters.

---

## 3. MVCC is not the same thing as optimistic locking

This was an important conceptual distinction.

### MVCC answers:

**Which version of a row should this transaction see?**

It is a database concurrency and visibility mechanism.

### Optimistic concurrency control answers:

**Can I commit my change if the data has not changed since I read it?**

A common application-level optimistic pattern is:

```sql
UPDATE accounts
SET balance = 450,
    version = version + 1
WHERE id = 7
  AND version = 12;
```

The application checks the affected-row count. If zero rows were updated, another transaction may have changed the row and the operation must be retried or rejected.

### Pessimistic locking

A pessimistic approach acquires a lock before performing the critical update:

```sql
BEGIN;

SELECT *
FROM accounts
WHERE id = 7
FOR UPDATE;

UPDATE accounts
SET balance = 450
WHERE id = 7;

COMMIT;
```

The row lock prevents conflicting updates/deletes/locking reads from proceeding until the transaction ends.

### The important conclusion

PostgreSQL uses MVCC for row-version visibility and concurrency between readers and writers. It also uses locks to coordinate conflicting writes. MVCC itself is not synonymous with optimistic locking.

---

## 4. `SELECT ... FOR UPDATE`: what it actually does

Example:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 7
FOR UPDATE;

-- Perform business validation and update
UPDATE accounts
SET balance = balance - 100
WHERE id = 7;

COMMIT;
```

The query obtains a row-level lock on the selected row. Another transaction attempting a conflicting update or row lock on that row must wait until the lock holder commits or rolls back.

Key points:

1. The lock is held until transaction end, not merely until the SELECT statement ends.
2. It does not lock the entire table in the ordinary case.
3. It does not mean all readers are blocked. Ordinary MVCC reads can generally continue to see an appropriate row version.
4. Keep the transaction short. Do not hold a row lock while making network calls or waiting for user input.
5. A locking query can wait if another transaction already holds a conflicting lock.

### A useful transfer scenario

Two requests attempt to transfer money from the same account.

Without coordination, both could read the same balance and independently make decisions based on stale application state.

With `SELECT ... FOR UPDATE`, one transaction locks the account row. The second transaction waits, then proceeds after the first transaction ends and observes the row according to its transaction isolation semantics.

For transfers involving two accounts, acquire locks in a deterministic order (for example, ascending account ID) to reduce deadlock risk.

---

## 5. PostgreSQL transaction isolation: READ COMMITTED vs REPEATABLE READ

An MVCC snapshot determines which committed row versions a transaction can see.

### 5.1 READ COMMITTED

This is PostgreSQL's default isolation level.

- Each statement gets a new snapshot.
- A later SELECT in the same transaction can see changes committed by other transactions after the earlier statement began.
- Two SELECT statements in one transaction can therefore return different results.

### 5.2 REPEATABLE READ

In PostgreSQL:

- The transaction uses a stable snapshot established by its first non-transaction-control statement.
- Repeated reads see a consistent view of committed data from that snapshot.
- PostgreSQL's REPEATABLE READ prevents phantom reads in the SQL-standard sense through its snapshot behavior.
- A transaction attempting to update a row changed by a concurrent transaction after its snapshot may receive a serialization-style failure and need to retry.

Example:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT balance
FROM accounts
WHERE id = 7;

-- Another transaction updates and commits this row.

SELECT balance
FROM accounts
WHERE id = 7;

COMMIT;
```

The second SELECT continues to use the transaction's established snapshot.

### 5.3 SERIALIZABLE

SERIALIZABLE aims to make concurrent transactions behave as if they had executed in some serial order. PostgreSQL uses Serializable Snapshot Isolation (SSI) to detect dangerous dependency patterns. Transactions can be aborted with serialization failures and must be retried by the application.

### Isolation-level summary

| Isolation level | Snapshot behavior in PostgreSQL | Important consequence |
|---|---|---|
| READ COMMITTED | New snapshot per statement | Later statements can see newer commits |
| REPEATABLE READ | Stable transaction snapshot | Consistent repeated reads; concurrent update conflicts can abort |
| SERIALIZABLE | Snapshot isolation plus SSI conflict detection | Strongest serializable behavior; retry serialization failures |

Do not confuse a stable read snapshot with a guarantee that every concurrent write will succeed.

---

## 6. Scaling PostgreSQL writes before sharding

The question explored was: **How would we scale PostgreSQL to millions of writes before introducing sharding?**

The correct sequence is to remove avoidable work, understand the workload, and scale a single primary as far as the workload and consistency requirements allow.

### 6.1 Optimize the write path

- Remove unnecessary indexes. Every index adds maintenance work on inserts, updates, and deletes.
- Keep transactions short.
- Avoid updating rows when values have not actually changed.
- Batch inserts/updates where semantics allow.
- Use `COPY` for high-volume bulk ingestion.
- Avoid repeated round trips and chatty application-to-database patterns.
- Make sure queries use appropriate indexes and avoid accidental full-table work.

### 6.2 Batch writes

Instead of sending many individual statements:

```sql
INSERT INTO events (event_id, payload)
VALUES (1, 'a');

INSERT INTO events (event_id, payload)
VALUES (2, 'b');
```

Use a multi-row insert or bulk-loading path:

```sql
INSERT INTO events (event_id, payload)
VALUES
    (1, 'a'),
    (2, 'b'),
    (3, 'c');
```

Batching reduces per-statement overhead and round trips. Batch size must be measured because oversized transactions can increase lock duration, memory use, WAL bursts, and rollback cost.

### 6.3 Asynchronous ingestion and backpressure

For workloads where the business contract permits asynchronous processing:

```text
Producers
   |
   v
Message broker / durable queue
   |
   v
Bounded consumer workers
   |
   v
Batching and validation
   |
   v
PostgreSQL primary
```

Benefits:

- Absorbs short traffic bursts.
- Decouples producer latency from database write latency.
- Allows batching and controlled concurrency.
- Makes backpressure explicit.

The queue does not increase the database's sustainable write capacity by itself. It protects the database from overload and smooths bursts. Define retry, idempotency, ordering, dead-letter, and durability behavior.

### 6.4 Connection management

Do not create an unbounded number of database connections. Each PostgreSQL backend consumes resources and adds scheduling/context-switching overhead.

A pooler such as PgBouncer can reduce connection churn and cap the number of active server connections. Pool sizing must reflect workload, query duration, CPU capacity, and transaction behavior.

### 6.5 Partitioning is not sharding

Partitioning divides a logical table into smaller pieces within one PostgreSQL database/cluster. It can help with:

- Partition pruning for queries with partition-key predicates.
- Lifecycle operations such as dropping old time partitions.
- Managing very large tables and indexes.

Partitioning does **not** automatically distribute writes across multiple independent database servers. A partitioned table on one primary still shares that primary's CPU, memory, storage, and WAL pipeline.

Sharding distributes data across multiple independent database nodes and introduces routing, cross-shard query, transaction, and operational complexity.

### 6.6 Replicas and caching do not directly scale primary writes

- Read replicas can offload eligible read traffic.
- Caches can reduce repeated reads.
- CQRS/read models can separate read-heavy access patterns from the write model.

These techniques can free resources on the primary, but they do not directly increase the write capacity of a single primary. Replication itself also consumes resources and produces lag considerations.

### 6.7 Find hot rows and hot keys

A database can have spare aggregate CPU and storage capacity yet be limited by a small number of highly contended rows.

Examples:

- One global counter updated by every request.
- One inventory row receiving all purchases.
- One account row receiving a high volume of balance updates.

Possible redesigns include sharded counters, append-only ledgers, batching/aggregation, partitioned ownership, or asynchronous aggregation. The right choice depends on consistency and correctness requirements.

---

## 7. Performance diagnosis: CPU, memory, storage, or locks?

A high-level diagnosis starts with symptoms and then validates them with database and operating-system evidence.

### 7.1 CPU-bound workload

Possible signals:

- CPU utilization remains high.
- Query latency rises with CPU saturation.
- A small number of expensive queries dominate total database time.
- Disk latency may remain healthy.
- Query plans show excessive row processing, expensive joins, sorts, or aggregations.

First tools:

```sql
-- Requires pg_stat_statements to be installed and configured.
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Then inspect important queries:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

### 7.2 Memory-bound workload

Possible signals:

- Sorts or hash operations spill to temporary files.
- Temporary file I/O increases.
- OS begins swapping or memory pressure rises.
- Cache misses increase and queries perform more physical reads.

Relevant PostgreSQL settings and concepts:

- `shared_buffers`: PostgreSQL's shared buffer cache.
- OS page cache: the operating system also caches filesystem pages.
- `work_mem`: memory budget used by individual query operations such as sorts and hash operations, not one fixed allocation per connection.

**Important:** Increasing `work_mem` globally can cause memory exhaustion. One query may have multiple memory-consuming operations, and many queries can run concurrently. Memory budgeting must account for concurrency and plan shape.

### 7.3 Storage-bound workload

Possible signals:

- High storage latency or saturated IOPS/throughput.
- WAL flush or checkpoint pressure.
- Increased read/write latency.
- Temp-file I/O from spilled operations.
- Query execution waits on I/O rather than CPU.

For write-heavy workloads, understand the WAL and commit path. PostgreSQL must preserve durability guarantees: with normal synchronous commit behavior, commit acknowledgement waits for the required WAL durability step. Do not treat disabling `fsync` as a production performance fix.

Useful views vary by PostgreSQL version:

- `pg_stat_wal` for WAL statistics.
- `pg_stat_io` (available in PostgreSQL 16+) for I/O statistics.
- `pg_stat_bgwriter` and checkpoint-related statistics, with view columns changing across versions.
- OS-level disk latency, queue depth, IOPS, and throughput metrics.

### 7.4 Lock/contention-bound workload

Possible signals:

- Many sessions are waiting.
- Throughput stops increasing as concurrency rises.
- A small set of rows or relations is frequently locked.
- Transactions have long durations.
- Deadlocks or lock wait time increase.

Useful SQL:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query,
    pg_blocking_pids(pid) AS blocking_pids
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

Inspect lock records:

```sql
SELECT *
FROM pg_locks;
```

Typical fixes:

- Shorten transactions.
- Avoid network calls while holding locks.
- Lock rows in a deterministic order.
- Reduce hot-row updates.
- Use atomic SQL updates where appropriate.
- Introduce batching or ownership partitioning where business semantics allow.

### 7.5 Diagnostic matrix

| Suspected bottleneck | Evidence to collect | Typical first investigation |
|---|---|---|
| CPU | CPU utilization, query execution time, rows processed | `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)` |
| Memory | RSS, swap, temp files, sort/hash spills, cache behavior | `work_mem`, temp-file logs, OS memory metrics |
| Storage | Read/write latency, IOPS, WAL flush, checkpoint activity | `pg_stat_wal`, `pg_stat_io`, OS disk metrics |
| Locks | Lock waits, blockers, long transactions, deadlocks | `pg_stat_activity`, `pg_blocking_pids()`, `pg_locks` |

No single metric proves a bottleneck. Correlate database wait events, query statistics, and operating-system metrics over the same time window.

---

## 8. CPU and query execution: how PostgreSQL runs a query

This is the first resource module we started in detail.

### 8.1 Query lifecycle

A simplified PostgreSQL query path:

```text
SQL statement
    |
    v
Parser
    |
    v
Analyzer / semantic analysis
    |
    v
Rewriter
    |
    v
Planner / optimizer
    |
    v
Execution plan
    |
    v
Executor
    |
    v
Rows returned or data modified
```

- **Parser:** checks SQL syntax and builds a parse representation.
- **Analyzer:** resolves tables, columns, types, functions, and semantics.
- **Rewriter:** applies rewrite rules and expands relevant constructs such as views.
- **Planner/optimizer:** estimates alternative execution strategies and chooses a plan based on estimated cost.
- **Executor:** runs the chosen plan.

The planner does not simply choose the plan with the fewest visible steps. It uses estimates and cost parameters to compare alternatives.

### 8.2 Sequential scan

A sequential scan reads table pages and checks rows against the filter.

```sql
SELECT *
FROM transactions
WHERE transaction_type = 'DEBIT';
```

A sequential scan can be the correct plan when:

- The table is small.
- A large fraction of rows qualifies.
- The query needs many columns from matching rows.
- Index access would cause many scattered heap fetches.
- Statistics or cost settings make a sequential scan cheaper.

**Misconception resolved:** An index is not always faster. If a query returns a large proportion of a table, walking an index and fetching heap rows can cost more than reading the table sequentially.

### 8.3 Index scan and bitmap heap scan

An index scan traverses an index to find matching tuple locations, then may visit the heap to retrieve the requested columns and verify visibility.

A bitmap heap scan can collect matching tuple locations into a bitmap, then visit heap pages in a more page-oriented way. This can be useful when many rows match but an index can still narrow the candidate set.

In PostgreSQL, ordinary tables are heap-organized. A secondary index generally points to heap tuple locations rather than storing the full row as the table's primary storage.

### 8.4 Selectivity

Selectivity describes the fraction of rows expected to satisfy a predicate.

```text
selectivity = qualifying rows / total rows
```

Example:

- Table has 100 million rows.
- Predicate matches 100,000 rows.
- Selectivity = 0.1%.

A selective predicate is often a good candidate for index access, but the planner also considers table size, row width, cache state, correlation, and the cost of fetching heap pages.

### 8.5 Join algorithms

PostgreSQL commonly uses three major join strategies.

#### Nested Loop Join

For each row from the outer input, search the inner input.

Conceptually:

```text
for each outer row:
    find matching inner row(s)
```

Often effective when the outer input is small and the inner side has a suitable index. It can become very expensive when the outer side is much larger than estimated.

#### Hash Join

1. Build a hash table from one input.
2. Probe it using rows from the other input.

Often useful for equi-joins when a hash table fits efficiently in memory. Large hash operations can require batching or spill behavior.

#### Merge Join

Both inputs are ordered on join keys, then advanced in key order to find matches.

Often useful when inputs are already suitably ordered or can be sorted at acceptable cost.

| Join type | Core idea | Often attractive when |
|---|---|---|
| Nested Loop | Repeatedly search inner input for each outer row | Outer side is small; inner lookup is efficient |
| Hash Join | Build hash table, then probe | Equi-join with suitable memory/workload |
| Merge Join | Walk two ordered inputs together | Inputs are already ordered or sorting is worthwhile |

### 8.6 Cardinality estimation and statistics

Cardinality estimation predicts how many rows an operation will produce.

The planner relies on statistics collected by `ANALYZE` and autovacuum's analyze activity.

If estimates are wrong, the planner may choose a poor join order or join algorithm.

Example:

- Estimated rows: 20.
- Actual rows: 500,000.
- A Nested Loop selected under the assumption of a tiny outer input may perform hundreds of thousands of repeated lookups.

Inspect a plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Look for:

- Estimated rows vs actual rows.
- Loops.
- Time spent in each node.
- Shared buffer hits and reads.
- Rows removed by filter.
- Sort or hash behavior.

Refresh statistics when appropriate:

```sql
ANALYZE transactions;
```

For correlated columns, ordinary single-column statistics may not capture the relationship. PostgreSQL extended statistics can help:

```sql
CREATE STATISTICS stats_transactions_type_region
    (dependencies, ndistinct, mcv)
ON transaction_type, region
FROM transactions;

ANALYZE transactions;
```

### 8.7 Aggregation, sorting, and CPU cost

Consider:

```sql
SELECT account_id, SUM(amount)
FROM transactions
WHERE transaction_type = 'DEBIT'
GROUP BY account_id
ORDER BY SUM(amount) DESC;
```

Possible plan stages include:

1. Scan rows and apply the filter.
2. Aggregate rows by `account_id`.
3. Sort the aggregated results by the computed sum.

A HashAggregate uses hashing to group rows. A sort-based aggregate uses ordered input. The planner chooses based on estimated cost and available plan options.

If the query filters 15 million rows, the scan and aggregation can consume substantial CPU even when storage latency is low.

An index may reduce work only if it meaningfully narrows or covers the needed access pattern. An index on `transaction_type` is not automatically beneficial if DEBIT represents a large fraction of the table. A composite or partial index may help a specific workload, but must be tested against write overhead and actual query shape.

### 8.8 Parallel query execution

PostgreSQL can use parallel workers for eligible operations. A leader process coordinates work while workers process portions of the plan.

Parallelism can improve throughput for large scans, joins, or aggregations when the operation is parallel-safe and worker resources are available.

It is not free:

- Worker processes consume CPU and memory.
- Coordination and tuple transfer add overhead.
- Small queries may be slower when parallelized.
- Too much concurrency can oversubscribe CPU and increase context switching.

The main benefit is dividing eligible query work across multiple processes, not making every query automatically faster.

### 8.9 Planner cost is not elapsed time

The planner's cost values are estimates in arbitrary units derived from configured cost parameters. They are useful for comparing plan alternatives, not direct milliseconds.

Use `EXPLAIN ANALYZE` to measure actual execution behavior. Remember that `EXPLAIN ANALYZE` executes the statement. For data-modifying statements, use a safe test environment or a transaction that can be rolled back where appropriate.

---

## 9. CPU lab scenario discussed

We used this illustrative workload:

- Table: 100 million transactions.
- Query filters `transaction_type = 'DEBIT'`, groups by `account_id`, and sorts by the aggregate.
- CPU utilization: approximately 96%.
- Average query latency: approximately 4.2 seconds.
- Memory: described as healthy.
- Disk latency: described as low.
- Plan shape: Sort -> HashAggregate -> Seq Scan.
- Filter removes approximately 15 million rows.

The purpose was to reason about why the database may be CPU-bound even when storage and memory appear healthy.

Investigation sequence:

1. Confirm the query is a major contributor using `pg_stat_statements`.
2. Run `EXPLAIN (ANALYZE, BUFFERS)` and inspect actual vs estimated rows.
3. Determine how many rows qualify and how much work the aggregate performs.
4. Check whether a sequential scan is rational for the predicate's selectivity.
5. Evaluate whether a partial/composite index or partition pruning can reduce scanned rows for this query pattern.
6. Inspect whether aggregation or sorting dominates.
7. Evaluate parallel query eligibility and actual worker usage.
8. Measure the impact of any change on both reads and writes, because indexes add write-maintenance cost.

Do not jump straight to “add an index” or “increase parallel workers.” First identify which plan node consumes the time and why.

---

## 10. Practical diagnostic command sheet

### Active sessions and wait events

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS query_age,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

### Blocking sessions

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Table activity and dead tuples

```sql
SELECT
    relname,
    seq_scan,
    idx_scan,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

### Expensive normalized queries

```sql
SELECT
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Query plan

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT ...;
```

### Important operational note

Some statistics views and columns vary across PostgreSQL major versions. Check the documentation for the deployed version before automating dashboards or incident runbooks. `pg_stat_statements` must be installed and configured.

---

## 11. Misconceptions and clarifications we resolved

| Initial question or possible misconception | Correct mental model |
|---|---|
| Is MVCC optimistic locking? | No. MVCC provides row versions and snapshot visibility. Optimistic concurrency control is a conflict-detection strategy, often implemented with a version column or conditional update. |
| Does PostgreSQL overwrite a row on UPDATE? | An UPDATE normally creates a new tuple version; the old version remains until safe cleanup. |
| What do `xmin` and `xmax` mean? | They are tuple transaction metadata associated with insertion and deletion/update visibility, with implementation nuances. |
| Why not delete old versions immediately? | An older active snapshot may still need to see them. |
| Does VACUUM FULL behave like ordinary VACUUM? | No. Ordinary VACUUM makes space reusable inside the relation; VACUUM FULL rewrites the table and needs a stronger lock. |
| Does SELECT FOR UPDATE lock the whole table? | Normally it locks selected rows with a row-level lock, plus the required table-level lock mode. |
| Does SELECT FOR UPDATE block every reader? | Ordinary MVCC readers can generally continue reading a visible version. Conflicting writers/locking operations may wait. |
| Does REPEATABLE READ take a new snapshot for every statement? | In PostgreSQL it uses a stable transaction snapshot, unlike READ COMMITTED's per-statement snapshots. |
| Does REPEATABLE READ guarantee concurrent updates always succeed? | No. A transaction can fail when a target row changed after its snapshot; retry logic may be required. |
| Does partitioning distribute writes across machines? | Not by itself. PostgreSQL table partitioning is generally within one database cluster; sharding distributes data across independent nodes. |
| Do read replicas increase primary write capacity? | Not directly. They offload eligible reads; replication also consumes resources. |
| Does a bigger machine automatically solve write throughput? | No. Hot rows, WAL durability, indexes, query work, locks, and connection pressure can remain limiting. |
| Is an index always faster than a sequential scan? | No. When many rows qualify, sequential access can be cheaper than many index-driven heap fetches. |
| Does parallel query make every query faster? | No. It helps eligible, sufficiently large operations and introduces worker/coordination overhead. |
| Is planner cost measured in milliseconds? | No. It is an estimated relative cost in planner units. Use actual execution metrics to measure elapsed time. |
| Should we increase `work_mem` globally to solve spills? | Not blindly. Memory is consumed by individual operations and concurrent queries, so concurrency multiplies demand. |
| Should we disable `fsync` to increase write throughput? | Not as a normal production optimization. It compromises crash durability. |

---

## 12. Production investigation workflow

When a PostgreSQL service slows down, use a disciplined sequence:

1. **Define the symptom:** throughput, p50/p95/p99 latency, error rate, queue depth, and when the regression began.
2. **Check the host:** CPU saturation, memory pressure/swap, disk latency/IOPS, network, and process count.
3. **Check PostgreSQL activity:** active sessions, wait events, long transactions, blockers, and connection count.
4. **Rank query workload:** use `pg_stat_statements` to find queries by total time, mean time, calls, and rows.
5. **Inspect plans:** use `EXPLAIN (ANALYZE, BUFFERS)` on representative queries.
6. **Check maintenance:** dead tuples, autovacuum progress, table/index bloat indicators, and statistics freshness.
7. **Form one hypothesis:** CPU plan inefficiency, memory spill, storage/WAL pressure, lock contention, or connection overload.
8. **Change one meaningful variable:** benchmark against a baseline and validate correctness.
9. **Check for shifted costs:** a new index can improve reads while reducing write throughput; more parallelism can increase CPU contention.
10. **Keep the fix observable:** add dashboards/alerts for the metric that exposed the bottleneck.

A useful monitoring stack can include PostgreSQL statistics views, Prometheus, Grafana, and log analysis tools such as pgBadger. Instrumentation should help connect application requests to database queries and waits.

---

## 13. Final architecture challenge (not solved yet)

The end-of-course scenario is:

> A PostgreSQL database has 128 CPU cores, 512 GB RAM, and extremely fast NVMe storage, but it still cannot process more than 10,000 writes per second. How would you investigate and fix it?

We will answer this only after completing the remaining modules. The point of the scenario is to avoid assuming that the advertised hardware resources are the bottleneck.

A strong investigation will test at least these hypotheses:

- **CPU:** query/trigger/index maintenance work, WAL processing, contention, or poor concurrency.
- **Memory:** cache effectiveness, memory pressure, per-operation memory, and concurrency.
- **Storage:** WAL flush latency, checkpoint behavior, write amplification, and durability settings.
- **Locks:** hot rows, lock queues, long transactions, and serialization conflicts.
- **Workload shape:** indexes per table, batch size, transaction size, connection count, synchronous commit requirements, and producer burstiness.
- **Architecture:** bounded ingestion, batching, partitioning, replicas for reads, and only then whether sharding is justified.

The correct fix depends on measurements and business guarantees. “Add hardware,” “disable durability,” and “shard immediately” are not diagnoses.

---

## 14. Remaining study plan

### Module 2: Memory internals

- PostgreSQL shared buffers vs operating-system page cache.
- Buffer lookup, cache hits/misses, and eviction.
- `work_mem` per sort/hash operation and concurrency multiplication.
- Sort/hash spills and temporary files.
- Memory sizing, OOM risks, and workload-aware tuning.

### Module 3: Storage and the write path

- WAL generation and WAL buffers.
- Commit path, WAL flush, `fsync`, and durability.
- Checkpoints, background writer, dirty buffers, and full-page writes.
- NVMe latency, IOPS, throughput, queue depth.
- Write amplification, WAL volume, and replication implications.

### Module 4: Locks and concurrency internals

- Row-level and table-level lock modes.
- MVCC visibility vs write conflict locking.
- Lock queues, blockers, hot rows, and contention.
- Deadlock detection and deterministic lock ordering.
- READ COMMITTED, REPEATABLE READ, SERIALIZABLE in practical concurrent workloads.
- Long-running transactions, vacuum horizons, and cleanup.

### Final assessment

Return to the 128-core / 512-GB / NVMe / 10K writes-per-second scenario and build a metrics-driven incident response, identify the bottleneck, propose safe optimizations, and explain the trade-offs.

---

## 15. Quick revision card

- **MVCC:** multiple tuple versions plus snapshots determine row visibility.
- **`xmin` / `xmax`:** transaction metadata used in tuple visibility and lifecycle.
- **Dead tuple:** obsolete row version that cannot yet or no longer needs to be visible.
- **VACUUM:** makes dead-tuple space reusable; not the same as VACUUM FULL.
- **HOT update:** can avoid index tuple updates when index keys are unchanged and page conditions permit.
- **`SELECT FOR UPDATE`:** locks selected rows against conflicting operations until transaction end.
- **READ COMMITTED:** statement-level snapshots.
- **REPEATABLE READ:** stable transaction snapshot in PostgreSQL.
- **SERIALIZABLE:** detects dangerous concurrent dependencies and can abort transactions for retry.
- **Partitioning:** divides a table within a database cluster.
- **Sharding:** distributes data across independent database nodes.
- **Sequential scan:** often optimal when a large fraction of rows must be read.
- **Index scan:** finds candidate tuple locations through an index, then may fetch heap rows.
- **Nested Loop:** repeated inner lookup for each outer row.
- **Hash Join:** build a hash table, then probe it.
- **Merge Join:** process ordered inputs together.
- **Cardinality estimate:** planner's prediction of rows; bad estimates can produce bad plans.
- **Parallel query:** workers divide eligible query work, with coordination overhead.
- **`EXPLAIN ANALYZE`:** executes the query and reports actual plan behavior.
- **`pg_stat_statements`:** identifies expensive query patterns across the workload.
- **Bottleneck-first tuning:** diagnose CPU, memory, storage, locks, and workload before changing settings.

## Key takeaway

**PostgreSQL performance is an end-to-end property of the query plan, tuple/version lifecycle, concurrency pattern, memory budget, WAL/durability path, and workload shape. Optimize the measured bottleneck while preserving correctness and durability.**
