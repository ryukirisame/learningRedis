
# Redis for Session store.

## What is session?
- Sessions allows a server to "remember" a user across requests.
  - If a client requests for a piece of data, how does the server know if this client is authenticated & authorized to access this data? Through sessions!
- How it works:
  - User logs in → server creates a session with a unique ID
  - Session ID is sent to the client (usually as a cookie)
  - On every subsequent request, the client sends the session ID
  - Server looks up the session data using that ID
  - If a session with that ID exists, the server will proceed further, otherwise that means the user is not **authenticated** and will ask user to login.
- Typical session data might include: user ID, roles/permissions, cart contents, preferences, login timestamps, etc.

## The Problem: Why can't we store sessions inside application server itself?

If you store sessions inside application server then it will break when you:
- Scale horizontally — user hits Server A on request 1, Server B on request 2 → session lost
- Restart the server — all sessions wiped
- Have many users — memory fills up fast

## How redis solves the problem
- Since Redis is used as a single layer, so it becomes a centralized place where all the sessions are stored. All the application servers share this store. So no sync issues.
- Redis has a feature of Time-to-Live (TTL), which we can set for all the session key-value pairs. So that sessions can expire automatically after a specified period of time. Redis deletes the key-value entry automatically.
  - In production, we use **sliding session window** rather than a hard expiration. Everytime a user makes a request and our backend successfully reads their session, we extend their session expiry time. 
- Since session data should be quickly accessed, so Redis is a good fit whose latency is very very low compared to traditional databases.
- Redis also has a persistence layer. So if redis restarts, the sessions can be restored.
- Since redis engine is single-threaded, so no read/write synchronization issues.


## Full Session Flow
```
Browser
   ↓
Login Request
   ↓
Backend Server
   ↓
Create Session
   ↓
Store Session in Redis
   ↓
Send Session ID Cookie
   ↓
Browser stores cookie
```
Then for future requests:
```
Browser sends session_id cookie
   ↓
Backend reads cookie
   ↓
Backend fetches session from Redis
   ↓
User authenticated
```

- To forcefully logout a user, just delete the key.




