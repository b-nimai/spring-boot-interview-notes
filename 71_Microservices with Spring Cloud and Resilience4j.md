# Microservices with Spring Cloud and Resilience4j

> **Part of:** Spring gap topics (63–71) · **Related:** [65_REST Clients](65_REST%20Clients%20-%20RestClient%2C%20WebClient%20and%20OpenFeign.md), [70_Kafka](70_Kafka%20Basics%20with%20Spring.md), [66_Observability](66_Actuator%2C%20Logging%20and%20Observability.md), [68_Docker](68_Docker%20for%20Spring%20Boot.md)

## Why this topic matters
At SDE-1/2 level you're expected to explain **when** microservices make sense, how services find and call each other, how failures are contained (circuit breakers), and how data consistency works across services (sagas). Deep Spring Cloud configuration matters less than the **concepts and trade-offs**.

---

## 1. Monolith vs microservices
| | Monolith (modular) | Microservices |
|---|---|---|
| Deploy | One unit | Independent per service |
| Scaling | Whole app | Per service |
| Data | One DB, ACID transactions | **Database per service**, eventual consistency |
| Complexity | Low ops, simple debugging | Network failures, distributed tracing, versioning, DevOps maturity needed |
| Team fit | Small team | Many teams owning bounded contexts |
**Good answer:** "Start with a well-modularized monolith. Split along **bounded contexts** (DDD) when team or scale pressure justifies the operational cost." Microservices trade code complexity for **distributed-systems complexity**.

## 2. The typical Spring Cloud landscape
```
Client ──► API Gateway (Spring Cloud Gateway: routing, auth/JWT check, rate limiting, CORS)
              │  lb://order-service  (service discovery: Eureka / Consul / Kubernetes DNS)
              ├──► order-service ──(REST/Feign + circuit breaker)──► inventory-service
              │         └──(Kafka event "OrderPlaced")──► payment-service, notification-service
              └──► user-service
   Config Server (centralized config) · Tracing (Zipkin/Tempo) · Logs (ELK/Loki) · Metrics (Prometheus)
```
| Concern | Spring Cloud piece | Kubernetes-native alternative |
|---|---|---|
| Service discovery | **Eureka** (`@EnableDiscoveryClient`), Consul | K8s Service DNS (`http://order-service`) |
| Client-side load balancing | Spring Cloud **LoadBalancer** (`lb://`, `@LoadBalanced`) | kube-proxy / service mesh |
| API gateway | **Spring Cloud Gateway** | Ingress / Envoy / Kong |
| Central config | **Config Server** (Git-backed) + refresh | ConfigMaps/Secrets |
| Resilience | **Resilience4j** (via Spring Cloud CircuitBreaker) | Service mesh (Istio) retries/timeouts |
| Tracing | Micrometer Tracing + OTel | Same |
On Kubernetes, many teams skip Eureka/Config Server and use the platform's features.

### Gateway route example
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates: [ Path=/api/orders/** ]
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter          # Redis-backed token bucket
              args: { redis-rate-limiter.replenishRate: 10, redis-rate-limiter.burstCapacity: 20 }
```

## 3. Resilience4j
```xml
<dependency><groupId>io.github.resilience4j</groupId><artifactId>resilience4j-spring-boot3</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-aop</artifactId></dependency>
```
(Boot 4 renames this starter to `spring-boot-starter-aspectj`.)
```java
@Service
public class InventoryGateway {
    @CircuitBreaker(name = "inventory", fallbackMethod = "stockFallback")
    @Retry(name = "inventory")
    @TimeLimiter(name = "inventory")                          // for CompletableFuture-returning methods
    @Bulkhead(name = "inventory")
    public CompletableFuture<StockDto> stock(String sku) {
        return CompletableFuture.supplyAsync(() -> inventoryClient.getStock(sku));
    }
    private CompletableFuture<StockDto> stockFallback(String sku, Throwable t) {   // same params + Throwable
        return CompletableFuture.completedFuture(StockDto.unknown(sku));            // cached / default / degrade
    }
}
```
```yaml
resilience4j:
  circuitbreaker.instances.inventory:
    sliding-window-size: 20
    failure-rate-threshold: 50              # % failures to OPEN
    slow-call-duration-threshold: 2s
    slow-call-rate-threshold: 50
    wait-duration-in-open-state: 10s
    permitted-number-of-calls-in-half-open-state: 3
  retry.instances.inventory:
    max-attempts: 3
    wait-duration: 200ms
    enable-exponential-backoff: true
    retry-exceptions: [ java.io.IOException, java.util.concurrent.TimeoutException ]
  timelimiter.instances.inventory.timeout-duration: 2s
  bulkhead.instances.inventory.max-concurrent-calls: 20
```
### Circuit breaker states
```
CLOSED ──(failure/slow rate ≥ threshold)──► OPEN ──(wait duration)──► HALF_OPEN
   ▲                                          │ calls fail fast            │ trial calls
   └──────────────(trials succeed)────────────┴──────(trials fail)──► OPEN
```
- **CLOSED**: calls pass, failures are counted in a sliding window.
- **OPEN**: calls are rejected immediately (`CallNotPermittedException` → fallback). This protects the struggling service and your own threads.
- **HALF_OPEN**: a few trial calls decide whether to close or reopen.
| Pattern | Protects against |
|---|---|
| Timeout / TimeLimiter | Hanging calls holding threads |
| Retry (backoff + jitter, idempotent only) | Transient blips |
| Circuit breaker | Repeated calls to a dead/slow dependency → **cascading failure** |
| Bulkhead | One dependency using all threads |
| Rate limiter | Overloading yourself or a downstream |
| Fallback | Gives the user a degraded but working response |
⚠️ Annotations are **proxy-based AOP**: self-invocation (`this.stock()`) bypasses them, the same as `@Transactional` ([42](42_Transactions%20and%20ACID.md)). Aspect order (default): Retry wraps CircuitBreaker wraps RateLimiter wraps TimeLimiter wraps Bulkhead.
⚠️ **Retry storms**: retries at every layer multiply (3 × 3 × 3 = 27 calls). Retry at one layer, and with budgets.

## 4. Communication styles
| Synchronous (REST/gRPC) | Asynchronous (Kafka/RabbitMQ events) |
|---|---|
| Simple request/response, immediate answer | Loose coupling, buffering, absorbs spikes |
| Temporal coupling: callee down means caller fails | Eventual consistency; harder to debug; needs idempotency |
| Use for queries needing an immediate answer | Use for "something happened" notifications and workflows |
Avoid long **synchronous chains** (A→B→C→D): latency adds up and availability multiplies (0.99⁴ ≈ 0.96).

## 5. Data consistency across services: the saga pattern
There's no distributed ACID transaction (2PC is avoided: blocking, poor availability). A **saga** = a sequence of local transactions, each publishing an event. A failure triggers **compensating transactions**.
```
Order: CREATED(pending) ─► Payment: CHARGED ─► Inventory: RESERVE fails
                                   ◄── compensate: REFUND ◄──┘
Order: CANCELLED
```
| Choreography | Orchestration |
|---|---|
| Services react to each other's events | A central orchestrator (state machine, e.g. Temporal / Camunda / custom) tells each step what to do |
| No central point; hard to follow with many steps | Explicit flow, easier to monitor; orchestrator is extra infrastructure |
Building blocks: **outbox** for reliable events ([70](70_Kafka%20Basics%20with%20Spring.md)), **idempotent consumers** ([69](69_Optimistic%20and%20Pessimistic%20Locking%20and%20Idempotency.md)), semantic locks (a "PENDING" status), and compensations designed as business operations (refund, not "delete row").

## 6. Other patterns worth naming
- **Database per service**: no shared tables. Others go through the owning service's API or events.
- **API composition / BFF**: the gateway or a backend-for-frontend aggregates calls for a UI.
- **CQRS**: separate write model and read model (often a projection built from events).
- **Strangler fig**: migrate a monolith piece by piece behind the gateway.
- **Service-to-service auth**: propagate the user's JWT or use OAuth2 client-credentials tokens ([48](48_OAuth2%20and%20OpenID%20Connect%20-%20Google%20Login.md)); mTLS in a mesh.
- **API versioning and backward compatibility**: additive changes, tolerant readers, consumer-driven contract tests (Spring Cloud Contract / Pact).
- **Health and observability**: readiness probes, tracing with traceId across hops ([66](66_Actuator%2C%20Logging%20and%20Observability.md)).

---

## Interview Q&A
**Q1. When would you NOT use microservices?**
Small team, unclear domain boundaries, early-stage product, or no DevOps/observability maturity. A modular monolith is cheaper and faster.

**Q2. How do services discover each other?**
A registry (Eureka/Consul) with client-side load balancing (`lb://name`), or Kubernetes Service DNS.

**Q3. Explain the circuit breaker pattern.**
Track failures and slow calls in a sliding window. Over the threshold, OPEN and fail fast with a fallback. After a wait, HALF_OPEN trial calls decide whether to close or reopen. It prevents cascading failures and thread exhaustion.

**Q4. Retry vs circuit breaker?**
Retry handles short transient faults. A circuit breaker stops calling a dependency that's persistently failing. Use them together with care (retries count toward the breaker; avoid retry storms).

**Q5. What is an API gateway for?**
A single entry point: routing, authentication, rate limiting, CORS, request aggregation, TLS termination, and observability.

**Q6. How do you maintain consistency across services without distributed transactions?**
Sagas (choreography or orchestration) with compensating actions, the outbox pattern for reliable events, and idempotent consumers. You accept eventual consistency.

**Q7. Sync vs async communication?**
Sync for immediate answers, at the cost of temporal coupling. Async events for decoupling and resilience, with eventual consistency and the need for idempotency.

**Q8. A downstream service is slow and your service starts failing. Why, and how do you fix it?**
Threads block waiting on it, so the pool is exhausted and you fail too (cascading). Fix with timeouts, a circuit breaker + fallback, a bulkhead, and async handling where possible.

**Q9. Why database per service?**
Independent deployability and schema evolution, plus clear ownership. A shared DB couples services at the data layer.

## Quick revision
- Microservices = independent deploy/scale per bounded context, at the price of distributed-systems complexity. Prefer a modular monolith first.
- Gateway + discovery + LoadBalancer + config + tracing (or the Kubernetes-native equivalents).
- Resilience4j: timeout, retry (idempotent, backoff), circuit breaker (CLOSED/OPEN/HALF_OPEN), bulkhead, fallback; proxy-based (self-invocation trap).
- Avoid long sync chains; prefer events for workflows.
- Sagas + outbox + idempotent consumers for cross-service consistency.
