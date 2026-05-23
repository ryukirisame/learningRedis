
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
