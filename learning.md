
# What is Redis?
- Redis is an in-memory key-value data store.
- Since data is stored in RAM, so latency is very low compared to traditional databases.
- It can be used for so many things like:
  - Caching
  - Pub/Sub
  - Rate Limiting
  - Session Storage
  - Leaderboards etc...
- Usually, there's a single Redis layer that is shared by all backend application servers in a typical web application architecture.
  - That single layer is logical. Inside that single logical layer, redis can scale. It's not like there's just one machine for Redis, there could be many that acts as one.

# Why is Redis fast?
 - Data is stored in RAM.
 - Redis stores data in a simple way (O(1) data structures) compared to tradition RDMS, which uses B/B+ trees (O(logn) data structures) to store the data, requires SQL parsing, joins etc to serve a query.
 - Redis is single-threaded, so no synchronisation & locking overhead. And it makes sense to keep it single-threaded because Redis is used as a single layer in a typical web architecture.

# So Why Not Store Everything In Redis?

- If we could, we would. But, RAM is expensive & have limited capacity compared to disk storage which is cheaper and have lots of capacity.
- Also, there is no relationship among data. Like RDBMS which allows us to create relationship among data by joining two tables, we cannot do that in Redis. So, its a disadvantage here.
- Since, Redis is Single-Threaded, so if that thread gets blocked by some reason, other requests cannot be completed until the thread unblocks.
- That’s why Redis is commonly used for:
  - hot data  
  - frequently accessed data
  - temporary data
  - performance-critical data
NOT usually as the primary permanent database for everything.

# Key-Value model
- Redis stores data like: `<key> -> <value>`.
- Keys are usually strings and values can be any data type provided by Redis, like Strings, JSON, Lists, Sets, Hashes, Sorted Sets etc...

# What is Redis not good at?
Redis is not ideal for:
- Complex Queries & Data Relationships
- While redis handles small data points (strings, numbers, small JSON blocks) incredibly well. It struggles heavily with large files like images, videos, or massive PDF blobs.
  - Because, Redis is in-memory and large files will take lots of RAM and things will become very expensive.
  - If we still decide to keep large data here, moving large data in Redis will definitely introduce latency, which defeats the purpose of Redis to be low latency/high speed layer.
- Since the engine of Redis is single-threaded, it can only execute one command at a time. So, if you end up running a command that takes a lot of time to process it, Redis will freeze and every other incoming request coming from application server will have to wait. 


# Data Types

Before we move to data structures, Redis stores everything as `key -> value`. Where `key` is a string. The `value` can be any data type provided by redis.

## 1. Strings
- It is the most basic data type in redis.
- `value` can be anything: text, integers, floating point numbers, or binary data (like images or serialized objects).

### Basic Commands

```
SET name "Alice"
GET name          # → "Alice"

SET age 25
GET age           # → "25"
```


```
EXPIRE session:123 3600   //  Key expires after 1 hour.
TTL session:123       // Returns remaining lifetime.
DEL session:123  // Deletes one or more key
```

Redis can treat strings as integers and do atomic math on them:
```
SET visits 0

INCR   visits       # → 1  (increment by 1)
INCR   visits       # → 2
INCRBY visits 10    # → 12
DECR   visits       # → 11
DECRBY visits 5     # → 6

# Float support
SET price 9.99
INCRBYFLOAT price 0.50   # → 10.49
```
Why "atomic" matters: Even with 1000 concurrent requests hitting INCR, Redis guarantees no lost updates — no race conditions. All of this because Redis is single-threaded.


### Counters
- Strings can be used for counters like page views.
- Key: `page:home:views`
- Increment: `INCR page:home:views`, `INCRBY page:home:views 5`

### OTP storage
- OTPs are temporary values. Redis is perfect because of TTL support.

```
SET otp:john@example.com 123456 EX 300  // Expires in 5 minutes
```

### When strings are not suitable
Suppose we store this as one big string:
```json
{
  "name": "Siddharth",
  "age": 24,
  "city": "Chennai"
}
```
Updating just one field would mean: Read the full object, update the desired field, and write the entire object back to redis. This is where Hashes makes more sense.

## Hashes

- A Redis Hash is a data type that stores a collection of field-value pairs under a single key.
- Think of it as: `one Redis key → multiple fields → multiple values`
```
Key: "user:42"
  ├── name    → "Alice"
  ├── email   → "alice@gmail.com"
  ├── age     → "28"
  └── city    → "Chennai"
```

### Basic Commands
```
# Set one or more fields
HSET user:42  name "Alice"  email "alice@gmail.com"  age 28  city "Chennai"

# Get a single field
HGET user:42 name        # → "Alice"
HGET user:42 age         # → "28"

# Get multiple fields
HMGET user:42 name email city
# → ["Alice", "alice@gmail.com", "Chennai"]

HGETALL user:42  // Get everything
```

- So hashes are good for object/JSON like structured data and partial updates.

### User / Session Profiles
```bash
HSET session:abc123
  userId   "42"
  role     "admin"
  browser  "Chrome"
  loginAt  "1716000000"

# Update just the last active time
HSET session:abc123 lastSeen "1716003600"
```

### Product Catalog
```bash
HSET product:101
  name     "Wireless Mouse"
  price    "799"
  stock    "200"
  category "Electronics"

# Decrement stock when purchased
HINCRBY product:101 stock -1
```

### Caching a DB row
```bash
# Cache a database row without serializing to JSON
HSET cache:order:5001
  status      "shipped"
  total       "1299"
  customerId  "42"
  createdAt   "2024-01-15"

# Read just the status without deserializing
HGET cache:order:5001 status
```

### Config/Feature flags
```bash
HSET app:config
  maintenance  "false"
  maxUploadMB  "10"
  theme        "dark"

HGET app:config maintenance    # → "false"
```

## Lists
- A Redis List is an ordered collection of strings, sorted by insertion order. Internally it's a doubly linked list, so adding elements to the head or tail is extremely fast (O(1)), regardless of list size.
- They are frequently used to:
  - Implement stacks & queues.
  -  Build queue management for background worker systems.
- We do not need to create an empty list (key) before pushing elements to it. It will be done automatically by Redis. Similary, we do not need to remove a list (key) explicitly if the list has 0 elements, redis will automatically delete the key and frees up the memory.

### Basic Commands

```bash
# Add to LEFT (head)
LPUSH tasks "task1"
LPUSH tasks "task2"
LPUSH tasks "task3"
# List: [task3, task2, task1]

# Add to RIGHT (tail)
RPUSH tasks "task4"
# List: [task3, task2, task1, task4]

# Push multiple at once
LPUSH notifications "notif1" "notif2" "notif3"
```
```bash
LPOP tasks          # → "task3"  (removes from head)
RPOP tasks          # → "task4"  (removes from tail)

# Pop multiple elements (Redis 6.2+)
LPOP tasks 2        # → ["task2", "task1"]
```

### Blocking Operations (VVIMP)
- This is mostly used in Producer-Consumer problem.
- The Problem: Busy-Waiting (Polling)
  - Without blocking operations, a consumer system must constantly "poll" Redis to see if new work has arrived.
  - ``` bash
    LPOP myqueue   # Returns (nil) — nothing there
    LPOP myqueue   # Returns (nil) — still nothing
    LPOP myqueue   # Returns (nil) — wasting CPU and connections
    ```
    In code, this will look like an infinite loop:
    ```
    while(true):
     check queue
    ```
  - This is called **busy-waiting** or **polling** (Keep checking the queue), and it wastes CPU cycles.
- Blocking operations solve this elegantly — the consumer simply waits until data arrives.

#### The Commands

```bash
BLPOP key [key ...] timeout
BRPOP key [key ...] timeout
```
- `BLPOP` — blocks and pops from the left (head)
- `BRPOP` — blocks and pops from the right (tail)
- `timeout` — how many seconds to wait before giving up (0 = wait forever)

#### Step-by-Step Execution Example

Terminal 1 (Consumer) - waits up to 30 secs
```bash
BLPOP myqueue 30
```
The consumer is now blocked and sleeping — no CPU used.


Terminal 2 (Producer) - pushes a job:
```bash
RPUSH myqueue "send-email"
```
Terminal 1 immediately wakes up and returns:
```bash
1) "myqueue"      # which list the element came from
2) "send-email"   # the element itself
```
Note: The response is always an array containing both the list name and the value. This matters because BLPOP can watch multiple lists at once.


#### Advanced Behavior: Watching multiple lists

This is where blocking operations get really useful. A single consumer can watch several queues at once and respond to whichever has data first. This allows us to build a **priority queue** system completely out of the box. (Remember: The priority queue for processes in OS, not the min/max heap one)

```bash
BLPOP high-priority normal-priority low-priority 0
```
**How Redis evaluates this:**
- Redis scans the keys from left to right (`high` -> `normal` -> `low`).
- If `high-priority` has elements, it pops from it immediately.
- If `high-priority` is empty but `normal-priority` has data, it pops from `normal`.
- If all lists are empty, the client blocks on all of them. The moment any of them receives data, the client wakes up and the first item is returned along with the name of the list it originated from.

#### Timeout Behaviour
- ```BLPOP myqueue 10``` If nothing arrives within 10 secs, redis returns `nil`.
- With `timeout = 0`, it waits indefinitely: `BLPOP myqueue 0` -> waits forever until something arrives.

### Use-Cases of lists
We can use lists whereever we need to maintain a queue or a stack. The following are some of the use cases:
- Producer-Consumer pattern
- Background jobs
- Notification systems
- Task Scheduling etc...

To describe a common use case step by step, imagine your home page shows the latest photos published in a photo sharing social network and you want to speedup access.
- Every time a user posts a new photo, we add its ID into a list with LPUSH.
- When users visit the home page, we use LRANGE 0 9 in order to get the latest 10 posted items.


## Sets
- It is an unordered collection of unique elements.
- Adding the same element twice has no effect.

### Basic Commands
Add members
```bash
SADD skills "python" "redis" "docker" "python"
# "python" added only once — duplicates ignored

SMEMBERS skills     # → {"python", "redis", "docker"}
SCARD skills        # → 3  (count of members)
```

Check membership
```bash
SISMEMBER skills "redis"    # → 1 (yes)
SISMEMBER skills "java"     # → 0 (no)
```

Remove members
```bash
SREM skills "docker"
SMEMBERS skills     # → {"python", "redis"}
```

### Set Operations

Let's consider the following sets:
```bash
SADD backend  "Alice" "Bob" "Carol" "Dave"
SADD frontend "Carol" "Dave" "Eve"  "Frank"
```

#### Intersection Operation (common members)
```bash
SINTER backend frontend
# → {"Carol", "Dave"}   ← work on BOTH teams
```

#### Union operation (all unique members)
```bash
SUNION backend frontend
# → {"Alice", "Bob", "Carol", "Dave", "Eve", "Frank"}
```

#### Difference operation (In first, not in others)
```bash
SDIFF backend frontend
# → {"Alice", "Bob"}    ← only in backend

SDIFF frontend backend
# → {"Eve", "Frank"}    ← only in frontend
```

### Real-World Use Cases

#### Online/Active users
```bash
# User comes online
SADD online:users "user:42"

# User goes offline
SREM online:users "user:42"

# Is user online?
SISMEMBER online:users "user:42"   # → 1 or 0

# How many online?
SCARD online:users
```

#### Tags on a post
```bash
SADD post:101:tags  "redis" "database" "backend"
SADD post:202:tags  "redis" "caching"  "performance"

# Common tag(s)
SINTER post:101:tags post:202:tags   # → {"redis"}

# All unique tags across posts
SUNION post:101:tags post:202:tags
# → {"redis", "database", "backend", "caching", "performance"}
```

#### Unique visitors
```bash
# Track unique visitors per day
SADD visitors:2024-01-15 "user:42" "user:17" "user:42"
# user:42 counted only once

SCARD visitors:2024-01-15    # → 2 unique visitors

# Visitors on both days (returning users)
SINTER visitors:2024-01-15 visitors:2024-01-16
```

#### Mutual Friends
```bash
SADD friends:alice  "bob" "carol" "dave"
SADD friends:bob    "alice" "carol" "eve"

# Mutual friends between Alice and Bob
SINTER friends:alice friends:bob    # → {"carol"}

# Suggestions for Alice (Bob's friends she doesn't know)
SDIFF friends:bob friends:alice     # → {"eve"}
```

#### Permission/Roles
```bash
SADD role:admin   "read" "write" "delete" "manage"
SADD role:editor  "read" "write"
SADD role:viewer  "read"

# What can admin do that editor can't?
SDIFF role:admin role:editor        # → {"delete", "manage"}

# Check if user has permission
SISMEMBER role:editor "delete"      # → 0 (not allowed)
```

#### Blacklist/Blocklist
```bash
SADD blocked:ips "192.168.1.50" "10.0.0.99"

# Check on every request
SISMEMBER blocked:ips "192.168.1.50"   # → 1 (block!)
SISMEMBER blocked:ips "192.168.1.1"    # → 0 (allow)
```


## Sorted Sets
- A set but with a score associated with each element which determines its order.
- Set stores: `(member, score)`. Redis automatically keeps elements sorted by score.
 
### Add elements
```bash
ZADD leaderboard 100 Alice
ZADD leaderboard 250 Bob
ZADD leaderboard 180 Charlie
```
Redis will automatically sort the elements internally:
```bash
Bob      250
Charlie  180
Alice    100
```
- Note: The elements are sorted in ascending order according to the score. So Alice will come first, then Charlie, then Bob.
 
### Retrieve elements
```bash
ZRANGE leaderboard 0 -1
```
Gets ordered members.

Reverse order: Most common for leaderboards.
```bash
ZREVRANGE leaderboard 0 9
```
Gets top 10 players

### Get Rank
```bash
ZRANK leaderboard Alice
```
Gets the rank, but neeche se :)

Reverse rank
```bash
ZREVRANK leaderboard Bob
```
Gets the rank, upar se :)


### Increment Score
```bash
ZINCRBY leaderboard 50 Alice
```


### Real-World Use Cases
1. Gaming Leaderboards
2. Trending Systems:
    - Example: Reddit, Youtube, Twitter trends.
    - Score can represent: likes, views, engagement
3. Priority Queues  

# Redis Persistence
- Redis is an in-memory data-store. So, if redis server crashes or restarts, does that mean all data in redis would be lost? Well, no. Redis does allow persistence.
- Persisting to disk allows redis to recover data after restart.

#### Persistence Mechanisms
1. RDB (Redis Database Snapshot)
2. AOF (Append Only File)

## RDB (Redis Database Snapshot)
- The idea is to take point-in-time snapshot of entire memory periodically.
- For example: Take a snapshot every 5 mins, or Take a snapshot after N write operations.
- This creates a single `.rdb` file, which is compact and easy to reload.
- How it works internally:
  - ```
    1. Redis calls fork() → creates a child process
    2. Child process writes snapshot to a temp `.rdb` file
    3. Main process keeps serving requests (no blocking!)
    4. Child finishes → atomically replaces old dump.rdb
    ```
- RDB creation should not be frequent operation, as it is a heavy operation. Otherwise our system would be busy in taking snapshot most of the time.

### Problems with RDB approach
- Possible Data Loss
- Suppose:
  - snapshot taken at 10:00
  - Redis crashes at 10:04
  - We will lose 4 mins of data.

## AOF (Append Only File)
- AOF logs every single **write** (not read) operation to a file and replays it upon server restart to recreate the dataset.

### `fsync` 
- Whenever Redis (or any program) writes data to disk, it doesn't go directly to the physical disk. Instead, the OS buffers the data in a temporary memory zone called the OS Page Cache.The OS flushes this buffer to disk at its own leisure, typically every 30 seconds.
```
Redis  →  OS Page Cache (RAM)  →  (eventually)  →  Physical Disk
```

- This creates a risk: If the Redis process crashes, your data is safe because the OS still holds it in memory and will eventually flush it to the disk. But if the entire server loses power or the OS crashes, that data in the cache is gone forever.
- This is where fsync comes in. `fsync()` is a system call that forces the Operating System to flush all the cached data from the page cache directly onto the physical disk.
- Redis allows us to configure how often `fsync()` is run by OS for our AOF file.

### `fsync` Policies:
1. always
    - Redis calls `fsync` after every single write command it receives.
    - Safest, but slowest.
2. everysec (default)
    - Flush once per second.
    - Good balance, but can lost ~ 1sec of data.
3. no
    - OS decides flush timing.
    - fastest, least safe

| Policy | Performance | Durability / Safety | Max Data Lost on Power Failure |
|---|---|---|---|
| always | Slow | Extremely High | A fraction of a second (1 write) |
| everysec | Fast (Near-Optimal) | High | 1–2 seconds |
| no | Fastest | Low | Up to 30 seconds (OS dependent) |

### AOF Compaction - Rewriting the log file
- Overtime, AOF file grows large. If a key is incremented a million times, the AOF will contain one million lines of log data.
- So, periodically, the entire AOF file is re-written for compaction.
- Example:
```bash
Original AOF:                    After Rewrite:
──────────────────────           ──────────────
SET counter 0                    SET counter 42
INCR counter      (42 times)     SET name "Alice"
INCR counter
...
SET name "Alice"
```
- This operation is called **BGREWRITEAOF**.
- We can trigger compaction (`BGREWRITEAOF`) manually or configure it to be done automatically.

- How REWRITE/COMPACTION happens in the background?
  - A background thread performs compaction to a temp AOF, and then renames it, replacing the old AOF file.
  - While the child process is busy in compaction, the main process is still accepting new traffic. Redis records these new incoming updates in an AOF Rewrite Buffer.
  - Once the child process finishes the base rewrite, Redis appends those buffered real-time updates to the end of the new file and atomically swaps the old, bloated AOF file out for the new compacted one.

 ### Problems with AOF
 - AOF files grow large overtime. Although we compact, but that also may grow large overtime.
 - AOF files are bigger than RDB.
 - Solution is to use a hybrid approach: RDB + AOF

## RDB + AOF (Hybrid approach)
- The idea is to take snapshot and write logs as well.
- Take snapshot every few minutes, and create AOF starting from the latest snapshot time only.

### Idea:
- Take a snapshot at 10:00 AM
- Start logging commands in AOF starting from 10:00 AM. (Optionally discard previous AOF file if we want)
- If redis restarts:
  - Load the snapshot first.
  - Then replay the AOF.

