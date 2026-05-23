
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
