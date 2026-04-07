# Redis: Caching Strategies, Cache Invalidation, and Rate Limiting

Redis is not just a key-value store; its speed and data structures make it a crucial component for performance optimization and traffic management in backend systems. In this guide, we dive deep into how to effectively manage caches and handle rate limiting using Redis in a Node.js/Express environment.

---

## 1. Caching Strategies

A caching strategy dictates how you handle reading from and writing to your database and cache. Choosing the right pattern improves read/write performance and data consistency.

### 1.1 Cache-Aside (Lazy Loading)
This is the most common strategy. The application acts as a middleman between the cache and the database.
1. Application checks the cache.
2. If the data is found (**Cache Hit**), it returns the data immediately.
3. If not found (**Cache Miss**), it fetches the data from the Database, stores it in the Cache for future requests, and then returns it.

**Example (Node.js):**
```javascript
const redis = require('redis');
const client = redis.createClient();

async function getUserCacheAside(userId) {
  // 1. Check Cache
  const cachedUser = await client.get(`user:${userId}`);
  if (cachedUser) {
    console.log("Cache Hit");
    return JSON.parse(cachedUser); 
  }

  console.log("Cache Miss");
  // 2. Fetch from DB
  const user = await database.query('SELECT * FROM users WHERE id = ?', [userId]);

  if (user) {
    // 3. Store in Cache with TTL (Time To Live), e.g., 3600 seconds (1 hour)
    await client.setEx(`user:${userId}`, 3600, JSON.stringify(user));
  }

  return user;
}
```

### 1.2 Write-Through Cache
When writing data, the application writes it to the cache and the database synchronously.
- **Pros:** The cache is always up to date and consistent. 
- **Cons:** Write latency increases because the application has to write to two places before returning a success response.

**Example:**
```javascript
async function saveUserWriteThrough(user) {
  // 1. Write to DB
  await database.query('INSERT INTO users...', [user]);
  
  // 2. Write to Cache synchronously
  await client.setEx(`user:${user.id}`, 3600, JSON.stringify(user));

  return user; // Return success only after both operations complete
}
```

### 1.3 Write-Behind (Write-Back) Cache
The application writes data to the cache immediately and acknowledges the write to the client. The cache then asynchronously writes the data to the database in the background.
- **Pros:** Very fast write performance.
- **Cons:** High risk of data loss if the cache server crashes before data is synced to the underlying database.

---

## 2. Cache Invalidation

Cache invalidation is the process of removing or replacing cached data to ensure clients don't receive stale (outdated) information. Keeping cache perfectly in sync with the database is known as one of the hardest problems in computer science!

### 2.1 Time-To-Live (TTL)
The simplest and safest approach. Every cached item is given a lifespan. Once the time expires, Redis automatically deletes the key.

```javascript
// Sets a key with a 60-second expiration duration
await client.setEx('dashboard_stats', 60, JSON.stringify(stats));
```

### 2.2 Event-Driven / Explicit Invalidation
When underlying data changes in the database (via POST/PUT/DELETE requests), you explicitly delete or update the corresponding cache key.

```javascript
async function updateUser(userId, newData) {
  // 1. Update the database
  await database.query('UPDATE users SET ... WHERE id = ?', [newData, userId]);

  // 2. Explicitly invalidate (delete) the cache
  await client.del(`user:${userId}`);
  
  // Next time the user is requested, it will result in a Cache Miss, 
  // forcing the app to fetch the updated data and re-cache it.
}
```

### 2.3 Eviction Policies (Memory Management)
When Redis runs out of memory, it automatically invalidates older keys to make room for new ones based on a configured policy (managed in `redis.conf` via the `maxmemory-policy` directive).
*   **noeviction:** Returns an error on write operations when memory is full (default).
*   **allkeys-lru:** Evicts the **Least Recently Used** keys among all keys (highly recommended for typical caches).
*   **allkeys-lfu:** Evicts the **Least Frequently Used** keys.
*   **volatile-lru:** Evicts LRU keys, but only those set with an expiration (TTL).

---

## 3. Rate Limiting with Redis

Rate limiting controls the number of requests a client can make in a given timeframe. It prevents abuse (e.g., DDoS attacks, credential stuffing) and avoids overloading APIs. Redis' atomic operations (`INCR`, `EXPIRE`) make it perfectly suited for distributed rate limiting across multiple server instances.

### 3.1 Fixed Window Strategy
Counts requests in a predefined time block (e.g., maximum 10 requests per minute).
- **Pros:** Very easy to implement.
- **Cons:** Suffers from the "Boundary Spike" problem. A user can execute 10 requests at 1:00:59, and another 10 requests at 1:01:01. The system technically allowed 20 requests within 2 seconds.

**Example (Express.js Middleware):**

```javascript
const express = require('express');
const redis = require('redis');

const app = express();
const client = redis.createClient({ url: 'redis://localhost:6379' });
client.connect();

const RATE_LIMIT = 10; // Max requests allowed
const WINDOW_SIZE_IN_SECONDS = 60; // 1-minute window

const fixedWindowLimiter = async (req, res, next) => {
  const ip = req.ip; 
  const redisKey = `rate_limit:${ip}`;

  try {
    // Atomically increment the request count
    const requests = await client.incr(redisKey);

    // If it's the very first request, set the expiration window
    if (requests === 1) {
      await client.expire(redisKey, WINDOW_SIZE_IN_SECONDS);
    }

    if (requests > RATE_LIMIT) {
      return res.status(429).json({ 
        error: "Too many requests. Please try again later." 
      });
    }

    next(); // Proceed to the route handler
  } catch (error) {
    console.error("Rate Limiter Error:", error);
    next(error);
  }
};

app.get('/api/data', fixedWindowLimiter, (req, res) => {
  res.json({ message: "Success! Data fetched securely." });
});
```

### 3.2 Sliding Window Log
Records the exact timestamp of every incoming request. To check limits, you remove timestamps older than the window limits, and count the remaining timestamps.
- **Pros:** Highly accurate and smooth. Completely eliminates boundary spikes.
- **Cons:** Memory intensive because every request's timestamp must be saved (usually stored using a Redis `Sorted Set` where the score is the timestamp).

### 3.3 Token Bucket Algorithm
Each user is assigned a "bucket" with a maximum capacity of tokens. Tokens are added to the bucket at a constant rate. Every request "spends" a token.
- **Pros:** Beautifully handles burst traffic (up to the maximum bucket limit) while still ensuring a smooth average rate over time.

> **Note:** For advanced and robust implementations like Token Bucket or Sliding Window in production Node.js apps, writing manual logic can get tricky and prone to race conditions unless carefully scripted with Lua. Using external, battle-tested packages like `rate-limiter-flexible` alongside Redis is heavily recommended.
