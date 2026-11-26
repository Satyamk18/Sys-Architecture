# Amazon Aurora: Storage–Compute Separation

> **Learning goal:** Understand why Aurora separates database compute from durable storage, how a write travels through the system, and what design lessons a Staff Engineer should take from it.
>
> **Source video:** [What the heck is Storage-Compute Separation?](https://youtu.be/DA5W8tO_7Nw)
>
> **Related reading:** [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://www.cs.purdue.edu/homes/bb/cs542-22Spr/readings/impl/sigmod-17-amazon-aurora-design.pdf)

---

## 1. Start with the problem

A traditional relational database often runs two big responsibilities together:

1. **Compute:** Parse SQL, build query plans, execute queries, manage transactions, and maintain caches.
2. **Storage:** Persist database pages, manage files and disks, and make sure committed data survives failures.

This tight coupling creates scaling and operational problems.

Imagine a database server with 32 CPU cores and a large amount of RAM. The application becomes read-heavy, so we add more database servers. If each server owns a separate copy of the data, we now need to keep those copies in sync. If we instead add CPU to the existing server, storage and compute still scale as one unit.

The key question is:

**Can we scale database compute independently while keeping one shared, highly durable database volume?**

Aurora's answer is to separate the database compute layer from a distributed storage layer.

## 2. The architecture at a glance

Think of Aurora as two cooperating systems:

| Layer | Main responsibility | Typical work |
|---|---|---|
| Compute / DB instances | Understand and execute SQL | Connections, SQL parsing, query planning, transaction processing, buffer cache |
| Distributed storage | Durably store and replicate database changes | Persist redo records, maintain replicas, reconstruct pages, repair failures, back up data |

The compute layer presents a familiar relational database interface. The storage layer behaves like a distributed, replicated database volume rather than a simple disk attached to one machine.

A simplified view:

```text
              Application
                   |
              SQL connection
                   |
          +-------------------+
          |  Writer instance  |
          | SQL + transactions|
          | buffer cache      |
          +-------------------+
                   |
             redo records
                   |
       +-----------+-----------+
       | Distributed storage   |
       |                       |
       |  AZ-1  AZ-2  AZ-3     |
       |   copies across AZs   |
       |  quorum + repair      |
       +-----------------------+
                   |
          Shared database volume
                   |
        +----------+----------+
        |                     |
   Reader instance       Reader instance
   (same volume)         (same volume)
```

The diagram is conceptual. Aurora's storage implementation is more detailed than a single storage box, and its replication and quorum mechanics are covered in the companion note.

### The important mental model

- Compute instances are replaceable processing capacity.
- The database volume is shared durable state.
- Adding a reader does not mean creating and continuously synchronizing a complete independent storage copy in the traditional way.
- A compute instance failure should not automatically mean the database's durable data is lost.

**Separation is about independent responsibilities and scaling, not about removing all communication between compute and storage.**

## 3. Why separate compute and storage?

### 3.1 Independent scaling

Suppose the workload changes:

- More concurrent SQL queries require more CPU.
- A reporting workload needs more read capacity.
- The dataset grows, but query traffic stays almost flat.
- A burst of traffic needs temporary compute capacity.

With tightly coupled compute and storage, scaling one dimension can force you to scale the other. With separation, the system can add or replace compute instances while the shared storage layer continues to own the durable volume.

This is a form of **independent resource scaling**.

Staff Engineer question: *Which resource is actually the bottleneck?* Do not add database instances blindly. First determine whether the constraint is CPU, memory, I/O, network, locks, or storage capacity.

### 3.2 Faster recovery from compute failure

If a database instance crashes, the durable data is not supposed to disappear with that machine. Another compute instance can be brought up and attached to the same database volume.

This reduces the amount of work that would otherwise be needed to copy a full database from another server before becoming useful.

Be precise: compute replacement and database recovery are not the same as instant failover. Connections may break, a new writer may need to be promoted, and applications still need retry and failover handling.

### 3.3 Shared storage for readers

Reader instances can use the same underlying database volume. This avoids the traditional requirement that each reader maintain a separate full storage copy through a conventional replica pipeline.

Readers still have their own compute resources and caches. They do not magically share RAM or buffer caches.

This distinction matters:

- **Shared storage** means the durable database volume is shared.
- **Separate compute** means each DB instance has its own CPU, memory, execution engine, and local cache.
- **Replication lag / read consistency** still needs careful treatment at the database-instance level.

### 3.4 Less work during scaling operations

If compute and durable storage are separate, changing compute capacity does not necessarily require copying the entire dataset. That can make scaling and replacement more practical.

The broader lesson is to avoid making expensive state movement a prerequisite for adding processing capacity.

## 4. The critical design choice: send changes, not whole pages

To understand Aurora, distinguish a **data page** from a **redo log record**.

### Data page

A database page is a block of data containing rows and index information. A database engine reads and updates pages in memory, then eventually persists modified pages.

A page can be several kilobytes. Sending a whole page for every small change can create significant network and storage traffic.

### Redo log record

A redo record describes a change that can be applied to database state.

For example, instead of sending an entire page after changing one row, the database can send the information needed to redo that change.

A redo stream is therefore often much smaller than repeatedly shipping complete modified pages.

### Why this matters

Traditional page-oriented persistence can cause **write amplification**:

- A small logical update can dirty a page.
- The database may later write the full page.
- Replication may also move large page images.
- Multiple copies of those pages create more network and storage work.

Aurora's architecture sends redo records from the compute layer to the storage layer. Storage nodes persist the log records and use them to maintain the page representation.

The core idea:

**Move the description of the change through the system, instead of repeatedly moving the full resulting page.**

This is one of the most important performance ideas in the architecture.

## 5. What happens during a write?

At a high level:

1. The application sends a SQL write to the writer instance.
2. The database engine executes the statement and generates redo records.
3. The writer sends redo records to the distributed storage layer.
4. Storage nodes persist the records according to Aurora's replication and quorum rules.
5. Once the required durability condition is met, the write can be acknowledged as durable.
6. Storage continues background work such as catching up replicas, organizing log records, and materializing data pages.

The companion note, **Amazon Aurora: Durability with Quorum Writes**, explains the six-copy layout and the 4-of-6 write quorum.

### Foreground versus background work

A useful systems-design distinction is separating work that must complete before the client receives success from work that can happen later.

| Foreground path | Background path |
|---|---|
| Execute transaction | Organize and group log records |
| Generate redo | Fill missing replica records |
| Send redo to storage | Materialize data pages |
| Wait for required durability acknowledgements | Backup and cleanup work |

Moving non-essential work out of the critical path can reduce latency. But this is safe only when the system has a clear definition of what must be durable before acknowledging success.

**Do not confuse “background” with “optional.”** Background work may be essential for long-term efficiency, recovery, repair, and storage management.

## 6. What does the storage layer do?

Aurora storage nodes do more than expose a disk-like interface. The storage tier participates in maintaining the database volume.

Conceptually, storage nodes:

- Receive redo records.
- Persist records in an update/log structure.
- Track the progress of the log.
- Exchange information with peers to fill missing records.
- Reconstruct or materialize database pages from redo history.
- Maintain durable copies across failure domains.
- Support backup, garbage collection, and integrity checks.

This is a key architectural insight:

**The storage tier is an active distributed system, not merely a passive block device.**

### Why materialize pages?

Redo records are efficient to replicate, but queries need data pages. Replaying a very long history of log records every time a page is requested would be expensive.

Storage therefore materializes pages over time or when needed. It combines earlier page state with later redo records to produce a usable page version.

This is similar to the relationship between:

- an append-only change history, and
- a materialized current representation.

The log is useful for durability and incremental updates. Pages are useful for efficient reads.

## 7. Failure domains and Availability Zones

A **failure domain** is a group of resources that can fail together because they share some dependency.

Examples:

- One process
- One machine
- One rack
- One power circuit
- One Availability Zone (AZ)
- An entire Region

An Availability Zone is an isolated location within an AWS Region. Multiple AZs are connected, but are designed to reduce the chance that a single local event takes them all down.

Why does this matter?

A system that replicates data across six machines in one building may still lose all six copies if that building loses power. The number of replicas is not enough. Their placement matters.

Aurora spreads storage copies across multiple AZs. The companion quorum note explains why correlated failures change the math.

**Staff-level lesson:** Count independent failure domains, not just replica instances.

## 8. What this architecture does not mean

Avoid these common misunderstandings:

| Incorrect interpretation | Better mental model |
|---|---|
| Compute and storage never communicate | They communicate constantly, but own different responsibilities |
| Every reader has its own independent full database volume | Reader instances use the shared cluster storage volume |
| Readers share one memory cache | Each DB instance has its own compute and memory |
| Storage is just a network disk | Aurora storage is a distributed system with replication and recovery logic |
| Sending redo means pages are never created | Pages are materialized in storage for efficient reads |
| Compute failure means data loss | Durable storage is independent, though service failover and client recovery still matter |
| More replicas automatically means more safety | Placement, quorum rules, correlated failures, and repair time matter |

## 9. Trade-offs and costs

Separation is powerful, but it does not make complexity disappear.

### 9.1 Network is now part of the database I/O path

The compute layer depends on a remote distributed storage service. Network latency, bandwidth, congestion, and partial failures become important.

### 9.2 Distributed storage is harder to operate

The storage tier must handle replica failures, slow nodes, missing log records, membership changes, repair, and integrity checks.

### 9.3 Shared storage creates a common dependency

Multiple compute instances benefit from a common durable volume, but the storage service becomes a critical shared subsystem. It must be engineered for availability and scale.

### 9.4 Cache behavior still matters

Compute instances have local buffer caches. A newly started or promoted instance may not have the same warm cache as the previous writer. Cache misses can affect performance even if the underlying data is safe.

### 9.5 Consistency must be explicit

A shared volume does not mean every reader immediately observes every write in every situation. The database must coordinate log progress and read visibility. Applications must understand the consistency guarantees of the database configuration they use.

## 10. How to reason about this as a Staff Engineer

When evaluating a storage-compute-separated architecture, ask these questions in order.

### A. Responsibility boundaries

- Which component owns transaction execution?
- Which component owns durable state?
- Which component is responsible for replication?
- Which component reconstructs read-optimized data?
- Where does recovery happen?

If ownership is unclear, incidents become difficult to debug.

### B. Scaling boundaries

- Can compute scale without copying durable data?
- Can storage capacity grow without scaling CPU?
- Are read and write workloads competing for the same resources?
- Does adding readers actually relieve the bottleneck?

### C. Critical path

- What must happen before acknowledging a write?
- How many network round trips are required?
- Which tasks can move to the background?
- What happens when one replica is slow?
- Does a slow replica block progress unnecessarily?

### D. Failure behavior

- What happens if the writer process crashes?
- What happens if a storage node becomes unavailable?
- What happens if an entire AZ fails?
- Can the system still accept writes?
- Can it still prove that acknowledged writes are safe?
- How quickly can it repair redundancy?

### E. Operational behavior

- How do we detect partial failures?
- How do we distinguish a slow node from a failed node?
- How do we avoid repair storms?
- How do we deploy storage software safely?
- Can we test failure scenarios without risking the whole volume?

### F. Cost and performance

- What is the cost of extra replicas?
- How much data is moved for a small update?
- How much work is on the foreground path?
- What is the read amplification?
- What is the repair bandwidth requirement?

A Staff Engineer does not stop at “this design scales.” They ask **which bottleneck moved, which new dependency appeared, and how the system behaves when assumptions fail.**

## 11. Apply the pattern to another system

Imagine building a multi-tenant SaaS application with:

- Stateless API servers
- A shared durable database
- Independent read capacity
- A distributed object store for files

The same design questions apply:

1. Can API compute scale without moving tenant data?
2. What is the durable source of truth?
3. What must be synchronously persisted before returning success?
4. Which copies are in independent failure domains?
5. How are missing copies repaired?
6. How does a newly started server warm its cache?
7. What happens if the shared storage service is unavailable?

The lesson is not “always separate compute and storage.” The lesson is to separate responsibilities when doing so improves scaling, recovery, and operational control enough to justify the added distributed-systems complexity.

## 12. Quick revision

- **Compute:** Runs SQL, transactions, and query execution.
- **Storage:** Owns durable, replicated database state.
- **Separation:** Lets compute capacity and durable storage evolve more independently.
- **Redo:** Describes database changes and can be cheaper to move than full pages.
- **Materialization:** Turns log history into pages that can be read efficiently.
- **Shared volume:** Multiple DB instances use the same underlying durable cluster storage.
- **Failure domain:** A set of resources that may fail together.
- **AZ-aware replication:** Protects against correlated failures in one location.
- **Critical path:** Work that must finish before a request can be acknowledged.
- **Staff mindset:** Understand boundaries, bottlenecks, failure behavior, repair, and operational cost.

## 13. Questions to test your understanding

Try answering these without looking at the note:

1. Why is it difficult to scale compute and storage independently in a traditional database?
2. What does Aurora gain by sending redo records instead of whole modified pages?
3. Why does the storage tier need to materialize pages?
4. If compute and storage are separated, why is the network now more important?
5. Why does a reader still need its own buffer cache?
6. What is the difference between a machine failure and an AZ failure?
7. Which tasks belong on the foreground write path, and which can happen asynchronously?
8. What new operational complexity is introduced by distributed storage?
9. How would you investigate whether a database is compute-bound or storage-bound?
10. Why is separation a trade-off rather than a universal rule?

---

## References

- [Arpit Bhayani: Storage–Compute Separation video](https://youtu.be/DA5W8tO_7Nw)
- [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases (SIGMOD 2017)](https://www.cs.purdue.edu/homes/bb/cs542-22Spr/readings/impl/sigmod-17-amazon-aurora-design.pdf)
- [AWS: High availability for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [AWS: Amazon Aurora as an Alternative to Oracle RAC](https://aws.amazon.com/blogs/database/amazon-aurora-as-an-alternative-to-oracle-rac/)

> **Scope note:** This is a concept-focused study guide for the video topic, expanded with the Aurora design paper and AWS documentation. It is not a verbatim transcript.
