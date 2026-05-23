
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
