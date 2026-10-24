# Caching with Spring Cache and Redis

> **Part of:** Spring gap topics (63–71) · 🟦 NestJS: `@nestjs/cache-manager` + `CacheInterceptor` ≈ Spring `@Cacheable`

## Why this topic matters
"How would you reduce DB load / latency?" leads to caching, then cache invalidation, TTLs, stampedes, and consistency. Spring's cache abstraction is **AOP-based**, so the proxy pitfalls from note 34 apply.

---

## 1. Where caching fits
```
Client → [CDN/HTTP cache] → App → [local cache (Caffeine)] → [distributed cache (Redis)] → DB
```
| | Local (in-memory: Caffeine, ConcurrentMap) | Distributed (Redis, Memcached) |
|---|---|---|
| Speed | Fastest (ns–µs, no network) | ~sub-ms network hop |
| Shared across instances | ❌ (each pod has its own copy) | ✅ |
| Consistency across pods | Hard (stale copies) | One source |
| Survives restart | ❌ | ✅ (if persisted) |
| Use | Hot, small, rarely-changing data (config, lookups) | Sessions, shared data, rate limits, horizontally scaled apps |

## 2. Spring Cache abstraction
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-cache</artifactId></dependency>
<!-- provider: Caffeine (local) or Redis -->
<dependency><groupId>com.github.ben-manes.caffeine</groupId><artifactId>caffeine</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-redis</artifactId></dependency>
```
```java
@SpringBootApplication
@EnableCaching                       // turns on the caching proxy (CacheInterceptor)
public class App { }
```
With no provider on the classpath, Boot uses a simple `ConcurrentHashMap` (no TTL, no eviction: dev only).

### Annotations
```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id", unless = "#result == null")   // read-through
    public ProductDto getProduct(Long id) {
        log.info("DB hit for {}", id);                    // only runs on a cache MISS
        return repo.findById(id).map(this::toDto).orElse(null);
    }

    @CachePut(value = "products", key = "#result.id")    // always runs, then updates the cache
    public ProductDto update(Long id, UpdateRequest r) { ... }

    @CacheEvict(value = "products", key = "#id")         // remove one entry
    public void delete(Long id) { repo.deleteById(id); }

    @CacheEvict(value = "products", allEntries = true)   // clear the whole cache
    public void reloadAll() { }

    @Caching(evict = { @CacheEvict(value = "products", key = "#id"),
                       @CacheEvict(value = "productLists", allEntries = true) })
    public void archive(Long id) { }
}
```
| Annotation | Behaviour |
|---|---|
| `@Cacheable` | Hit → return the cached value, **method skipped**. Miss → run the method, store the result |
| `@CachePut` | Always runs the method, then writes the result (keeps the cache fresh after updates) |
| `@CacheEvict` | Removes entries (`beforeInvocation = true` to evict even if the method throws) |
| `@Caching` | Combine several |
| `@CacheConfig(cacheNames = "products")` | Class-level defaults |
- **Key**: SpEL (`#id`, `#user.email`, `#root.methodName`). The default key is built from all parameters (`SimpleKey`). Custom: a `KeyGenerator` bean.
- **`condition`** (evaluated before the call, on args) vs **`unless`** (after, can use `#result`).
- `sync = true`: only one thread computes a missing key (helps with stampedes on a single instance).

### ⚠️ Proxy pitfalls (same as `@Transactional`)
- **Self-invocation** (`this.getProduct()`) bypasses the cache.
- `private`/`final` methods aren't intercepted.
- **Cache DTOs, not JPA entities** (lazy proxies, detached state, serialization problems).
- Cached objects are shared: **don't mutate them** (with a local cache, the caller gets the same instance). Prefer immutable records.

## 3. Caffeine (local) configuration
```properties
spring.cache.cache-names=products,users
spring.cache.caffeine.spec=maximumSize=10000,expireAfterWrite=10m,recordStats
```
Caffeine uses W-TinyLFU eviction (near-optimal hit ratio), size/time bounds, and async refresh.

## 4. Redis cache
```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.type=redis
spring.cache.redis.time-to-live=10m
spring.cache.redis.key-prefix=myapp:
```
```java
@Bean
RedisCacheConfiguration cacheConfig() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .disableCachingNullValues()
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()));   // JSON instead of JDK serialization
}
```
Default JDK serialization requires `Serializable` and is brittle across versions, so use JSON.
Direct Redis use (beyond caching): `StringRedisTemplate` / `RedisTemplate` for counters, rate limits (`INCR` + `EXPIRE`), distributed locks (`SET key val NX PX 30000`, or Redisson), and pub/sub.

## 5. Caching patterns
| Pattern | How | Notes |
|---|---|---|
| **Cache-aside (lazy loading)** | App reads the cache → on a miss, reads the DB and populates | What `@Cacheable` does; the most common |
| Read-through | The cache loads from the DB itself | Caffeine `LoadingCache` |
| Write-through | Write the cache + DB synchronously | Consistent, slower writes |
| Write-behind | Write the cache, flush to the DB async | Fast, risk of loss |
| **Invalidate on write** | Update DB → **evict** the key | Safer than updating the cache (avoids race-induced stale data) |

## 6. Classic problems
| Problem | What happens | Fix |
|---|---|---|
| **Stale data** | DB changed, cache didn't | TTL + evict on write; events (Kafka/CDC) for cross-service |
| **Cache stampede / thundering herd** | A hot key expires and 1000 requests hit the DB at once | `sync = true`, locking/single-flight, early refresh, jittered TTLs |
| **Cache penetration** | Requests for non-existent keys always miss | Cache null/empty with a short TTL, Bloom filter |
| **Cache avalanche** | Many keys expire together | Random TTL jitter, warm-up |
| **Hot key** | One key overloads a Redis shard | Local L1 cache in front, key replication |
| Memory blow-up | Unbounded local cache | Max size + eviction (LRU/LFU) |
Order on update: **update DB, then delete the cache** (not delete-then-update, which races with readers). For strict needs, delayed double delete or CDC.

**What to cache:** read-heavy, expensive, tolerant of slight staleness (product catalogue, user profile, config, computed aggregates). **Avoid:** fast-changing data that needs strong consistency (balances, inventory counts) unless carefully designed.

## 7. Monitoring
`/actuator/caches` lists the caches. Micrometer `cache.gets{result=hit|miss}`, `cache.evictions` (enable `recordStats` for Caffeine). Track the **hit ratio**.

---

## Interview Q&A
**Q1. `@Cacheable` vs `@CachePut` vs `@CacheEvict`?**
Skip the method on a hit / always run and update the cache / remove entries.

**Q2. Why doesn't my `@Cacheable` work?**
No `@EnableCaching`, self-invocation, a private/final method, a key that differs per call, or `unless` preventing storage.

**Q3. Local vs distributed cache?**
Local is fastest but per-instance and inconsistent across pods. Distributed (Redis) is shared and survives restarts, at the cost of a network hop. Often both (L1 + L2).

**Q4. How do you keep the cache consistent with the DB?**
TTL + evict-on-write (update DB, then delete the key), `@CachePut` for write paths, events/CDC across services, and accept bounded staleness.

**Q5. What is a cache stampede and how do you prevent it?**
Many concurrent misses on an expired hot key overwhelm the DB. Prevent with single-flight (`sync=true`/locks), early refresh, and jittered TTLs.

**Q6. Cache penetration vs avalanche?**
Repeated misses for non-existent keys (cache nulls, Bloom filter) vs mass simultaneous expiry (TTL jitter).

**Q7. Why cache DTOs rather than entities?**
Entities carry lazy proxies and persistence state, serialize poorly, and can leak managed state between transactions.

**Q8. How would you implement rate limiting with Redis?**
`INCR key` with `EXPIRE` per time window (fixed window), sorted sets (sliding window), or a token bucket via a Lua script / Bucket4j.

## Quick revision
- `@EnableCaching` + `@Cacheable`/`@CachePut`/`@CacheEvict`; SpEL keys; `unless`/`condition`/`sync`.
- AOP proxy: no self-invocation, cache immutable DTOs.
- Caffeine local (size + TTL) vs Redis distributed (TTL, JSON serializer).
- Cache-aside; update DB then evict.
- Stampede, penetration, avalanche, hot key, and their fixes; monitor the hit ratio.
