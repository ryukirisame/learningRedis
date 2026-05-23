
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
