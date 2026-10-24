# REST Clients: RestClient, WebClient and OpenFeign

> **Part of:** Spring gap topics (63–71) · 🟦 NestJS `HttpModule` (axios) ≈ `RestClient`/`WebClient`
> **Related:** [71_Microservices](71_Microservices%20with%20Spring%20Cloud%20and%20Resilience4j.md) (retries, circuit breakers)

## Why this topic matters
Every backend calls other services or third-party APIs. Interviewers ask which client to use, and especially **timeouts, retries, and failure handling**. "What happens when the payment API is slow?" is a classic.

---

## 1. The options
| Client | Style | Status |
|---|---|---|
| `RestTemplate` | Blocking, template methods | **Maintenance mode** (still works, no new features) |
| **`RestClient`** (Spring 6.1 / Boot 3.2) | Blocking, **fluent** API (like WebClient) | **Recommended for Spring MVC apps** |
| `WebClient` (WebFlux) | Non-blocking, reactive (`Mono`/`Flux`) | Reactive apps, high-concurrency streaming |
| HTTP Interface (`@HttpExchange`) | Declarative interface backed by RestClient/WebClient | Feign-like, built into Spring |
| **OpenFeign** (Spring Cloud) | Declarative interface (`@FeignClient`) | Popular in microservices |

## 2. RestClient
```java
@Configuration
public class ClientConfig {
    @Bean
    RestClient paymentClient(RestClient.Builder builder) {          // Boot-provided builder (Jackson, observability)
        var factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(2));
        factory.setReadTimeout(Duration.ofSeconds(5));                // ALWAYS set timeouts
        return builder
                .baseUrl("https://payments.example.com")
                .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .requestFactory(factory)
                .build();
    }
}

@Service
public class PaymentGateway {
    private final RestClient client;
    PaymentGateway(RestClient paymentClient) { this.client = paymentClient; }

    public PaymentResponse charge(ChargeRequest req) {
        return client.post()
                .uri("/v1/charges")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Idempotency-Key", req.idempotencyKey())          // safe retries (see 69)
                .body(req)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError,
                          (request, response) -> { throw new PaymentRejectedException(response.getStatusCode()); })
                .body(PaymentResponse.class);
    }

    public List<Order> orders(Long userId) {
        return client.get().uri("/users/{id}/orders", userId)
                .retrieve()
                .body(new ParameterizedTypeReference<List<Order>>() {});   // generic type (erasure workaround, see 56)
    }

    public ResponseEntity<Order> withHeaders(Long id) {
        return client.get().uri("/orders/{id}", id).retrieve().toEntity(Order.class);   // status + headers
    }
}
```
- Default error handling: 4xx/5xx → `HttpClientErrorException` / `HttpServerErrorException` (`RestClientResponseException`). Timeouts/IO → `ResourceAccessException`.
- In Boot 3.4+ you can set timeouts with `spring.http.client.connect-timeout` / `read-timeout` (and `ClientHttpRequestFactorySettings`).

### Declarative HTTP interface (built into Spring 6)
```java
public interface OrderApi {
    @GetExchange("/orders/{id}") Order get(@PathVariable Long id);
    @PostExchange("/orders") Order create(@RequestBody CreateOrder body);
}
@Bean
OrderApi orderApi(RestClient.Builder b) {
    RestClient rc = b.baseUrl("http://order-service").build();
    return HttpServiceProxyFactory.builderFor(RestClientAdapter.create(rc)).build().createClient(OrderApi.class);
}
```

## 3. WebClient (reactive)
```java
WebClient web = WebClient.builder().baseUrl("https://api.example.com").build();

Mono<User> user = web.get().uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class)
        .timeout(Duration.ofSeconds(3))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));

// parallel calls, non-blocking
Mono<Dashboard> dash = Mono.zip(userMono, ordersMono).map(t -> new Dashboard(t.getT1(), t.getT2()));
```
In a **Spring MVC** app you *can* use WebClient and `.block()`, but `RestClient` is now the simpler choice. Use WebClient when the app is fully reactive (WebFlux) or needs streaming.

## 4. OpenFeign
```java
@SpringBootApplication
@EnableFeignClients
public class App { }

@FeignClient(name = "inventory-service", url = "${clients.inventory.url}",   // or service discovery by name
             fallbackFactory = InventoryFallbackFactory.class)
public interface InventoryClient {
    @GetMapping("/api/stock/{sku}")
    StockDto getStock(@PathVariable String sku);
}
```
```properties
spring.cloud.openfeign.client.config.default.connect-timeout=2000
spring.cloud.openfeign.client.config.default.read-timeout=5000
```
It integrates with service discovery (load-balanced by name), Resilience4j circuit breakers, and an `ErrorDecoder` for mapping errors.

## 5. Resilience: the part interviewers care about
| Concern | Do |
|---|---|
| **Timeouts** | Always set connect + read timeouts. A call with no timeout can hang a request thread **forever** and exhaust the Tomcat pool |
| **Retries** | Only for **idempotent** operations (GET, PUT, or POST with an Idempotency-Key), only on transient errors (timeouts, 502/503/504, connection reset), with **exponential backoff + jitter** and a max attempt count |
| **Circuit breaker** | Stop calling a failing dependency, fail fast, recover gradually (Resilience4j, [71](71_Microservices%20with%20Spring%20Cloud%20and%20Resilience4j.md)) |
| **Bulkhead** | Limit concurrent calls per dependency so one slow API can't consume all threads |
| **Fallbacks** | Cached or default data, or a graceful error |
| **Connection pooling** | Use Apache HttpClient 5 / Reactor Netty pools rather than one connection per request |
| **Observability** | Propagate trace headers (Micrometer Tracing does this for Boot-built clients), log latency + status |
```java
@Retry(name = "payments")                       // Resilience4j annotations
@CircuitBreaker(name = "payments", fallbackMethod = "chargeFallback")
public PaymentResponse charge(ChargeRequest r) { ... }
```
**Never retry a non-idempotent POST blindly.** You might charge a customer twice.

---

## Interview Q&A
**Q1. RestTemplate vs RestClient vs WebClient?**
RestTemplate is legacy blocking (maintenance mode). RestClient is the modern blocking fluent API (Spring 6.1). WebClient is reactive and non-blocking (WebFlux).

**Q2. What happens if you don't set timeouts?**
Calls can hang indefinitely, tying up request threads until the server pool is exhausted and the app stops responding (cascading failure).

**Q3. When is it safe to retry an HTTP call?**
For idempotent requests (or ones with an idempotency key), on transient failures, with backoff + jitter and bounded attempts.

**Q4. What is OpenFeign?**
A declarative HTTP client: annotate an interface and Spring generates the implementation. It integrates with discovery, load balancing, and circuit breakers.

**Q5. How do you deserialize a generic response like `List<Order>`?**
`ParameterizedTypeReference<List<Order>>() {}`, which captures the type despite erasure.

**Q6. How do you handle 4xx vs 5xx from a downstream?**
Map 4xx to domain errors (don't retry). Treat 5xx/timeouts as transient (retry/circuit-break/fallback). Use `onStatus` or a Feign `ErrorDecoder`.

**Q7. Why might WebClient `.block()` in a WebFlux app be bad?**
It blocks an event-loop thread, defeating non-blocking concurrency (Reactor may throw `IllegalStateException`).

## Quick revision
- RestClient for MVC apps; WebClient for reactive; Feign / `@HttpExchange` for declarative clients.
- Always set connect/read timeouts; use connection pools.
- Retry only idempotent + transient errors with backoff and jitter; add circuit breakers and bulkheads.
- `ParameterizedTypeReference` for generics; `onStatus` for error mapping.
