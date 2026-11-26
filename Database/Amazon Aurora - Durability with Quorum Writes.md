# Amazon Aurora: Durability with Quorum Writes

> **Learning goal:** Understand quorum mathematics, why Aurora uses six copies across three Availability Zones, how a write becomes durable, and how to reason about correlated failures and repair.
>
> **Source video:** [How Amazon Aurora Achieves Durability with Quorum Writes](https://youtu.be/EaWlpn24eAs)
>
> **Related reading:** [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://www.cs.purdue.edu/homes/bb/cs542-22Spr/readings/impl/sigmod-17-amazon-aurora-design.pdf)

---

## 1. Why do we need quorum writes?

A database must answer a deceptively difficult question:

**When can I tell the client that a write is safe, even though some machines may be slow, unreachable, or broken?**

A simple replication strategy might send every write to every replica and wait for all of them.

That is safe when all replicas respond, but one slow or failed replica can delay the entire write. In a distributed system, waiting for every participant can make availability and latency fragile.

Another strategy is to wait for only one replica. That is fast, but it may not provide enough protection. If that one replica fails before the data reaches the others, an acknowledged write could be lost.

A **quorum** is a carefully chosen minimum number of replicas whose acknowledgements are sufficient to make progress while preserving the required overlap properties.

Quorums let a system tolerate some unavailable replicas without treating every missing response as a reason to stop.

## 2. First understand the basic quorum model

Assume a piece of data has (V) replicas.

Define:

- (V): total number of replicas (votes).
- (V_w): number of replicas required to acknowledge a write.
- (V_r): number of replicas required for a read quorum.

For a simple majority-style quorum system, two important conditions are:

### Rule 1: A read quorum must overlap every write quorum

[
V_r + V_w > V
]

Why?

A read must intersect the set that accepted the latest write. If the read set and write set are disjoint, the read could miss the latest update.

### Rule 2: Every pair of write quorums must overlap

[
2V_w > V
]

Why?

If two write quorums could be completely disjoint, two writers might each believe they have successfully written a new value without seeing the other's write. Requiring a strict majority makes any two write quorums intersect.

### A small example

Suppose there are 3 replicas:

- (V = 3)
- (V_w = 2)
- (V_r = 2)

Check the rules:

[
V_r + V_w = 2 + 2 = 4 > 3
]

[
2V_w = 4 > 3
]

So the read and write sets overlap, and any two write quorums overlap.

This is the core mathematical idea behind quorum systems.

**Important:** Quorum intersection is a necessary building block, not a complete transaction protocol. Real systems also need versioning or ordering, conflict handling, membership management, and rules for deciding which value is newest.

## 3. Why Aurora uses six copies

Aurora's design targets two different failure goals:

1. **Lose an entire Availability Zone and one additional storage node without losing the ability to read/recover the data.** This is commonly called the AZ+1 failure objective.
2. **Lose an entire Availability Zone without losing write availability.**

To achieve this, Aurora's design uses:

- 6 copies of each protection-group segment.
- 3 Availability Zones.
- 2 copies in each AZ.
- Write quorum: 4 of 6.
- Read quorum: 3 of 6.

Conceptual layout:

```text
             One protection group
       +----------+----------+----------+
       |   AZ-1   |   AZ-2   |   AZ-3   |
       +----------+----------+----------+
       | Copy A   | Copy C   | Copy E   |
       | Copy B   | Copy D   | Copy F   |
       +----------+----------+----------+

             6 total copies
          2 copies in each AZ

       Write quorum = 4 of 6
       Read quorum  = 3 of 6
```

The replicas are not six copies on six unrelated machines in one failure domain. Their placement across three AZs is a central part of the design.

## 4. Verify the quorum math for Aurora

For Aurora's simplified quorum model:

- (V = 6)
- (V_w = 4)
- (V_r = 3)

Read/write intersection:

[
V_r + V_w = 3 + 4 = 7 > 6
]

Write/write intersection:

[
2V_w = 8 > 6
]

Therefore:

- Any read quorum of 3 intersects any write quorum of 4.
- Any two write quorums of 4 intersect each other.

The overlap is not an accident. It is the mathematical reason the chosen quorum sizes can provide the required guarantees.

### What does intersection actually guarantee?

Suppose a write is acknowledged by four copies. A later read quorum of three cannot be entirely made up of the other two copies, because there are only two copies outside the write quorum.

Therefore at least one member of the read quorum has participated in the acknowledged write.

That does **not** mean the application can simply read any arbitrary replica and always get the newest value. The system must use the log sequence, replica progress, and recovery rules to identify a sufficiently up-to-date copy.

## 5. Why not use three copies?

A common question: why not keep three copies across three AZs and require two acknowledgements?

A 2-of-3 write quorum and 2-of-3 read quorum satisfy the basic intersection math. But the failure goals are different.

With one copy per AZ:

- Losing one AZ leaves two copies.
- Losing another independent copy leaves one copy.
- A 2-of-3 write quorum is no longer available after the AZ failure plus one additional copy failure.
- The exact safety and availability objectives must be checked against the failure model.

Aurora's six-copy arrangement has two copies in each of three AZs. It is designed to tolerate an entire AZ loss and still have enough surviving copies to accept writes, while also meeting the AZ+1 durability objective.

**Staff-level lesson:** Do not choose a replica count by convention. Start with the failure scenarios you promise to survive, then derive placement and quorum sizes.

## 6. Walk through an AZ failure

Suppose AZ-1 becomes unavailable.

That removes two of the six copies.

The four copies in AZ-2 and AZ-3 remain.

Write quorum required:

[
V_w = 4
]

Surviving copies:

[
6 - 2 = 4
]

So the system can still form a 4-of-6 write quorum.

This is why the placement of two copies per AZ matters.

### What if one additional node fails too?

After losing AZ-1, suppose one more storage node fails in AZ-2 or AZ-3.

Now only three copies remain available.

- A 4-of-6 write quorum is no longer available.
- A 3-of-6 read quorum can still be available.

This distinction is crucial:

**The system may temporarily stop accepting writes while still retaining enough information to read and repair the data.**

The goal is not to promise that every operation continues under every failure. It is to preserve data and recoverability under the specified failure model.

## 7. Durability and availability are different

These terms are often mixed up.

### Durability

If the database acknowledges a committed write, the data should not be lost under the failures the system promises to tolerate.

### Availability

The system can continue accepting and serving requests.

A system can preserve data but temporarily reject writes. That is a durability-preserving loss of write availability.

A system can also be available but unsafe if it acknowledges writes without enough durable copies.

### A useful way to think

| Situation | Data safety | Can accept writes? |
|---|---|---|
| Healthy system | Protected | Yes |
| One AZ unavailable | Protected under the design objective | Yes, with 4 surviving copies |
| One AZ + one additional copy unavailable | Designed to retain read/recovery quorum | Not with a 4-of-6 write quorum |
| Too many copies unavailable to form read quorum | Cannot prove safe recovery from available copies | No safe progress |

Never describe a system as simply “highly available” without stating the failure scenario and the operations that remain available.

## 8. What does an Aurora write look like?

At a high level:

1. The writer DB instance executes a transaction and generates redo records.
2. It sends the redo records to the six storage copies in parallel.
3. Storage copies persist the records and acknowledge progress.
4. Once four of the six copies have acknowledged, the required write quorum has been reached.
5. The database can acknowledge the write as durable.
6. The remaining copies can catch up asynchronously.

The key is that the foreground write does not need to wait for all six copies.

### Why this helps latency

Imagine six replicas with response times:

```text
Replica A:  4 ms
Replica B:  5 ms
Replica C:  6 ms
Replica D:  7 ms
Replica E: 45 ms
Replica F: unavailable
```

The fourth acknowledgement arrives at 7 ms. The write can reach its quorum without waiting for the 45 ms replica or the unavailable replica.

This reduces the impact of **tail latency**: the slowest responses in a distributed request.

It does not make network latency disappear. It makes progress depend on a quorum rather than every replica.

## 9. Tail latency and stragglers

In distributed systems, the average response time can look healthy while a small percentage of requests take much longer.

Those slow requests are often called tail-latency events or stragglers.

If a write waits for all six replicas, one slow replica can dominate the response time.

If the write waits for four acknowledgements, the system can make progress when four replicas respond, provided the quorum and safety rules are satisfied.

This is a general design pattern:

- Fan out work to multiple independent participants.
- Wait for the required subset.
- Let slower participants catch up.
- Track lag and repair missing work.

The same reasoning appears in replicated caches, distributed metadata stores, object storage, and consensus-related systems. The exact correctness protocol differs by system.

## 10. Why redo records matter to quorum writes

A quorum write can be expensive if every replica must receive a full database page for every small update.

Aurora reduces this cost by replicating redo records rather than shipping complete modified pages as the normal write unit.

Benefits:

- Redo records are generally smaller than full data pages.
- The storage layer can acknowledge durable log progress before all page materialization is complete.
- Page reconstruction can happen asynchronously.
- The write path can be optimized around durable log persistence.

This is an important combination:

**Quorum replication provides fault tolerance; redo-based replication helps make the quorum write path efficient.**

The two ideas solve different problems and work together.

## 11. Why six replicas are not enough by themselves

A design can have six copies and still be unsafe if failures are correlated or repair is too slow.

Consider:

1. One storage node fails.
2. The system detects it and begins repair.
3. Before repair completes, an entire AZ fails.
4. Another independent failure occurs.

The risk depends on how long the system remains in a degraded state.

This brings us to two reliability concepts.

### MTTF: Mean Time To Failure

The average time between failures.

A higher MTTF generally means failures happen less often.

### MTTR: Mean Time To Repair

The average time needed to restore a failed component or return the system to its intended redundancy level.

A lower MTTR means the system spends less time exposed to additional failures.

A resilient distributed system must reason about both:

- Reduce the probability of failures.
- Reduce the time spent in a degraded state.

In large fleets, some failures are normal. The system must be designed to detect, isolate, and repair them continuously.

## 12. Segmented storage: reduce the size of the repair problem

Aurora divides the database volume into small segments. The design paper describes segments of up to about 10 GB, each protected independently by a six-copy protection group.

Why is this useful?

Imagine a database volume of many terabytes. If one small part becomes unhealthy, repairing the entire database volume would be wasteful.

With segmentation:

- Each segment has its own replica group.
- Failures can be isolated to a smaller unit.
- Repair transfers less data.
- Many independent segments can be repaired in parallel.
- A small failure does not require rebuilding the whole database.

This is a general distributed-storage principle:

**Make the unit of repair small enough that restoring redundancy is fast and operationally manageable.**

### Protection groups

A protection group is the set of replicas responsible for protecting one segment.

For the basic Aurora design discussed here:

- 6 replicas per protection group.
- 3 AZs.
- 2 copies per AZ.
- 4 acknowledgements for a write quorum.
- 3 copies for a read quorum.

The quorum is applied per protection group/segment, not as one giant vote across the entire database volume.

## 13. Full segments and tail segments

The Aurora design also describes using different kinds of segment replicas to manage storage cost.

Conceptually:

- **Full segments** hold both redo log records and materialized data blocks.
- **Tail segments** hold redo log records without maintaining the same full materialized data-block representation.

Why make replicas different?

Data pages usually consume much more storage than the redo log history needed to describe changes. Keeping every replica as a fully materialized copy would be expensive.

By using a mix of full and tail segments, Aurora can preserve the log replication and quorum properties while reducing the cost of storing complete page representations everywhere.

A useful way to remember it:

- Redo log copies help preserve the history of changes.
- Full segments also maintain materialized data blocks for efficient reads.
- Tail segments avoid the full cost of materializing all data blocks.

Do not overgeneralize this into “some replicas are disposable” or “tail replicas are not durable.” They participate in the protection design. The exact read/write selection rules account for the different replica roles.

## 14. Repair is part of the design, not an afterthought

When a replica is unavailable, the system must decide whether it is:

- Temporarily slow.
- Temporarily disconnected.
- Permanently failed.
- Returning after an outage and needing to catch up.

The system needs mechanisms to:

1. Detect missing or delayed progress.
2. Identify which log records a replica is missing.
3. Obtain those records from healthy peers.
4. Rebuild the replica's state.
5. Restore the protection group's redundancy.
6. Avoid declaring a healthy but slow node permanently dead too early.

This is difficult because false failure detection can create unnecessary membership changes and repair work.

### Repair versus failover

These are different operations:

- **Failover** moves application traffic to another compute instance.
- **Replica repair** restores missing storage redundancy.
- **Catch-up** brings a lagging replica up to date.
- **Membership change** changes which replicas are considered part of the protection group.

A Staff Engineer should keep these failure-handling paths separate when designing and debugging a distributed system.

## 15. Quorum is not the same as consensus

This distinction matters in system design interviews and real architecture work.

### Quorum

A quorum defines how many replicas must participate in an operation and provides overlap properties between read and write sets.

### Consensus

A consensus protocol helps distributed nodes agree on a value or ordered sequence of decisions despite failures, according to a formal protocol.

Quorum intersection alone does not solve every problem consensus solves.

For example, a production system may still need to answer:

- Who is allowed to be the writer?
- How are concurrent writes ordered?
- How are conflicting updates resolved?
- How does membership change safely?
- How do nodes agree on the current configuration?

Aurora's design is notable for using database techniques such as logging, ordering, and quorum sets to avoid placing a heavyweight distributed consensus round on every I/O operation. It does not mean Aurora has no coordination or membership protocol.

**Interview-safe phrasing:** Quorum intersection is a correctness building block. It is not, by itself, a complete consensus algorithm.

## 16. Common mistakes and corrections

| Mistake | Correct understanding |
|---|---|
| “A quorum means most replicas.” | A quorum is a set size chosen to satisfy specific intersection and failure requirements. |
| “4 of 6 means any four replicas are always enough for every operation.” | The operation, replica roles, log progress, and protocol rules matter. |
| “If the system cannot write, data is lost.” | It may retain a read/recovery quorum while lacking a write quorum. |
| “If a write gets four acknowledgements, all six are immediately up to date.” | The other replicas may catch up asynchronously. |
| “Three copies across three AZs are always enough.” | It depends on the failure objectives and quorum sizes. |
| “Quorum automatically solves concurrent writes.” | Ordering, versions, and conflict rules are still required. |
| “Six copies guarantee safety under any failure.” | No finite replica count survives every possible correlated failure. |
| “MTTF is all that matters.” | Repair time (MTTR) determines how long the system remains exposed. |
| “A quorum and consensus are synonyms.” | They are related concepts, but not equivalent. |
| “Repair is just copying a disk.” | Repair requires tracking progress, selecting valid sources, restoring membership, and preserving safety. |

## 17. Staff Engineer thinking: design from failure objectives

When designing a replicated storage system, do not start with “let's use three replicas” or “let's use five replicas.”

Start with the questions below.

### Step 1: State the guarantee

Examples:

- Survive one machine failure without losing acknowledged writes.
- Survive one AZ failure and continue accepting writes.
- Survive one AZ failure plus one additional node failure without losing data.
- Preserve read access during a maintenance event.

These are different guarantees.

### Step 2: Define failure domains

List what can fail together:

- Process
- Host
- Rack
- AZ
- Region
- Network partition
- Shared control-plane dependency

Do not assume failures are independent when they share power, networking, software, or physical infrastructure.

### Step 3: Choose replica placement

Decide how many copies live in each independent failure domain.

Placement often matters as much as total replica count.

### Step 4: Derive quorum sizes

Use the intersection conditions:

[
V_r + V_w > V
]

[
2V_w > V
]

Then test every promised failure scenario against the actual placement.

### Step 5: Define acknowledgement semantics

What exactly does a successful write acknowledgement mean?

- Received in memory?
- Written to local disk?
- Persisted on a quorum?
- Replicated across failure domains?
- Replicated to every copy?

The client contract must match the actual durability guarantee.

### Step 6: Design repair before launch

Ask:

- How is a failed replica detected?
- How is a slow replica distinguished from a dead one?
- Where does repair data come from?
- How much bandwidth does repair consume?
- How quickly can redundancy be restored?
- Can repair overload healthy replicas?
- What happens if another failure occurs during repair?

### Step 7: Measure the right things

Useful metrics include:

- Write quorum acknowledgement latency (p50, p95, p99).
- Number of replicas acknowledging each write.
- Replica lag / log sequence progress.
- Time spent in degraded protection groups.
- Repair duration and repair throughput.
- Number of protection groups awaiting repair.
- AZ-level failure and latency signals.
- Failed quorum attempts.
- Storage network saturation.

Metrics should answer whether the system is safe, whether it can make progress, and how long it takes to return to full redundancy.

## 18. Apply the idea to a new design

Suppose you are designing a distributed metadata service.

You have 5 replicas across 3 AZs. You want to survive one AZ failure and tolerate a slow replica.

Before selecting quorum sizes, answer:

1. How many replicas are placed in each AZ?
2. Which AZ failure removes the most replicas?
3. How many replicas remain after that failure?
4. Can a write quorum still be formed?
5. Does every read quorum intersect every acknowledged write quorum?
6. Do any two write quorums intersect?
7. How are writes ordered?
8. How does a returning replica catch up?
9. How quickly is redundancy restored?

Do not copy Aurora's 4-of-6 configuration into a different system without re-deriving the math. The right quorum depends on replica count, placement, operation semantics, and failure goals.

## 19. Quick revision

- **Quorum:** Minimum replica set needed for an operation.
- **Read/write intersection:** (V_r + V_w > V).
- **Write/write intersection:** (2V_w > V).
- **Aurora layout:** 6 copies across 3 AZs, 2 per AZ.
- **Aurora write quorum:** 4 of 6.
- **Aurora read quorum:** 3 of 6.
- **AZ failure:** Removes two copies; four remain, enough for a write quorum.
- **AZ + one additional copy failure:** Three remain; read/recovery quorum may remain, but a 4-copy write quorum does not.
- **Tail latency:** Waiting for every replica makes the slowest replica control request latency.
- **Redo replication:** Replicates change records rather than repeatedly shipping whole pages.
- **Segmentation:** Limits the size of an individual repair operation.
- **MTTF:** Mean time between failures.
- **MTTR:** Mean time to repair.
- **Full segment:** Maintains redo and materialized data blocks.
- **Tail segment:** Maintains redo without the same full data-block representation.
- **Staff mindset:** Define failure goals, model correlated failures, derive quorum math, define acknowledgement semantics, and engineer repair.

## 20. Questions to test your understanding

1. Why must a read quorum intersect a write quorum?
2. Why must two write quorums intersect?
3. Prove that 3-of-6 reads and 4-of-6 writes satisfy the intersection rules.
4. Why does Aurora place two copies in each of three AZs?
5. Why can Aurora still form a write quorum after losing one AZ?
6. What changes after losing one AZ and one additional storage copy?
7. How can a system retain data safety while temporarily losing write availability?
8. Why does waiting for all replicas increase tail latency?
9. Why is replica repair time part of the durability design?
10. What is the difference between quorum and consensus?
11. Why divide a large volume into independently protected segments?
12. Why might full and tail segments have different storage costs?
13. What metrics would tell you that repair is falling behind?
14. How would you derive quorum sizes for five replicas across three AZs?

---

## References

- [Arpit Bhayani: Durability with Quorum Writes video](https://youtu.be/EaWlpn24eAs)
- [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases (SIGMOD 2017)](https://www.cs.purdue.edu/homes/bb/cs542-22Spr/readings/impl/sigmod-17-amazon-aurora-design.pdf)
- [AWS: Aurora under the hood: quorums and correlated failure](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-quorum-and-correlated-failure/)
- [AWS: Aurora under the hood: quorum reads and mutating state](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-quorum-reads-and-mutating-state/)
- [Amazon Aurora: On Avoiding Distributed Consensus for I/Os, Commits, and Membership Changes (SIGMOD 2018)](https://www.eecs.umich.edu/courses/cse584/static_files/papers/aurora-sigmod-18.pdf)

> **Scope note:** This is a concept-focused study guide for the video topic, expanded with the Aurora design papers and AWS engineering material. It is not a verbatim transcript.
