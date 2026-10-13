# Kafka Notes

*Study notes from our session · October 14, 2026*

## Table of Contents

1. [The Big Idea](#1-the-big-idea)
2. [Core Building Blocks](#2-core-building-blocks)
3. [How Data Is Stored on Disk](#3-how-data-is-stored-on-disk)
4. [Producers](#4-producers)
5. [Replication](#5-replication-how-kafka-survives-crashes)
6. [Consumers and Consumer Groups](#6-consumers-and-consumer-groups)
7. [Offsets: Who Tracks Them?](#7-offsets-who-tracks-them)
8. [Can Multiple Consumers Read the Same Partition?](#8-can-multiple-consumers-read-the-same-partition)
9. [The Cluster Manager (Controller)](#9-the-cluster-manager-controller)
10. [How Long Data Stays](#10-how-long-data-stays)
11. [Delivery Guarantees](#11-delivery-guarantees)
12. [The Journey of One Message](#12-the-journey-of-one-message)
13. [Why Kafka Is Fast](#13-why-kafka-is-fast)
14. [When to Use Kafka](#14-when-to-use-kafka)
15. [Quick Revision Cheat Sheet](#15-quick-revision-cheat-sheet)

---

## 1. The Big Idea

Kafka is a **distributed, append-only log**.

**Analogy:** A shared notebook. Writers can only add new lines at the end and never change old lines. Each reader keeps their own bookmark, so many readers can read the same notebook at their own speed.

**The problem it solves:** An online store has an order system. When an order is placed, the payment app, email app, stock app, and analytics app all need to know. If the order system talks to each of them directly, it becomes messy, and one slow or broken app hurts everyone.

**How Kafka helps:** The order system writes each order into Kafka. Every other app reads from Kafka at its own speed. Messages are not deleted when read, so a slow or crashed app can come back later and continue where it stopped.

---

## 2. Core Building Blocks

| Part | What it is | Analogy |
|---|---|---|
| **Broker** | One Kafka server. A cluster is many brokers working together. | A cupboard that stores notebooks |
| **Topic** | A named stream of messages, like `orders` or `payments`. | The notebook's name |
| **Partition** | A topic is split into partitions. Each partition is one ordered log. This is how Kafka scales. | Splitting one notebook into several |
| **Offset** | The number of a message inside a partition (0, 1, 2...). | The line number, or the bookmark position |
| **Producer** | A program that writes messages. | The writer |
| **Consumer** | A program that reads messages. | The reader |

**Order rule:** Messages are in order *inside one partition*, not across the whole topic.

**Why partitions:** One log can only be written and read by so many clients at once. With 3 partitions, 3 writers and 3 readers can work in parallel.

---

## 3. How Data Is Stored on Disk

- Each partition is a folder of **segment files**. Messages are only added to the end of the newest segment. When it gets big, a new segment starts.
- Each segment has a small **index file**, so Kafka can jump to any offset quickly.
- Writing at the end of a file is **sequential disk writing**, which is very fast, even on normal disks.
- Kafka leans on the operating system's **page cache** (disk data kept in memory) instead of building its own cache.
- When sending data to consumers, Kafka uses **zero-copy**: the OS sends bytes from the file straight to the network, skipping extra copies.

---

## 4. Producers

### 4.1 Choosing a partition
- If the message has a **key** (like a customer ID), Kafka hashes the key, so the same key always goes to the same partition.
- That keeps the order for that key (for example, all events for one customer stay in order).

### 4.2 Batching and compression
- The producer collects several messages and sends them together as a batch, compressed.
- This is like posting one parcel instead of ten letters. It saves network and disk work.

### 4.3 How sure do you want to be? (`acks`)

| `acks` | Meaning | Trade-off |
|---|---|---|
| `0` | Don't wait for any reply | Fastest, can lose data |
| `1` | Wait for the leader only | Balanced. Data can be lost if the leader dies right away. |
| `all` | Wait for all in-sync copies | Safest, a bit slower |

---

## 5. Replication (How Kafka Survives Crashes)

**Problem:** If a server dies, its partitions are lost.
**Solution:** Each partition has copies on other brokers.

| Term | Meaning |
|---|---|
| **Leader** | The main copy. Producers and consumers talk to it. |
| **Follower** | A backup copy that keeps copying from the leader. |
| **ISR (in-sync replicas)** | The group of copies that are fully caught up. |
| **High watermark** | The newest offset that all in-sync copies have. Consumers only see messages up to here, so they never read something that could still be lost. |
| **`min.insync.replicas`** | How many copies must have the message before a write is accepted when `acks=all`. |

**If the leader dies:** a follower from the ISR becomes the new leader.

---

## 6. Consumers and Consumer Groups

- A **consumer group** is a team that shares the reading work.
- Kafka gives each partition to **only one consumer in the group** at a time.
- 6 partitions and 3 consumers means each consumer gets 2 partitions.
- To read faster, add consumers. But **more consumers than partitions** means some sit idle.
- **Different groups** read the same topic independently. For example, one group sends emails and another updates analytics.
- If a consumer crashes, another consumer in the group continues from the group's saved bookmark.

### Rebalance
When a consumer joins or leaves a group, the partitions are shared out again. Reading pauses for a short time during this, so teams try to avoid frequent rebalances.

### Example
The `orders` topic has 3 partitions.

| Group | Consumers | What happens |
|---|---|---|
| `email-service` | 3 | Each consumer gets 1 partition and they split the work |
| `analytics` | 1 | The one consumer reads all 3 partitions, with its own bookmarks |

Both groups read the same messages without affecting each other.

---

## 7. Offsets: Who Tracks Them?

**Usually the Kafka client library does it for you.** The consumer remembers how far it has read and saves that bookmark into Kafka's internal topic `__consumer_offsets`. By default it saves automatically every few seconds (`enable.auto.commit`).

**Important detail:** The bookmark is kept **per consumer group, per partition**, not per individual consumer.

### What developers decide: when to save the bookmark

| Choice | What happens | Risk |
|---|---|---|
| **Auto commit** (default) | Bookmark is saved on a timer, whether or not processing finished | If you crash mid-work, a message may be skipped (lost) |
| **Manual commit after processing** | You save the bookmark only after the work is done | If you crash before saving, the message is read again (duplicate), so the work should be safe to repeat |
| **Store offsets in your own database** | Save the result and the offset together in one database transaction | Most control, more work. Used when you need exactly-once results |

Devs can also move the bookmark on purpose (`seek`), for example to replay the last 2 days of events after fixing a bug.

---

## 8. Can Multiple Consumers Read the Same Partition?

**Yes, but only if they are in different consumer groups.**

- **Different groups:** Each group has its own bookmarks, so many apps can read the same partition at the same time, each at its own speed. This works because reading does not change the log. Nothing is removed when it is read, so readers don't affect each other.
- **Same group:** Kafka gives each partition to only one consumer at a time, so two consumers in the same group will not read the same partition together.

**The exception:** A developer can skip the group system and assign partitions by hand (`assign()`). Then two consumers can read the same partition even in one group. But Kafka no longer coordinates them, and they could overwrite each other's saved bookmarks unless the developer manages that. This is rarely done.

**Rule of thumb:**
- To have several readers **see the same messages**, use **different groups**.
- To **share the work** of reading, use **the same group**.

---

## 9. The Cluster Manager (Controller)

- One broker acts as the **controller**. It tracks which brokers are alive and picks new leaders when one fails.
- **Older Kafka** used a separate tool called **ZooKeeper** for this.
- **Newer Kafka** uses **KRaft**, a built-in system where some brokers keep the cluster information themselves.
- ZooKeeper is removed in Kafka 4.0 and later.

---

## 10. How Long Data Stays

Kafka does **not** delete a message just because it was read. It deletes by rule:

| Rule | Meaning |
|---|---|
| **Time or size limit** | For example, keep 7 days, then delete the oldest segments. |
| **Log compaction** | Keep only the *latest* message for each key. Good for 'current state' topics, like a user's latest profile. |

---

## 11. Delivery Guarantees

| Guarantee | Plain meaning |
|---|---|
| **At-least-once** (usual default) | A message is never lost, but a retry can cause a duplicate. |
| **At-most-once** | No duplicates, but a message can be lost. |
| **Exactly-once** | Not lost and not repeated. Uses an **idempotent producer** (Kafka ignores repeated sends from the same producer) plus **transactions** (a group of writes either all happen or none happen). |

---

## 12. The Journey of One Message

1. The producer picks a partition by key and puts the message in a batch.
2. The batch goes to that partition's **leader**, which adds it to the end of the log.
3. Followers copy it. Once the in-sync copies have it, the leader replies with success (based on `acks`).
4. The **high watermark** moves up, so consumers can now see the message.
5. A consumer in a group reads it from its offset, processes it, and saves the new offset.

---

## 13. Why Kafka Is Fast

- Sequential disk writes (only adds at the end)
- The OS page cache
- Zero-copy sending
- Batching and compression
- Partitions spread the work across many servers

---

## 14. When to Use Kafka

- Many apps need to react to the same events (orders, clicks, payments).
- You need to handle a very large flow of messages.
- You want a slow or crashed app to catch up later without losing anything.
- You want to replay old events, for example to rebuild data after a bug.

---

## 15. Quick Revision Cheat Sheet

| Question | Short answer |
|---|---|
| What is Kafka? | A distributed, append-only log |
| What gives Kafka scale? | Partitions |
| Where is order guaranteed? | Inside one partition only |
| How to keep one customer's events in order? | Use the customer ID as the message key |
| What is an offset? | A message's position number inside a partition |
| Who stores consumer bookmarks? | Kafka itself, in `__consumer_offsets`, per group per partition |
| Who tracks offsets in code? | The client library, but developers choose when to commit |
| Safest `acks` setting? | `all` |
| What is the ISR? | The copies that are fully caught up with the leader |
| What can consumers read? | Only up to the high watermark |
| Two consumers, same partition? | Only if they are in different groups |
| More consumers than partitions? | The extra consumers sit idle |
| What is a rebalance? | Partitions being shared out again after a consumer joins or leaves |
| Is a message deleted after being read? | No. It is deleted by time, size, or compaction rules |
| Default delivery guarantee? | At-least-once |
| How to get exactly-once? | Idempotent producer plus transactions |
| What replaced ZooKeeper? | KRaft |
