# Actuator, Logging and Observability

> **Part of:** Spring gap topics (63–71) · 🟦 NestJS: `@nestjs/terminus` (health) + Winston/Pino (logs) + OpenTelemetry
> **Related:** [61_JVM and GC](61_JVM%20Memory%2C%20Class%20Loading%20and%20Garbage%20Collection.md) (heap dumps, GC metrics)

## Why this topic matters
Product companies run services in Kubernetes with dashboards and alerts. Expect "how do you debug a slow API in production?", "how do health checks work?", and "how do you trace a request across services?". The answer is the **three pillars**: logs, metrics, traces.

---

## 1. Spring Boot Actuator
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
```
```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,loggers
management.endpoint.health.show-details=when-authorized
management.endpoint.health.probes.enabled=true      # /actuator/health/liveness and /readiness
management.server.port=9090                         # optional: separate management port
```
| Endpoint | Shows |
|---|---|
| `/actuator/health` | UP/DOWN + components (db, diskSpace, redis…) |
| `/actuator/health/liveness` / `readiness` | Kubernetes probes |
| `/actuator/info` | Build/git/app info |
| `/actuator/metrics`, `/actuator/metrics/{name}` | Micrometer metrics |
| `/actuator/prometheus` | Prometheus scrape format (needs `micrometer-registry-prometheus`) |
| `/actuator/loggers` | View/**change log levels at runtime** (POST) |
| `/actuator/env`, `/configprops`, `/beans`, `/mappings`, `/conditions` | Debugging config and auto-config |
| `/actuator/threaddump`, `/heapdump` | JVM diagnostics |
| `/actuator/caches`, `/scheduledtasks`, `/flyway` | Feature-specific |
By default only `health` is exposed over HTTP. **Secure the others**: `env`, `heapdump`, and `beans` leak secrets and internals. Expose them on an internal port, put them behind Spring Security, or keep them unexposed.

### Liveness vs readiness (Kubernetes)
| Probe | Question | Failing means |
|---|---|---|
| **Liveness** | "Is the process stuck/broken?" | Kubernetes **restarts** the pod |
| **Readiness** | "Can it serve traffic right now?" (warming up, dependency down) | Removed from the **load balancer**, not restarted |
Don't put the DB in liveness (a DB outage would restart every pod in a loop). Readiness is the right place, if anywhere.

### Custom health indicator
```java
@Component
public class PaymentApiHealth implements HealthIndicator {
    public Health health() {
        return client.ping() ? Health.up().withDetail("latencyMs", 12).build()
                             : Health.down().withDetail("error", "timeout").build();
    }
}
```

## 2. Logging
Spring Boot uses **SLF4J** (facade) + **Logback** (default implementation).
```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);   // or Lombok @Slf4j
log.info("Order {} placed by user {}", orderId, userId);        // ✅ parameterized (no string concat cost)
log.debug("Payload: {}", payload);                              // only built if DEBUG is enabled
log.error("Payment failed for order {}", orderId, ex);          // ✅ exception as the LAST arg → stack trace
// ❌ log.error("failed: " + ex.getMessage())  → loses the stack trace
```
Levels: `TRACE < DEBUG < INFO < WARN < ERROR`.
```properties
logging.level.root=INFO
logging.level.com.myapp=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.file.name=logs/app.log
logging.structured.format.console=ecs      # Boot 3.4+: JSON structured logs (ecs | logstash | gelf)
```
Advanced: `logback-spring.xml` (profile-specific appenders with `<springProfile>`, rolling files, async appenders).

### MDC: a correlation id in every log line
```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader("X-Request-ID")).orElse(UUID.randomUUID().toString());
        MDC.put("requestId", id);
        res.setHeader("X-Request-ID", id);
        try { chain.doFilter(req, res); }
        finally { MDC.remove("requestId"); }          // ThreadLocal → always clean up
    }
}
// pattern: %d %-5level [%X{requestId}] %logger{36} - %msg%n
```
With Micrometer Tracing, `traceId`/`spanId` are added to the MDC automatically.

### Logging best practices
- Use **structured (JSON) logs** in production, shipped to ELK/Loki/Datadog/CloudWatch.
- Log **events with context** (ids, outcome, duration), not noise.
- **Never log secrets or PII** (passwords, tokens, card numbers); mask them.
- Log an exception once, where it's handled (not at every layer).
- Use the right level: ERROR = someone should act; WARN = degraded or unexpected; INFO = business events; DEBUG = diagnostics.

## 3. Metrics with Micrometer
Micrometer is a vendor-neutral facade ("SLF4J for metrics"): Prometheus, Datadog, CloudWatch, New Relic…
Auto-captured: `http.server.requests` (count, latency, status, URI), JVM (`jvm.memory.used`, `jvm.gc.pause`, threads), HikariCP pool, Tomcat, caches, logback events.
```java
@Service
public class OrderService {
    private final Counter ordersPlaced;
    private final Timer checkoutTimer;
    OrderService(MeterRegistry registry) {
        ordersPlaced = Counter.builder("orders.placed").tag("channel", "web").register(registry);
        checkoutTimer = Timer.builder("checkout.duration").publishPercentiles(0.95, 0.99).register(registry);
    }
    public void checkout() { checkoutTimer.record(() -> { /* … */ ordersPlaced.increment(); }); }
}
// or annotate: @Timed("checkout.duration") / @Counted  (with the ObservedAspect/TimedAspect bean)
```
Meter types: **Counter** (only increases), **Gauge** (current value: queue size), **Timer** (latency + count), **DistributionSummary** (sizes).
⚠️ **Tag cardinality**: never tag with userId/orderId (unbounded series explode the metrics backend).

### What to monitor (RED + USE)
- **RED** per endpoint: **R**ate, **E**rrors, **D**uration (p95/p99 latency, not the average).
- **USE** per resource: **U**tilization, **S**aturation, **E**rrors (CPU, heap, DB pool active/pending, thread pool queue).
Typical stack: **Prometheus** (scrape `/actuator/prometheus`) → **Grafana** dashboards → **Alertmanager** alerts.

## 4. Distributed tracing
A single request crosses gateway → order-service → payment-service → DB. **Trace** = the whole journey (`traceId`); **span** = one operation (`spanId`, parent id).
```xml
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-otlp</artifactId></dependency>
```
```properties
management.tracing.sampling.probability=0.1     # sample 10% in prod
```
Boot instruments incoming HTTP, `RestClient`/`WebClient`/Feign (propagates the W3C `traceparent` header), JDBC, and Kafka. Spans go to **Zipkin/Jaeger/Tempo**, and `traceId` appears in logs, so one id links logs, metrics, and traces.
The **Observation API** (`@Observed`, `ObservationRegistry`) produces metrics and traces from one instrumentation.

## 5. Debugging a slow API in production (answer template)
1. **Metrics**: is p99 up for one endpoint or all? Error rate? DB pool pending? GC pauses? CPU?
2. **Traces**: which span is slow (DB query? downstream call? lock wait?).
3. **Logs** by `traceId`: errors, retries, timeouts.
4. Narrow it down: slow SQL (enable slow-query log, check for N+1 or a missing index, [40](40_JPA%20Cascading%2C%20Lazy%20Loading%20and%20the%20N%2B1%20Problem.md)), thread dump for blocked threads, heap/GC for memory pressure.
5. Fix, then add an alert or dashboard so it's caught earlier next time.

## 6. Graceful shutdown
```properties
server.shutdown=graceful                          # default in Boot 3.4+
spring.lifecycle.timeout-per-shutdown-phase=30s
```
On SIGTERM: readiness goes DOWN, the app stops accepting new requests, finishes in-flight ones, then closes beans (`@PreDestroy`).

---

## Interview Q&A
**Q1. What is Spring Boot Actuator?**
Production-ready endpoints for health, metrics, info, loggers, env, thread/heap dumps, and so on, plus Micrometer integration.

**Q2. Liveness vs readiness probes?**
Liveness failure → restart the pod. Readiness failure → stop routing traffic. Don't tie liveness to external dependencies.

**Q3. Which Actuator endpoints are risky to expose?**
`env`, `configprops`, `heapdump`, `threaddump`, `beans`, `loggers` (POST). Secure them or keep them internal.

**Q4. How do you correlate logs for one request across services?**
A correlation/trace id in MDC, propagated via headers (W3C traceparent) by Micrometer Tracing. Search logs by `traceId`.

**Q5. What are the three pillars of observability?**
Logs (events), metrics (aggregated numbers over time), traces (request path across services).

**Q6. Counter vs gauge vs timer?**
Monotonic count / current value / duration plus count with percentiles.

**Q7. Why is high-cardinality tagging bad?**
Each unique tag combination is a new time series, which blows up memory and cost in Prometheus.

**Q8. How would you change a log level in production without redeploying?**
`POST /actuator/loggers/com.myapp` with `{"configuredLevel":"DEBUG"}` (secured), or a config server refresh.

## Quick revision
- Actuator: health (+liveness/readiness), metrics, prometheus, loggers. Expose minimally and secure the rest.
- SLF4J + Logback; parameterized logs; exception as the last arg; JSON logs; MDC correlation id; no secrets.
- Micrometer: Counter/Gauge/Timer, RED/USE, p95/p99, low-cardinality tags → Prometheus + Grafana.
- Tracing: traceId/spanId via OpenTelemetry bridge; propagated automatically by Boot clients.
- Graceful shutdown on SIGTERM.
