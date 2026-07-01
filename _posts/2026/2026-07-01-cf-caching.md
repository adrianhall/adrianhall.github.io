---
title:  "Caching options in Cloudflare Workers"
date:   2026-07-01
categories:
  - "Cloud Development"
tags:
  - cloudflare
---

When you get into systems design, eventually you are going to have to deal with caching.  I am currently writing [my serverless MUD]({% post_url 2026/2026-05-01-cf-workers-3 %}) and part of that system requires an automatic registration of users hitting the API.  This means that every single API call reads from [Cloudflare D1] in the hot path.  Reading from D1 is incredibly fast, but it's not latency-free, particularly during periods of high load.

This got me thinking about caching the read "somewhere cheaper".  By cheaper, I mean lower latency and cost.  So, what are your options, and when would you choose one over the other?  

![Cloudflare Caching Options]({{site.baseurl}}/assets/images/2026/Jul01-banner.png)

In my Worker, I have a simple cache definition:

```ts
export interface CacheManager<T> {
  get(key: string): Promise<T | undefined>;
  set(key: string, value: T): Promise<void>;
  remove(key: string): Promise<void>;
  clear(): Promise<void>;
}
```

I can then have multiple implementations of the cache manager and inject them into my auto-registration middleware to get different behavior.

## An in-memory cache

An in-memory cache is perhaps the easiest to comprehend.  Use a `Map<K, V>` to store data and automatically evict stored values to prevent the cache from getting too big.  Here is the code:

```ts
export class InMemoryCacheManager<T> implements CacheManager<T> {
  private readonly store = new Map<string, T>();
  private readonly maxEntries: number;

  constructor(maxEntries: number = DEFAULT_MAX_ENTRIES) {
    this.maxEntries = maxEntries;
  }

  get(key: string): Promise<T | undefined> {
    return Promise.resolve(this.store.get(key));
  }

  set(key: string, value: T): Promise<void> {
    if (!this.store.has(key) && this.store.size >= this.maxEntries) {
      const oldest = this.store.keys().next().value;
      if (oldest !== undefined) {
        this.store.delete(oldest);
      }
    }
    this.store.set(key, value);
    return Promise.resolve();
  }

  // ... other methods not shown
}
```

This is your fastest and least reliable option and best used as an opportunistic L1 cache.  There is no explicit per-read/write billing, but there are significant drawbacks:

- Not durable.
- Not shared across isolates or colos.
- Not guaranteed to exist on the next request.
- Limited by the Workers memory limit of 128MB (total).

[Cloudflare Workers] are stateless; each request *may* run on a different instance, in a different location, with no shared memory between requests.  At the same time, global scope can persist when an isolate is reused.  There is no fixed TTL for Worker isolates - each request may hit the same isolate or it may hit a completely different one.  Assume a Worker isolate can vanish at any time.

As a result, the in-memory cache should be considered a best-effort hot-isolate cache, not a real cross-request store.  Use it for ultra-short-lived hot-key caching and per-isolate memoization.  Don't use it to reduce the reads/writes on your D1 database or for revocation-sensitive auth state.

## Workers Cache API

Each colo within the Cloudflare network has a CDN cache. The [Workers Cache API] can be used to read and write data to that cache, which provides a mechanism for storing small bits of data.  It's useful when your users are likely to be hitting the same colo all the time (which, in normal operation, they would be).  Here is the implementation:

```ts
export class ColoCacheManager<T> implements CacheManager<T> {
  private url(key: string) {
    return `https://cache.local/${encodeURIComponent(key)}`;
  }

  async get(key: string): Promise<T | undefined> {
    const res = await caches.default.match(this.url(key));
    return res ? (await res.json()) as T : undefined;
  }

  async set(key: string, value: T): Promise<void> {
    const res = new Response(JSON.stringify(value), {
      headers: { 
        "Content-Type": "application/json", 
        "Cache-Control": "max-age=60" 
      },
    });
    await caches.default.put(this.url(key), res);
  }

  // ... other methods not shown
}
```

The Cache API mirrors the browser [CacheStorage API], so it should be a familiar interface.  All operations use a `Request` -> `Response` store, so you have to serialize data accordingly.

The Cache API has major advantages over the in-memory cache - most notably, it's really good when the user tends to land in the same colo repeatedly, which is the common case.  Also, the TTL is under your control.  It's still included in your Workers / CDN price.

That being said, it does not replicate outside the originating data center. That makes revoking or deleting cache keys problematic, especially when you are storing authorization data. Workers Cache API is a good fit when user data changes infrequently and slight staleness is acceptable. Workers Cache API does not work when the application is fronted by Cloudflare Access, and it requires a custom domain.

This is the solution to choose when your main goal is to reduce repeated reads from the same region.  Use it for user profile blobs, hydrated session responses and per-user response caching.  Don't use it when the data must be globally fresh (e.g. authorization revocation data) or where users roam between colos frequently.

## Workers KV

[Workers KV] is the key-value store of choice when a global cache is required and eventual consistency is acceptable.  KV data is centrally stored and then cached through Cloudflare's network after access.  As a result of this design, it has good 'read-hit' latency, but relatively poor 'read-miss' latency.  Use this for read-heavy workloads.

```ts
export class KVCacheManager<T> implements CacheManager<T> {
  private kv: KVNamespace;
  
  constructor(kv: KVNamespace) {
    this.kv = kv;
  }

  async get(key: string): Promise<T | undefined> {
    const value = await this.kv.get<T>(ey, { type: "json", cacheTtl: 60 });
    return value ?? undefined;
  }

  async set(key: string, value: T): Promise<void> {
    await this.kv.put(key, JSON.stringify(value), {
      expirationTtl: 60,
      metadata: { cachedAt: Date.now() }
    });
  }

  // ... other methods not shown  
}
```

You will need access to the KV namespace binding, which is passed in via context to a Hono app.  This makes the construction of the pipeline doing auto-registration a little more complex since you no longer have a globally defined cache.

There are two TTL values you control here:

- `expirationTtl` is set on `put()` and controls when the key is deleted (min: 60s).
- `cacheTtl` is set on `get()` and determines how long a read stays cached at the edge (min: 30s).

Workers KV is more globally available than Workers Cache and built for read-heavy workloads.  It's also got better limits than the in-memory cache - up to 25MB values.  It's good for write-once / write-rarely workloads.  However, it is *eventually* consistent, so you have to tune the TTL according to your needs by setting the combination of `expirationTtl` (on writes) and `cacheTtl` (on reads). It's also more expensive than a D1 read (although it has lower latency).

## No cache option

Of course, you don't have to use a cache.  D1 has many advantages even in situations where you think a cache might be useful. Writes are durable and atomic ([ACID]), and the [D1 Sessions API] can introduce (eventually consistent) read replicas so reads reflect committed writes (with a slight lag).

This does involve making certain decisions about your code, however.  For instance, reads are much cheaper than writes:

| Unit billed | Free plan/day   | Included amount/month | Cost per million |
|-------------|-----------------|-----------------------|------------------|
| Rows read   | 5 million       | 25 billion            | $0.001           |
| Rows written| 100 thousand    | 50 million            | $1.00            |

If I'm not paying for the writes, I'd be tempted to use an INSERT operation that protects against race conditions on every API call:

```sql
INSERT INTO users (email, sub) VALUES (?1, ?2)
  ON CONFLICT(email) DO UPDATE SET email = excluded.email
  RETURNING *
```

This is an atomic operation.  I don't have to worry about ordering.  However, it incurs a write cost per API call.  Instead, do a SELECT operation before the INSERT.  The SELECT call hits your D1 read usage, but it avoids writing in the hot path.  Your write usage remains low.  For my use case, I only incur a write when a user first uses the site (and potentially during a race condition where two API calls for the same new user happen at the same time).

## Final thoughts

What can you expect from each layer?

| Layer | Latency | Scope | Freshness | Durability | Max value size | Good for |
|-------|---------|-------|-----------|------------|----------------|----------|
| In-memory | Best | One isolate | Unpredictable | None | Very small | Burst reuse / memoization |
| Workers Cache | Very good on hit | One colo | TTL-based, colo-local | Temporary | 512MB | Edge response caching |
| Workers KV | Good, especially hot | Global read distribution | Eventually consistent | Durable | 25MB | Read-heavy workloads |
| D1 | Higher than cache | Primary + replicas | Stronger than KV for truth | Durable | N/A | Authoritative user state |

For my situation, I'm using D1 with a small in-memory cache.  The reasons?

- My hot path is on every single API call.
- Users tend to issue API calls in rapid succession, so they are more likely to hit the same isolate given the low volume of users I expect.
- I'm using D1 as the source of truth.
- I'm not using the cache for authorization decisions.
- I can't use Workers Cache API because my app is behind Cloudflare Access.

Every situation is unique.  You need to look at your constraints, consider the cost and latency strategy, then decide which architecture to pick for effective use of the resources you have available.

<!-- Links -->
[Cloudflare D1]: https://developers.cloudflare.com/d1/
[D1 Sessions API]: https://developers.cloudflare.com/d1/best-practices/read-replication/
[Workers KV]: https://developers.cloudflare.com/kv/
[Cloudflare Workers]: https://developers.cloudflare.com/workers/
[Workers Cache API]: https://developers.cloudflare.com/workers/runtime-apis/cache/
[CacheStorage API]: https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage
[ACID]: https://en.wikipedia.org/wiki/ACID
