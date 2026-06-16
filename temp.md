Sure. A **Token Bucket + Redis + Lua** solution is often preferred over a sliding-window ZSET approach because it uses much less memory and naturally supports bursts.

---

# Problem Statement

Allow:

* Capacity = 100 tokens
* Refill rate = 10 tokens/sec

This means:

* Bucket can hold at most 100 requests.
* Every second, 10 new tokens are added.
* A request consumes 1 token.

---

# Data Stored in Redis

For each user:

```text
rate_limit:user:123
```

Store:

```json
{
  "tokens": 75,
  "last_refill_time": 1718532000
}
```

You can store this as a Redis hash:

```redis
HSET rate_limit:user:123 tokens 75 last_refill 1718532000
```

---

# Request Flow

Assume:

```text
capacity = 100
refill_rate = 10/sec
```

Current state:

```text
tokens = 20
last_refill = 1000
```

Current request arrives at:

```text
now = 1003
```

---

## Step 1: Calculate elapsed time

```text
elapsed = now - last_refill

elapsed = 3 sec
```

---

## Step 2: Refill bucket

```text
new_tokens = elapsed * refill_rate

new_tokens = 3 * 10 = 30
```

Bucket becomes:

```text
20 + 30 = 50
```

Cap at max capacity:

```text
min(50, 100)

= 50
```

---

## Step 3: Check availability

Need:

```text
1 token
```

Available:

```text
50
```

Allowed.

---

## Step 4: Consume token

```text
50 - 1 = 49
```

Update Redis:

```text
tokens = 49
last_refill = now
```

---

# Why Lua?

Without Lua:

```text
HGET tokens
HGET last_refill

calculate refill

HSET tokens
HSET last_refill
```

Two concurrent requests could both read:

```text
tokens = 1
```

and both succeed.

Result:

```text
tokens = -1
```

Race condition.

---

# With Lua

Everything happens atomically.

```lua
local key = KEYS[1]

local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET',
                        key,
                        'tokens',
                        'last_refill')

local tokens = tonumber(data[1])
local last_refill = tonumber(data[2])

if tokens == nil then
    tokens = capacity
    last_refill = now
end

local elapsed = now - last_refill

local refill = elapsed * refill_rate

tokens = math.min(
    capacity,
    tokens + refill
)

if tokens < requested then
    return {
        0,
        tokens
    }
end

tokens = tokens - requested

redis.call(
    'HMSET',
    key,
    'tokens',
    tokens,
    'last_refill',
    now
)

redis.call(
    'EXPIRE',
    key,
    math.ceil(capacity/refill_rate)*2
)

return {
    1,
    tokens
}
```

---

# Execution

```redis
EVALSHA <sha> 1 rate_limit:user:123 100 10 1718532003 1
```

Meaning:

```text
100          bucket capacity
10           refill/sec
1718532003   current timestamp
1            tokens required
```

---

# Example Timeline

### Initial

```text
tokens = 100
```

---

### User sends 100 requests instantly

```text
100 -> 99
99 -> 98
...
1 -> 0
```

Bucket empty.

---

### Request #101

Lua sees:

```text
tokens = 0
```

Returns:

```text
blocked
```

---

### Wait 5 seconds

Refill:

```text
5 * 10 = 50
```

Bucket:

```text
0 -> 50
```

---

### User sends 30 requests

```text
50 -> 20
```

All succeed.

---

# Typical API Gateway Architecture

```text
Client
   |
   v
Load Balancer
   |
   v
API Service
   |
   v
Redis EVALSHA(Token Bucket Lua)
   |
   +--> Allowed -> Continue
   |
   +--> Rejected -> HTTP 429
```

Every API node can safely call the same Lua script because Redis guarantees:

```text
Script A
runs completely

then

Script B
runs completely
```

No interleaving occurs.

---

# Production Improvements

Most production systems also return:

```json
{
  "allowed": true,
  "remaining_tokens": 49,
  "retry_after": 0
}
```

or

```json
{
  "allowed": false,
  "remaining_tokens": 0,
  "retry_after": 3
}
```

The Lua script can calculate `retry_after` as:

```text
(required_tokens - available_tokens)
------------------------------------
          refill_rate
```

This lets your API return proper rate-limit headers such as:

```http
X-RateLimit-Remaining: 49
Retry-After: 3
```

This Redis + Lua token bucket pattern is essentially what many API gateways and microservice platforms use because it provides:

* Atomicity (no race conditions)
* Constant memory per user
* Burst handling
* Very high throughput
* Simple horizontal scaling across application servers.












That's an important architectural question.

## Where does the Lua script live?

There are two common approaches.

### Option 1: Script stored in application code

Your application contains the Lua script as a string:

```java id="f6r91x"
String luaScript = """
local key = KEYS[1]
...
return result
""";
```

On startup:

```redis id="9n9c9j"
SCRIPT LOAD <lua_script>
```

Redis returns:

```text id="wpx6z2"
8f3a4e9a8c7d...
```

(SHA1 hash of the script)

Your application caches this SHA.

Then for every request:

```redis id="kj9g5i"
EVALSHA 8f3a4e9a8c7d... 1 rate_limit:user:123 ...
```

This is the most common production approach.

---

### Option 2: Send script every time

```redis id="c7c1ca"
EVAL "
local key = KEYS[1]
...
" 1 rate_limit:user:123
```

Redis executes it immediately.

This works but is inefficient because Redis must parse the script every time.

---

## Where does it execute?

The Lua script executes **inside the Redis server process**.

Architecture:

```text id="vtqspw"
Application
     |
     | EVALSHA
     v
+----------------+
| Redis Process  |
|                |
| Lua Engine     |
|                |
| Executes:      |
| ZADD           |
| HGET           |
| HMSET          |
+----------------+
```

Your application does **not** execute the Lua code.

Redis executes it internally.

---

## What actually happens?

Suppose your API receives a request.

Application:

```java id="syl5u2"
redis.evalsha(
   tokenBucketSha,
   keys,
   args
);
```

Redis receives:

```text id="2f9d9z"
EVALSHA abc123...
```

Internally Redis does:

```text id="jv2hza"
1. Find Lua script by SHA
2. Start Lua interpreter
3. Execute script
4. Run redis.call(...)
5. Return result
```

All within the Redis server process.

---

## Why is it atomic?

Redis processes commands sequentially.

Imagine:

```text id="c7s3hm"
Client A -> EVALSHA
Client B -> EVALSHA
```

Redis event loop:

```text id="uqrl7t"
Execute A completely

then

Execute B completely
```

The Lua script is treated as one Redis command.

So:

```text id="m72n11"
HGET
calculate refill
HMSET
```

cannot be interrupted midway.

---

## What does `redis.call()` do?

Inside Lua:

```lua id="nvfxdf"
local tokens =
    redis.call(
        'HGET',
        key,
        'tokens'
    )
```

`redis.call()` is basically invoking Redis commands from within Redis itself.

Think of it as:

```text id="hwn4x7"
Lua Engine
    |
    +--> Redis Command Executor
            |
            +--> HGET
            +--> HMSET
            +--> ZADD
```

No network round-trip occurs.

---

## What happens after Redis restart?

Redis does **not permanently store** scripts loaded via `SCRIPT LOAD`.

Example:

```redis id="smrq70"
SCRIPT LOAD <script>
```

returns:

```text id="0kyswe"
abc123
```

After Redis restarts:

```redis id="8esajm"
EVALSHA abc123
```

fails with:

```text id="tkrm8y"
NOSCRIPT No matching script
```

Therefore applications typically:

```text id="jlwmmb"
Startup
  -> SCRIPT LOAD

Request
  -> EVALSHA

If NOSCRIPT
  -> SCRIPT LOAD again
  -> EVALSHA
```

Most Redis client libraries handle this automatically.

---

## Redis Functions (newer approach)

Since Redis 7, you can deploy Lua as a **Redis Function**.

Instead of:

```redis id="2twxbu"
SCRIPT LOAD
EVALSHA
```

you do:

```redis id="lm2s4y"
FUNCTION LOAD
```

The function becomes:

```text id="2v4bvv"
Persistent
Named
Versioned
```

Example:

```lua id="4nk2kg"
#!lua name=ratelimit

redis.register_function(
  'token_bucket',
  function(keys, args)
     ...
  end
)
```

Then call:

```redis id="owm7kf"
FCALL token_bucket 1 key arg1 arg2
```

For new Redis 7+ systems, Functions are often preferred because they survive restarts and are easier to manage.

---

### Mental model

Think of Lua in Redis as:

```text id="y4u90n"
Application
    |
    | EVALSHA
    v
Redis
 ├── Data
 ├── Command Engine
 └── Embedded Lua Interpreter
         |
         └── redis.call(...)
```

The Lua code runs **inside Redis**, next to the data, as a single atomic command. That's why it's so effective for rate limiting, locking, counters, inventory reservations, and other read-modify-write operations.















Yes, that's essentially correct. A few nuances are worth knowing.

### What happens

```text
Request A
    |
    v
EVALSHA token_bucket
    |
    v
Redis starts Lua script
    |
    +--> HGET
    +--> calculate tokens
    +--> HMSET
    |
    v
Script finishes

Request B
    |
    v
Redis starts Lua script
```

While Request A's Lua script is executing:

* Redis will not execute another Lua script.
* Redis will not execute normal commands from other clients.
* The script behaves like one giant atomic Redis command.

So from the perspective of Redis data:

```text
HGET
calculate
HMSET
```

is indivisible.

---

### What is actually blocked?

It's not just "other Lua scripts".

It's:

```text
Client A -> Lua script

Client B -> GET
Client C -> SET
Client D -> Lua script
Client E -> HINCRBY
```

All of those wait until Client A's script completes.

Redis processes them afterward.

---

### Example

Suppose the bucket has:

```text
tokens = 1
```

Two API servers receive requests simultaneously.

#### Request A

Lua starts:

```text
read tokens = 1
consume token
write tokens = 0
```

#### Request B

Cannot start yet.

It waits.

After A finishes:

```text
tokens = 0
```

Now B executes:

```text
read tokens = 0
reject request
```

Exactly one request succeeds.

---

### Why Redis warns against long-running scripts

Since Redis is blocked during script execution, a bad Lua script can freeze Redis.

Bad example:

```lua
for i = 1, 1000000000 do
end
```

While that runs:

```text
GET user:1
SET user:2
INCR counter
```

all wait.

That's why Redis Lua scripts should usually:

* Execute in a few milliseconds.
* Avoid large loops.
* Avoid scanning huge datasets.

Rate-limiter scripts are ideal because they typically do only a handful of operations.

---

### One subtle correction

When you say:

> "no other work can interrupt this Lua script execution"

That's true from Redis's command-processing perspective.

However, Redis itself may still be doing background activities such as:

* Persistence (RDB snapshots)
* AOF writes
* Replication networking

But no other client command can interleave with the script's Redis operations.

---

### Mental model

Treat this:

```lua
local count = redis.call(...)
if count < limit then
    redis.call(...)
end
```

as if Redis had a built-in command:

```redis
TOKEN_BUCKET_CHECK_AND_CONSUME
```

Even though it contains multiple Redis commands internally, Redis executes it as one atomic unit.

So your understanding is accurate:

1. API Gateway receives request.
2. Gateway calls `EVALSHA`.
3. Lua executes inside Redis.
4. No other client commands can interleave during execution.
5. Script updates Redis state atomically.
6. Script returns allow/reject.
7. Redis moves on to the next queued command or script.















That's a very good intuition, with one small caveat.

### Similarity to JavaScript's main thread

The analogy is:

```text
Browser JavaScript Main Thread
            ≈
Redis Command Execution Thread
```

In both cases:

* One unit of work runs at a time.
* Other work waits in a queue.
* Long-running work blocks everything behind it.

Browser:

```text
onclick handler
    |
    +--> expensive loop
```

While that runs:

* UI freezes
* Clicks wait
* Rendering waits

Redis:

```text
Lua Script
    |
    +--> expensive loop
```

While that runs:

* GET waits
* SET waits
* Other Lua scripts wait
* Rate limiter calls wait

So the "keep it short" principle applies in both worlds.

---

### About the lock analogy

Conceptually, yes:

```text
Lua Script
     =
Implicit lock on Redis data
```

But Redis isn't actually acquiring a lock object like:

```java
synchronized(lock) {
    ...
}
```

or

```java
mutex.lock()
...
mutex.unlock()
```

Instead, atomicity comes from Redis's execution model itself.

Think of it this way:

```text
Traditional DB

Request A
    acquires lock
Request B
    waits for lock
Request A
    releases lock
```

vs

```text
Redis

Request A
    enters execution queue
    runs completely

Request B
    waits in queue
```

No explicit lock is needed because only one command/script executes at a time.

---

### Another useful mental model

Without Lua:

```text
Request A
    HGET

Request B
    HGET

Request A
    HSET

Request B
    HSET
```

Commands from different requests can interleave.

With Lua:

```text
Request A
    [HGET + logic + HSET]

Request B
    [HGET + logic + HSET]
```

Each bracketed block is executed as one atomic unit.

So Lua effectively gives you a **critical section** around a group of Redis operations.

---

### Why rate limiting is a perfect use case

A token bucket script usually does:

```text
HMGET
calculate refill
HMSET
return result
```

That's only a few microseconds to a few milliseconds.

So:

```text
Request A
    runs Lua (1 ms)

Request B
    waits 1 ms

Request C
    waits 2 ms
```

Very little contention.

If instead the script did:

```text
scan 1 million keys
loop over them
perform calculations
```

then every client of Redis would feel the slowdown.

---

### One final distinction

A traditional lock protects **some resource**:

```java
lock(account123)
```

Other accounts can still be updated.

Redis Lua is more like:

```text
pause all Redis command execution
run script
resume Redis command execution
```

So it's actually stronger than a per-resource lock—it temporarily monopolizes Redis's command execution thread.

That's why Redis scripts should be:

* Short
* Deterministic
* O(1) or close to it
* Free of expensive loops

For a token bucket rate limiter, Lua is almost an ideal fit because the work is tiny and the benefit of atomicity is huge.














Exactly. As an analogy, your understanding is spot on.

You can think of it as:

```text
Redis Event Loop

Queue:
---------------------------------
Request A -> Lua Script
Request B -> GET
Request C -> Lua Script
Request D -> HSET
---------------------------------

Execution:

Run A completely
Run B
Run C completely
Run D
```

While Request A's Lua script is executing:

```text
Lua Script A
    |
    +--> HGET
    +--> calculations
    +--> HMSET
```

Requests B, C, and D are effectively waiting behind it.

That's why people often say:

> "Lua scripts in Redis behave like an implicit global lock."

Not because Redis creates a mutex, but because the single-threaded execution model guarantees exclusivity.

A useful comparison:

| System              | Why others wait                 |
| ------------------- | ------------------------------- |
| Java `synchronized` | Explicit lock                   |
| Database row lock   | Explicit lock                   |
| Redis Lua script    | Single-threaded execution queue |
| Browser JS          | Single-threaded event loop      |

The effect is similar:

```text
Only one unit of work is modifying shared state at a time.
```

The implementation is different.

This is also why Redis scripts are usually designed to be:

```text
Read a few values
Perform small calculations
Write a few values
Return
```

rather than:

```text
Loop over 1M keys
Perform heavy computation
Make external calls
```

because during that time you're effectively holding up Redis's command-processing pipeline.

One thing senior engineers often realize after working with Redis for a while:

> A Lua script doesn't just block competing rate-limit requests—it blocks *all* commands hitting that Redis instance.

So if your Redis instance is shared for:

* rate limiting
* session storage
* caching
* counters
* feature flags

then a slow Lua script can increase latency for every one of those workloads.

That's why token bucket scripts are such a good fit: they typically execute in microseconds or a few milliseconds, giving you atomicity with very little impact on throughput.
