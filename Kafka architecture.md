# Kafka : Basics, Internals & Exactly-Once Semantics

## 1. Kafka Basics

### What is Kafka?
Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant, append-only publish/subscribe messaging and stream processing.

### Core Concepts
- **Topic:** A named category or feed to which records are published (e.g., `orders-incoming`, `inventory-reserved`).
- **Partition:** A topic is split into one or more ordered, immutable, append-only logs called partitions. Parallelism and throughput in Kafka scale primarily through partitions.
- **Record / Message:** A unit of data containing a key, value, timestamp, optional headers, and a partition **offset**.
- **Offset:** A sequential, zero-based integer ID assigned to each message within a specific partition (`0, 1, 2, ...`). Offsets are unique per partition, not across the entire topic.
- **Producer:** Client application that publishes records to topic partitions (often routing by key hash or round-robin/sticky batching).
- **Consumer & Consumer Group:**
  - A **Consumer** reads records from assigned partitions and tracks its progress via committed offsets.
  - A **Consumer Group** is a set of consumers cooperating on a topic; each partition in a topic is consumed by at most one consumer within a given group.
- **Broker:** A Kafka server node that stores partition log segments and serves producer/consumer requests.

---

## 2. Kafka Internals

### Append-Only Log & Segment Files
- Each partition is backed by a directory on disk containing **log segment files** (`.log`) along with index files (`.index`, `.timeindex`).
- Writes are sequential appends to the active segment, avoiding random disk seeks and maximizing throughput.

### Sparse Indexing
- Instead of indexing every single message (which would consume excessive memory/disk), Kafka uses **sparse indices** that map offsets (or timestamps) to physical byte positions at intervals (`index.interval.bytes`).
- To locate an offset, Kafka binary-searches the sparse `.index` file to find the closest entry less than or equal to the target offset, jumps to that byte position in the `.log` file, and scans sequentially from there.

### Replication, Leaders, Followers & ISR
- Each partition has a **Replication Factor** (e.g., `3`), with one **Leader** replica and follower replicas on other brokers.
- Producers write to and consumers read from the partition leader (unless follower fetching is configured).
- **ISR (In-Sync Replicas):** The set of replicas (including the leader) that are caught up with the leader within `replica.lag.time.max.ms`.

### Log Retention & Log Compaction
- **Time / Size Retention:** Old log segments are eligible for deletion once expired (`retention.ms`) or once the log exceeds `retention.bytes`.
- **Log Compaction:** Retains the **latest value for each message key** within a partition (outside the active head segment), useful for changelog/state topics.

### Internal System Topics
- **`__consumer_offsets`:** A compacted internal topic storing committed consumer offsets per `(consumer_group, topic, partition)`.
- **`__transaction_state`:** A compacted internal topic storing transaction metadata and state machine transitions for transactional producers.

---

## 3. Exactly-Once Semantics (EOS)

### Why Delivery Guarantees Are Tricky
- **At-Most-Once:** Messages may be lost (e.g., offset committed before processing completes), never duplicated.
- **At-Least-Once:** No messages lost, but duplicates can occur on retries or crashes before offset commit.
- **Exactly-Once (EOS):** Every input record is processed and reflected in output topics and consumer offsets once—even across network retries and client/broker crashes.

### Pillar 1: Idempotent Producer
- Prevents duplicate messages on a single partition when network acknowledgments are lost and the producer retries.
- The broker assigns the producer a **Producer ID (PID)** and tracks a monotonically increasing **sequence number** per partition.
- If a retry sends a batch with a sequence number already committed for that `(PID, partition)`, the broker deduplicates it safely without appending a second copy.

### Pillar 2: Transactions & The Transaction Coordinator
- Enables **atomic writes across multiple partitions and topics** ("all-or-nothing").
- **Transaction Coordinator:** A module running inside a Kafka broker determined by hashing the producer's `transactional.id` onto a partition of `__transaction_state`:
  `Partition = abs(transactional.id.hashCode()) % num_partitions`
- **Transaction State Machine:** Tracks lifecycle states durably in `__transaction_state`:
  `Empty` -> `Ongoing` -> `PrepareCommit` (or `PrepareAbort`) -> `CompleteCommit` (or `CompleteAbort`) -> `Empty`.
- **Zombie Fencing:** A higher **producer epoch** associated with a `transactional.id` fences out stale/zombie producer instances so they cannot corrupt an active transaction.

### Pillar 3: Commit & Abort Markers (Control Records)
- Because partition logs are append-only, markers are special **control records** appended to the partition log (consuming an offset) rather than mutating prior data messages:
  - **Commit Marker:** Appended to participating output partitions (and `__consumer_offsets`) when a transaction commits.
  - **Abort Marker:** Appended when a transaction aborts or times out/crashes before committing.

### Pillar 4: Read-Committed Consumers & Atomic Offset Commits
- Downstream consumers set `isolation.level=read_committed`:
  - They buffer transactional messages and only expose records up to the **Last Stable Offset (LSO)** once a **Commit Marker** is read.
  - If an **Abort Marker** is encountered, aborted transactional records are discarded and never surfaced to application logic.
- **Consume-Transform-Produce (`sendOffsetsToTransaction`):** The input consumer's offset update is written to `__consumer_offsets` as part of the *same* transaction as the output records. If the app or broker crashes before commit, output records are aborted and the input offset does not advance, so the record is cleanly reprocessed on restart.
