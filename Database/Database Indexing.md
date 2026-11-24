# Database Indexing: Fundamentals, Index Families, and Distributed Indexes

> Study notes based on the discussion of distributed database indexing, Local Secondary Indexes (LSIs), Global Secondary Indexes (GSIs), and the broader taxonomy of database indexes.

## 1. What is a database index?

An index is an auxiliary data structure that helps a database locate rows without scanning the entire table. It stores searchable key information and, depending on the implementation, a row identifier, primary key, or a representation of the indexed data.

A useful mental model is the index at the back of a book: it maps a term to the page where the term can be found.

Indexes improve read efficiency, but they are not free. They consume storage and require maintenance when data changes. More indexes can mean more write work.

### Without an index

For a query such as:

```sql
SELECT *
FROM Users
WHERE email = 'user@example.com';
```

the database may need to scan many or all rows.

### With an index

The database can use the index to locate matching row identifiers and then retrieve the corresponding rows. The query planner chooses whether using the index is cheaper than a sequential scan.

An index does not guarantee that the optimizer will use it. Selectivity, table size, statistics, and the query shape all matter.

---

## 2. Database indexing is a family of overlapping classifications

There is no single mutually exclusive list of index types. Indexes can be classified along several independent dimensions:

1. **Underlying data structure:** B-tree/B+ tree, hash, bitmap, inverted index, LSM-based structures, and specialized search indexes.
2. **Relationship to table storage:** clustered and non-clustered.
3. **Key definition and query coverage:** single-column, composite, unique, partial, expression, and covering indexes.
4. **Distribution in a partitioned or sharded system:** local and global indexes.

These dimensions overlap. For example, an index can be a composite, non-clustered B-tree index. A global secondary index is a logical/distribution concept; it does not, by itself, specify whether the storage engine uses a B+ tree, LSM tree, or another physical structure.

## 3. Index families by underlying data structure

### 3.1 B-tree and B+ tree indexes

Tree-based indexes keep keys ordered and support equality lookups, range scans, and ordered traversal.

A B+ tree typically has:
- Internal nodes containing separator keys and child pointers.
- Leaf nodes containing key entries and row references or records.
- Leaf nodes linked in key order in many implementations.

Why they are useful:
- Efficient point lookups.
- Efficient range queries such as `age BETWEEN 20 AND 30`.
- Ordered scans, sorting, and prefix/range access.

Typical operations involve traversing from the root to a leaf. Tree height is kept relatively small through high fanout. Inserts and deletes may cause page splits, merges, or redistribution.

Example:

```sql
CREATE INDEX idx_users_age ON Users(age);

SELECT *
FROM Users
WHERE age BETWEEN 20 AND 30;
```

**B-tree vs B+ tree:** These names are often used loosely in database discussions, but they are not identical structures. Many relational database engines use B+ tree variants for their ordered indexes.

### 3.2 Hash indexes

A hash index applies a hash function to a key to locate a bucket.

Best suited to equality predicates:

```sql
SELECT *
FROM Users
WHERE user_id = 101;
```

Hash indexes generally do not support ordered range scans in the way B-tree indexes do. Hash collisions must be handled by the implementation.

### 3.3 Bitmap indexes

Bitmap indexes represent the rows containing a value using bitmaps.

They can be effective for low-cardinality columns, especially in analytical workloads. For example, a column such as `is_active` or a small set of statuses may be represented compactly.

Bitmap operations can combine predicates efficiently using bitwise AND/OR operations. They are not universally ideal for high-concurrency, write-heavy OLTP workloads.

### 3.4 LSM-tree-based storage and indexes

Log-Structured Merge Trees (LSM trees) are storage/indexing structures designed to make writes efficient.

A simplified lifecycle:
1. New writes are recorded in an in-memory structure (often called a MemTable).
2. When full, the structure is flushed to immutable sorted files (often called SSTables).
3. Background compaction merges files and reclaims obsolete entries.

LSM-based designs trade some read/compaction complexity for efficient write paths. They are used by systems and storage engines such as LevelDB and RocksDB, and underpin several distributed database designs.

Important distinction: LSM tree is a storage-engine/index organization approach, not the same classification dimension as local vs global index.

### 3.5 Specialized indexes

- **Inverted index:** maps a term/token to documents or records containing it; common in full-text search.
- **GIN (PostgreSQL):** useful for values containing multiple elements, such as arrays, JSONB, and text-search vectors.
- **GiST (PostgreSQL):** extensible index framework used for geometric, range, and other specialized operators.
- **BRIN (PostgreSQL):** stores summaries for ranges of physical table blocks; useful for very large tables where indexed values correlate with physical row order.

These are specialized index families. Their availability and exact behavior depend on the database engine.

---

## 4. Index families by how the index relates to table storage

### 4.1 Clustered index

A clustered index determines how the table's rows are physically organized (or, more precisely, the primary storage organization) around an index key.

Conceptually, the leaf level contains the table rows themselves.

This can make range retrieval by the clustering key efficient because nearby key values are stored together.

### 4.2 Non-clustered index

A non-clustered index is a separate structure. Its entries contain indexed key values and a row locator, primary key, or equivalent reference used to find the base row.

A lookup may therefore involve:
1. Searching the index.
2. Following the row reference to retrieve columns not present in the index.

**Database-specific nuance:** SQL Server has an explicit clustered/non-clustered index model. PostgreSQL tables are heaps; its `CLUSTER` command physically reorders a table using an index but does not continuously maintain that order as rows change. Do not assume every database uses the same clustered-index semantics.

### 4.3 Primary and secondary indexes

These terms describe the role of an index in a schema or access strategy:
- A **primary-key index** supports the table's primary-key constraint and lookup.
- A **secondary index** provides another access path using a different key.

The exact relationship between primary/secondary and clustered/non-clustered is engine-specific. They are not universal synonyms.

---

## 5. Index families by key definition and query coverage

### 5.1 Single-column index

Indexes one column.

```sql
CREATE INDEX idx_orders_customer
ON Orders(CustomerId);
```

Useful for queries filtering by `CustomerId`.

### 5.2 Composite (multi-column) index

Indexes multiple columns in a defined order.

```sql
CREATE INDEX idx_orders_customer_date
ON Orders(CustomerId, CreatedAt);
```

This is commonly useful for queries such as:

```sql
SELECT *
FROM Orders
WHERE CustomerId = 101
  AND CreatedAt >= '2026-01-01';
```

**Column order matters.** For a conventional B-tree composite index, the leftmost-prefix principle means an index on `(CustomerId, CreatedAt)` is generally useful for predicates beginning with `CustomerId`, but is not generally an equivalent substitute for an index on `CreatedAt` alone.

### 5.3 Unique index

Enforces uniqueness of the indexed key (subject to the database's NULL semantics and constraint rules).

```sql
CREATE UNIQUE INDEX ux_users_email
ON Users(email);
```

### 5.4 Covering index

An index is covering for a particular query when it contains all columns needed to satisfy that query, so the engine may avoid fetching the base table row.

Some engines support explicitly included non-key columns (for example, SQL Server's `INCLUDE` and PostgreSQL's `INCLUDE`).

### 5.5 Partial / filtered index

Indexes only rows satisfying a predicate.

Example (syntax varies by database):

```sql
CREATE INDEX idx_open_orders
ON Orders(CreatedAt)
WHERE Status = 'OPEN';
```

This can reduce index size and maintenance when queries repeatedly target a subset of rows.

### 5.6 Expression / functional index

Indexes the result of an expression rather than only a raw column.

Example (syntax varies):

```sql
CREATE INDEX idx_users_lower_email
ON Users(LOWER(email));
```

It can help queries that use the same expression, such as `WHERE LOWER(email) = ...`.

---

## 6. Distributed database indexing: why it becomes a different problem

A conventional index on a single database instance is not the whole story once data is partitioned or sharded across machines.

Consider a `Blogs` table with:

| BlogId | AuthorId | Category |
|---|---|---|
| B1 | U1 | Java |
| B2 | U2 | System Design |
| B3 | U3 | Java |
| B4 | U4 | Database |
| B5 | U5 | Java |
| B6 | U6 | System Design |

Assume the table is sharded by `AuthorId`:

```text
shardId = hash(AuthorId) % numberOfShards
```

For illustration, the data could be distributed as:

**Shard 1**
- B1, U1, Java
- B2, U2, System Design
- B3, U3, Java

**Shard 2**
- B4, U4, Database
- B5, U5, Java
- B6, U6, System Design

The important invariant is that all records with the same partition/shard key value must route to the same logical partition under a deterministic partitioning scheme.

### 6.1 Important correction: the earlier inconsistent example

An earlier illustration placed two rows with `AuthorId = U1` on different shards while claiming the table was sharded by `AuthorId`. That was incorrect.

If the partition key is `AuthorId`, both rows for U1 must be routed to the same logical partition. The corrected example above uses distinct author IDs across the two shards.

This invariant is essential to understanding partition-key routing.

### 6.2 Query by the partition key

```sql
SELECT *
FROM Blogs
WHERE AuthorId = 'U1';
```

The router can hash or otherwise map U1 to its owning shard. This is a targeted, single-shard query (assuming the partitioning scheme and routing metadata are available).

### 6.3 Query by a non-partition-key attribute

```sql
SELECT *
FROM Blogs
WHERE Category = 'Java';
```

The table is partitioned by `AuthorId`, not `Category`. Java blogs may exist on any shard. The router cannot determine the owning shard from Category alone.

Without a suitable global access path, it may need to use **scatter-gather**:
- Scatter: send the query to multiple/all relevant shards.
- Gather: collect, merge, and possibly sort/aggregate the results.

Potential costs:
- Network fan-out.
- CPU and I/O work on many shards.
- Higher tail latency, because the overall request may wait for a slow shard.
- Increased internal request volume.

For example, 10,000 external requests per second fanning out to 100 shards could generate up to 1,000,000 shard-level query operations per second. This is an illustrative upper-bound multiplication, not a universal workload estimate.

---

## 7. Local and global secondary indexes

The words **local** and **global** describe index scope and distribution, not a universal physical data structure. Exact semantics vary by database product.

### 7.1 Local secondary index (LSI): conceptual model

A local index is maintained within the scope of the local partition/shard and covers data belonging to that partition.

Conceptually:

```text
Base shard 1                     Base shard 2
  B1, B2, B3                       B4, B5, B6
      |                                |
  Local index                      Local index
  Java -> B1, B3                   Java -> B5
  System -> B2                     Database -> B4
```

A local index can efficiently find matching rows within a known shard. However, if the query does not identify which shard(s) to search, local indexes alone do not necessarily eliminate scatter-gather. The system may still need to query every relevant shard's local index.

Local indexes are useful when the query is scoped to a known partition and the index provides another ordering/filtering dimension within that partition.

### 7.2 Global secondary index (GSI): conceptual model

A global secondary index provides an access path whose key/partitioning can differ from the base table's partitioning.

For example:

```text
Base table:
  Partition key = AuthorId

GSI:
  Partition key = Category
  Optional sort key = CreatedAt
```

Conceptually, the GSI maintains index entries organized around Category. A query for `Category = 'Java'` can be routed to the GSI's relevant partition(s), instead of broadcasting the query to every base-table shard.

This is another representation/access path over the same logical records, organized to serve a different query pattern.

**Critical clarification:** A GSI is not defined by being a sorted key-value store rather than a B+ tree. It is a logical/distribution-level index concept. The physical engine may use B-tree-like structures, LSM-based structures, or proprietary implementations. A B+ tree is itself one possible implementation of an ordered key-value index.

### 7.3 LSI vs GSI: high-level comparison

| Dimension | Local index | Global secondary index |
|---|---|---|
| Scope | Within a local partition/shard | Across the dataset, with an independent access path |
| Key/partitioning | Often tied to the base partition key | Can use a different partition key |
| Query routing | May require checking multiple shards if shard is unknown | Can target the index partition(s) using its key |
| Maintenance | Usually local to the owning partition | May require propagation to a different index partition |
| Main use | Alternate lookup/order within a known partition | Alternate global query pattern |

This is a conceptual comparison, not a universal product specification. For example, DynamoDB gives LSI and GSI precise product-specific meanings.

---

## 8. DynamoDB: concrete LSI and GSI semantics

Consider a DynamoDB Orders table:

| CustomerId (PK) | OrderId (SK) | Status | Amount |
|---|---|---|---:|
| C101 | O1 | PLACED | 1000 |
| C101 | O2 | SHIPPED | 2000 |
| C102 | O3 | PLACED | 500 |
| C103 | O4 | DELIVERED | 3000 |

### 8.1 DynamoDB Local Secondary Index

An LSI:
- Uses the same partition key as the base table.
- Uses a different sort key.
- Is created when the table is created.
- Allows alternate sort-key access within the same partition-key value.

Example:

```text
Base table PK: CustomerId
Base table SK: OrderId

LSI PK: CustomerId
LSI SK: Status
```

This supports customer-scoped queries using the alternate sort key.

### 8.2 DynamoDB Global Secondary Index

A GSI:
- Can use a different partition key from the base table.
- Can have its own sort key.
- Can be added after table creation.
- Has its own index storage and throughput/capacity characteristics, depending on table capacity mode.

Example:

```text
GSI PK: Status
GSI SK: OrderId
```

This supports querying orders by status, even though the base table is keyed by CustomerId.

**DynamoDB consistency details:**
- GSI reads are eventually consistent; strongly consistent reads are not supported on GSIs.
- LSI queries support strongly consistent reads.
- LSIs and GSIs have different storage, capacity, and lifecycle constraints. Consult the official documentation for current limits and detailed behavior.

Official documentation: [DynamoDB secondary indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html)

---

## 9. The costs and failure modes of global indexes

A global index improves reads for an alternate access pattern, but maintaining it introduces work.

### 9.1 Write amplification

Suppose one logical record is written to the base table and three secondary indexes need updates:

```text
One logical write
  ├── Base table write
  ├── GSI 1 maintenance
  ├── GSI 2 maintenance
  └── GSI 3 maintenance
```

This can increase:
- Storage usage.
- Network traffic.
- CPU and disk I/O.
- Write capacity/throughput consumption.
- Operational and recovery complexity.

The exact write path and atomicity guarantees depend on the database.

### 9.2 Index propagation lag and stale reads

Consider an update to a record:
1. The base-table write becomes visible.
2. The index update has not yet propagated.
3. A query through the index may temporarily return stale results or omit the latest change.
4. After successful propagation, the index catches up.

This is often described as replication lag or index propagation lag.

**Important nuance:** Eventual consistency does not mean a failed index update automatically fixes itself. The database needs a reliable retry/recovery mechanism. Eventual convergence assumes updates are eventually propagated successfully.

### 9.3 Hot partitions

Suppose a GSI uses `Status` as its partition key and a large fraction of writes have:

```text
Status = PENDING
```

Many writes may concentrate on the same logical partition-key value. Depending on the database's physical partitioning and adaptive-scaling behavior, this can create a hot partition or hot key.

A common technique is write sharding / bucketing:

```text
PENDING#1
PENDING#2
PENDING#3
...
PENDING#N
```

Writes are distributed among buckets. The trade-off is that a query for all PENDING records must query multiple buckets and merge the results.

The bucket count and routing strategy must be designed for the expected traffic and access patterns.

### 9.4 Why not create a GSI for every possible query?

Because every index has a cost:
- Additional storage.
- Additional write amplification.
- Additional capacity and network work.
- Potential hot keys/partitions.
- More operational complexity.

Create indexes for important, measured access patterns. Do not index every conceivable predicate without considering workload and cost.

---

## 10. Answers to the questions used to test understanding

### Q1. The base table is partitioned by UserId. Why is a local index on Email not enough to avoid scatter-gather?

A local index can find email matches within one shard. But the application does not know which shard contains a given email, because the partition key is UserId. It may have to query every shard's local index. A global access path keyed by Email can route the lookup to the relevant index partition(s).

### Q2. Why does a GSI introduce additional write overhead?

A write to the base table may also require creating, updating, or deleting the corresponding GSI entry. The index may be partitioned differently, so propagation can involve additional storage and network work.

### Q3. What if the base write succeeds but the GSI update is delayed?

The base table may show the new value while an eventually consistent GSI query temporarily returns stale or incomplete results. The database must propagate/retry the index update for the index to converge.

### Q4. What if a GSI uses Status as its partition key and 500,000 orders become PENDING?

A concentration of writes on the same key can create a hot key/partition. Write sharding, such as `PENDING#1` through `PENDING#N`, can distribute writes. Reads then need to fan out across the buckets and merge results.

### Q5. Why not create 50 GSIs?

Every GSI consumes storage and adds index-maintenance work, write capacity/network overhead, and operational complexity. Index only high-value access patterns.

---

## 11. A deeper consistency scenario: changing an indexed attribute

Suppose a blog changes category from Java to Database.

The logical index maintenance is:

1. Update the base-table record.
2. Remove the old index entry from the Java index partition.
3. Insert the new index entry into the Database index partition.

Conceptually:

```text
Before:
  Base record: category = Java
  GSI: Java -> Blog B6

Update:
  Base record: category = Database

Index maintenance:
  Remove Java -> B6
  Add Database -> B6

After:
  Base record: category = Database
  GSI: Database -> B6
```

If the base update and index changes are not one atomic operation, a crash or failure between steps can temporarily leave the index inconsistent with the base table.

How a particular database prevents, retries, or exposes this intermediate state is implementation-specific. Managed databases may own the index-maintenance machinery; applications should not assume they need to manually update a managed GSI.

This scenario connects to broader distributed-systems concepts:
- Atomicity and transaction boundaries.
- Asynchronous replication and propagation lag.
- Retry and idempotency.
- Durable change capture.
- Transactional Outbox Pattern (for application-managed projections).

### Transactional Outbox connection

When an application itself maintains a separate search index or read model, the Outbox Pattern can help avoid losing the intent to update the projection:

1. In one database transaction, update the business row and insert an outbox event.
2. A background publisher reads committed outbox events.
3. It publishes the event to a broker or projection worker.
4. The consumer updates the secondary read model/index idempotently.
5. Failed processing is retried; duplicates are handled safely.

This does not make two independent stores magically atomic. It makes change propagation durable and recoverable. It is a common pattern for application-managed projections, not a claim about DynamoDB's internal GSI implementation.

---

## 12. Index selection: think in access patterns

A senior engineer should not begin with "Which index types can I create?" Instead, begin with:

**Which queries matter, how frequently do they run, what latency do they need, and how is the data partitioned?**

Example e-commerce access patterns:

| Requirement | Query/access pattern |
|---|---|
| Find a customer's orders | CustomerId |
| Find orders by status | Status |
| Find orders by merchant | MerchantId |
| Find a specific order | OrderId |
| Find a customer's recent orders | CustomerId + CreatedAt |

Possible design choices:
- Partition the base table by CustomerId.
- Use a sort key or local index for customer-scoped ordering/filtering.
- Consider a GSI for merchant- or status-based access.
- Evaluate hot-key risk for low-cardinality keys such as Status.
- Measure write amplification and index storage before adding more indexes.

A partition key that works well for one access pattern may be poor for another. Data modeling is therefore a workload and distribution problem, not merely a syntax exercise.

---

## 13. Quick revision sheet

| Concept | Mental model |
|---|---|
| B+ Tree | Ordered tree index; strong for point and range queries |
| Hash index | Hash-to-bucket lookup; equality-oriented |
| Bitmap | Bit vectors; useful for low-cardinality analytical predicates |
| LSM Tree | Write-optimized organization using sorted files and compaction |
| Inverted index | Term to documents/records |
| Clustered index | Determines primary physical row organization |
| Non-clustered index | Separate key structure pointing to rows |
| Composite index | Multiple ordered key columns |
| Covering index | Contains all data required by a particular query |
| Partial index | Indexes only rows matching a predicate |
| Local index | Index scoped to local partition/shard |
| Global secondary index | Alternate access path across the dataset |
| Scatter-gather | Fan out query, collect and merge results |
| Write amplification | Extra work to maintain indexes |
| Hot partition | Disproportionate traffic concentrated on a partition/key |
| Eventual consistency | Replicas/projections converge after propagation succeeds |

## 14. Recommended next topics

To deepen this understanding, study in this order:

1. B+ Tree internals: fanout, page splits, leaf pages, range scans, and disk I/O.
2. Clustered vs non-clustered indexes: row locators, index lookups, and heap/table access.
3. Composite indexes: leftmost-prefix behavior, selectivity, and column ordering.
4. Query planning: sequential scan vs index scan, cardinality estimates, and `EXPLAIN ANALYZE`.
5. LSM Trees: MemTable, SSTable, compaction, bloom filters, and read amplification.
6. Distributed indexing: index propagation, consistency, hot keys, and write sharding.

## Key takeaway

**An index is an access path over data. In a distributed database, the index's partitioning and maintenance strategy are just as important as its lookup structure.**

A B+ tree or LSM tree describes how an index/storage engine may organize data. Local vs global describes the scope and distribution of an index. Clustered vs non-clustered describes its relationship to the table's row storage. These are different dimensions and should not be confused.
