# Redis System Design Notes

*Study notes from our session · October 14, 2026*

These notes cover how Redis handles many connections, the common system design problems it solves, how to build a distributed lock, and the main rate limiting strategies.

## 1. How Redis Handles Many Connections

Redis uses **I/O multiplexing**. One thread watches thousands of client connections at the same time, instead of using one thread for each connection.

- Redis registers all client sockets with the operating system (`epoll` on Linux, `kqueue` on macOS/BSD, with `select`/`poll` as backups).
- The main loop asks the OS: 'Which sockets are ready right now?' The OS returns only the active ones.
- Redis reads each request, runs the command, queues the reply, then asks again.
- Commands run in memory and finish in microseconds, so one thread is enough.
- With no thread per connection, there are no locks, no context switching, and no race conditions on the data. Every command is atomic.
- Since Redis 6, optional I/O threads can help with reading and writing sockets, but commands still run on the single main thread.

## 2. System Design Problems and How Redis Solves Them

### 2.1 Too many database reads

**Scenario:** A shopping site shows the same product page to thousands of people. Each visit asks the database the same question, and the database gets slow.

**How Redis solves it:** Keep a copy of the answer in memory (this is called cache-aside). The first request goes to the database and the answer is saved in Redis. Later requests read it from Redis, which is much faster. Set a timer (TTL) so old copies are deleted. When the data changes, delete or update the copy.

### 2.2 A popular item expires and everyone hits the database (cache stampede)

**Scenario:** The home page data expires at 10:00:00. At that exact second, 5,000 users ask for it, and all 5,000 requests go to the database.

**How Redis solves it:** Use a lock for that item. The first request gets the lock and fetches fresh data. The others wait a moment, then read the new copy from Redis. Also add a small random extra time to each expiry so items don't all expire together.

### 2.3 Requests for things that don't exist (cache penetration)

**Scenario:** Someone keeps asking for product ID 99999999, which doesn't exist. Every request goes to the database and finds nothing.

**How Redis solves it:** Save the answer 'not found' for a short time. You can also add a quick pre-check (a Bloom filter) that says 'this definitely doesn't exist' before the request reaches the database.

### 2.4 One very popular key overloads one server (hot key)

**Scenario:** A celebrity posts something and millions of people read it. The one Redis server holding it gets overloaded.

**How Redis solves it:** Save the same post under several names (like `post:1:a`, `post:1:b`, `post:1:c`), possibly on different servers. Each reader picks one at random, so the load is spread out. You can also keep a small copy in each app server's own memory.

### 2.5 Stopping people from sending too many requests

**Scenario:** You want to allow only 100 requests per minute per user.

**How Redis solves it:** Keep a counter per user, add 1 for each request, and reject when it goes over the limit. See Section 4 for all the strategies.

### 2.6 Game leaderboard

**Scenario:** A game has 10 million players. You want to show the top 10 and each player's rank, and scores change every second.

**How Redis solves it:** Use a **sorted set**, a list that always stays in order by score. `ZADD` updates a score, `ZREVRANGE` gets the top N, and `ZRANK` gets one player's position. All of these are fast.

### 2.7 Making sure only one server does a job

**Scenario:** You have 5 servers and each one tries to send the monthly invoice emails. Customers would get 5 copies.

**How Redis solves it:** A distributed lock. Only one server can create the lock key. See Section 3.

### 2.8 Keeping users logged in across many servers

**Scenario:** A user logs in through server A, but the next click goes to server B, and server B doesn't know who they are.

**How Redis solves it:** Save the session data in Redis under a session ID, with a timer. Any server can look up that ID. The session expires on its own if the user is inactive.

### 2.9 Doing slow work in the background (job queue)

**Scenario:** A user uploads a video. Converting it takes 5 minutes, and you don't want the user to wait on the page.

**How Redis solves it:** The website pushes a task into a Redis list (`LPUSH`). Worker programs take tasks from the other end (`BRPOP`). For tasks that must not be lost, use **Redis Streams** with consumer groups. A worker confirms when a task is done, and a failed task can be tried again.

### 2.10 Live chat or notifications

**Scenario:** When one user sends a chat message, everyone in the room should see it immediately.

**How Redis solves it:** **Pub/Sub**. Users subscribe to a channel, and a message published to it goes instantly to all subscribers. Redis does not save the message, so offline users miss it. If they need it later, use Streams or also save the messages in a database.

### 2.11 Counting unique visitors

**Scenario:** You want to know how many different people visited today. There are 50 million visits, and storing every user ID would use a lot of memory.

**How Redis solves it:** **HyperLogLog** (`PFADD`, `PFCOUNT`) gives a very close estimate using only about 12 KB of memory, with about 1% error. For exact numbers, use a **bitmap** (`SETBIT`, `BITCOUNT`), where each user is one bit.

### 2.12 Finding nearby places

**Scenario:** A ride app needs to find all drivers within 3 km of a customer.

**How Redis solves it:** Use the built-in location commands. `GEOADD` saves each driver's latitude and longitude. `GEOSEARCH` returns everything within a given distance of a point.

### 2.13 Social media feed

**Scenario:** When you open the app, you want to see the latest posts from people you follow, and it must load fast.

**How Redis solves it:** When someone posts, add the post ID to each follower's feed in Redis (**fan-out on write**). When a follower opens the app, Redis gives the ready-made list of IDs and the app loads the post details. For accounts with millions of followers this is too much work, so collect their posts only when a follower opens the app (**fan-out on read**). Using both is called a hybrid model.

### 2.14 Stopping duplicate actions

**Scenario:** A user clicks 'Pay' twice because the page was slow, and gets charged twice.

**How Redis solves it:** Each request has a unique ID. Before doing the work, run `SET request_id NX EX ttl`. If the key was already there, it is a repeat, so return the earlier result instead of doing the work again.

### 2.15 Keeping cache and database in agreement

**Scenario:** A product's price changes in the database, but Redis still shows the old price.

**How Redis solves it:** Update the database first, then delete the product's key in Redis. The next read finds no copy, loads the new price from the database, and saves it. For stronger agreement, send an event every time the database changes and use it to delete the cached copy.

### 2.16 Redis crashes or runs out of space

**Scenario:** Your Redis server fails, or your data becomes too big for one machine.

**How Redis solves it:**

- **Replicas** keep copies of the data on other servers.
- **Sentinel** notices when the main server is down and promotes a replica.
- **Redis Cluster** splits the data across several servers using 16,384 hash slots.
- **RDB snapshots and AOF** save data to disk so it isn't lost on restart.

## 3. Distributed Locking with Redis

### The idea

Create a key in Redis only if it doesn't already exist. Whoever creates it owns the lock. When the job is done, delete the key.

### Step 1: Take the lock

```
SET lock:invoice-job <unique-id> NX PX 30000
```

| Part | Meaning |
| --- | --- |
| `NX` | Only set the key if it does not exist yet. Only one client can win. |
| `PX 30000` | Delete the key after 30 seconds. If the owner crashes, the lock frees itself. |
| `<unique-id>` | A random value (like a UUID) that only you know. It proves the lock is yours. |

Redis replies `OK` if you got the lock, or `nil` if someone else has it.

### Step 2: Do the work

Run the job only if you got `OK`. If you got `nil`, give up or wait a short random time and try again.

### Step 3: Release the lock safely

Don't just call `DEL`. Delete the key only if the value is still your unique ID, and do the check and the delete together in one Lua script:

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
```

**Why this matters:** If your job takes 40 seconds but the lock timer was 30 seconds, the lock expires and another server takes it. A plain `DEL` would then delete their lock. The ID check prevents that.

### Analogy: a whiteboard on a meeting room door

- Redis is the whiteboard. Writing your name claims the room.
- `NX` means you can only write if the board is empty.
- `PX` means the ink fades after 30 seconds, so a crashed owner never locks the room forever.
- The unique ID is your secret name tag, so you only wipe your own note.

### Problems to know

1. **The job takes longer than the lock timer.** Pick a longer timer, or use a watchdog that extends the timer while the job is still running (it must check your ID first).
2. **Redis fails over at a bad moment.** If the main server dies right after you took the lock, and the backup had not received it yet, a second client can take the same lock. This is fine for most jobs but not for things that must never overlap.
3. **Stronger protection.**
   - **Redlock:** take the lock on a majority of 5 separate Redis servers (3 out of 5).
   - **Fencing token:** hand out an increasing number with each lock. The service you write to rejects any request with an older number.
4. **Waiting for the lock.** Wait a small random time between retries so clients don't all retry at once.

### In real projects: use Redisson (Java)

```java
RLock lock = redisson.getLock("lock:invoice-job");
if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {   // wait up to 5s, hold up to 30s
    try {
        sendInvoices();
    } finally {
        lock.unlock();
    }
}
```

Redisson also extends the timer for you automatically.

### When not to use Redis for locking

If a wrong answer would be very costly and you can't use fencing tokens, use ZooKeeper, etcd, or a database row lock. Redis locks are great for 'usually only one at a time' jobs, but they are not a perfect guarantee.

## 4. Rate Limiting Strategies

Rate limiting controls how many requests a user can make in a given time, for example 100 per minute. It stops spam, protects servers, and keeps things fair.

### 4.1 Fixed window counter

**Analogy:** A shop gives each customer 100 tokens per hour, and the count resets exactly on the hour.

**How it works:** Keep one counter per user per time slot. Each request adds 1. If the counter goes above the limit, reject the request. When the slot ends, the counter starts over.

**Redis:** `INCR` the counter and set an expiry of 60 seconds on it.

- **Good:** Very simple and fast.
- **Problem:** A user can send a burst at the edge. With a limit of 100 per minute, they can send 100 at 10:00:59 and another 100 at 10:01:00, which is 200 requests in 2 seconds.

### 4.2 Sliding window log

**Analogy:** A guard writes down the exact time of every visit. For a new visitor, he looks back at the last 60 seconds in his notebook and counts.

**How it works:** Save the time of every request. On a new request, remove entries older than one minute, then count the rest. If the count is under the limit, allow it and save its time.

**Redis:** A sorted set with timestamps as scores.

- **Good:** Very accurate, with no edge bursts.
- **Problem:** Uses more memory, since it stores every request.

### 4.3 Sliding window counter

**Analogy:** Instead of writing every visit down, the guard keeps only two totals: last minute's count and this minute's count, and estimates the last 60 seconds from them.

**How it works:** Keep counters for the current and previous time slots. Estimate the recent total by taking part of the previous slot's count (based on how far you are into the current slot) and adding the current count.

- **Good:** Almost as accurate as the log, with very little memory.
- **Problem:** It is an estimate, not an exact count.

### 4.4 Token bucket

**Analogy:** A bucket holds up to 10 tokens, and a new token drops in every second. Each request takes one token. If the bucket is empty, the request is rejected.

**How it works:** Store the number of tokens and the time of the last refill. On each request, add the tokens earned since last time, then take one.

**Redis:** A small Lua script, so the check and the update happen together.

- **Good:** Allows short bursts (up to the bucket size) while keeping the long-term speed under control. Very popular for APIs.
- **Problem:** Slightly more work to set up than a simple counter.

### 4.5 Leaky bucket

**Analogy:** A bucket with a small hole at the bottom. Water (requests) pours in at any speed but leaks out at one steady speed. If the bucket overflows, the extra water is thrown away.

**How it works:** Requests enter a queue and are processed at a fixed rate. If the queue is full, new requests are rejected.

- **Good:** The output is smooth and steady, which protects a server that can't handle spikes.
- **Problem:** Bursts are slowed down instead of allowed through, so users may wait.

### Comparison

| Strategy | Accuracy | Memory | Allows bursts? |
| --- | --- | --- | --- |
| Fixed window | Low (edge problem) | Very low | Accidentally, at the edges |
| Sliding window log | Highest | High | No |
| Sliding window counter | Good | Low | Slightly |
| Token bucket | Good | Low | Yes, on purpose |
| Leaky bucket | Good | Medium | No, it smooths them out |

### Which one to pick?

- **Simple and cheap:** fixed window.
- **Strict and exact:** sliding window log.
- **Good balance for most apps:** sliding window counter or token bucket.
- **Protecting a backend that needs a steady flow:** leaky bucket.

### One rule for all of them

Always do the check and the update **together**, using a Lua script or atomic commands like `INCR`. If you do them as two separate steps, two requests can slip through at the same moment.

## 5. Quick Revision Cheat Sheet

| Problem | Redis feature |
| --- | --- |
| Slow reads | Cache with TTL (cache-aside) |
| Stampede on expiry | Lock + random expiry time |
| Requests for missing data | Cache 'not found', Bloom filter |
| Hot key | Copies under several names, local cache |
| Rate limiting | `INCR` + expiry, sorted set, Lua script |
| Leaderboard | Sorted set (`ZADD`, `ZREVRANGE`, `ZRANK`) |
| One job at a time | `SET key id NX PX ttl` + Lua release |
| Login sessions | Key with TTL (or hash) per session ID |
| Background jobs | List (`LPUSH`/`BRPOP`) or Streams |
| Live messages | Pub/Sub (or Streams if durable) |
| Unique counts | HyperLogLog (estimate), bitmap (exact) |
| Nearby search | `GEOADD`, `GEOSEARCH` |
| News feed | Lists or sorted sets, hybrid fan-out |
| Duplicate requests | `SET request_id NX EX ttl` |
| Cache and DB match | Update DB, then delete cache key |
| Failures and big data | Replicas, Sentinel, Cluster, RDB/AOF |
