# Kafka Basics with Spring

> **Part of:** Spring gap topics (63–71) · 🟦 NestJS: `@nestjs/microservices` Kafka transport / kafkajs ≈ Spring for Apache Kafka
> **Related:** [69_Idempotency](69_Optimistic%20and%20Pessimistic%20Locking%20and%20Idempotency.md) (idempotent consumers), [71_Microservices](71_Microservices%20with%20Spring%20Cloud%20and%20Resilience4j.md)

## Why this topic matters
Most product companies use Kafka for event-driven communication: order placed → payment, inventory, notification. Expect "how does Kafka guarantee ordering?", "what happens when a consumer crashes?", "at-least-once vs exactly-once?", and "how do you publish an event reliably after a DB write?" (the outbox pattern).

---

## 1. Core concepts
```
Producer ──► Topic "orders" ─┬─ Partition 0: [m0][m1][m2][m3] ...  ──► Consumer A (group "payment")
                             ├─ Partition 1: [m0][m1][m2] ...      ──► Consumer B (group "payment")
                             └─ Partition 2: [m0][m1] ...          ──► Consumer B
                                    offsets ▲ (per partition, per consumer group)
```
| Term | Meaning |
|---|---|
| **Broker** | A Kafka server; a **cluster** has several |
| **Topic** | A named stream of records (like a table of events) |
| **Partition** | An ordered, append-only log; the unit of **parallelism and ordering** |
| **Offset** | A record's position in a partition |
| **Key** | Decides the partition (`hash(key) % partitions`); the same key always goes to the same partition, so it's **ordered** |
| **Consumer group** | Consumers sharing work; **each partition goes to exactly one consumer in the group** |
| **Replication factor** | Copies of each partition across brokers (leader + followers, ISR) |
| **Retention** | Records are kept for a time or size limit **even after being consumed** (unlike a classic queue) |
Key consequences:
- Ordering is guaranteed **only within a partition**. Use a key (e.g. `orderId`) for per-entity ordering.
- Max useful consumers in a group = number of partitions (extra ones sit idle).
- Different groups each get **all** messages (pub/sub). Consumers in one group split them (queue).
- **Rebalance**: when consumers join or leave, partitions are reassigned (processing pauses briefly).

## 2. Kafka vs RabbitMQ (common question)
| Kafka | RabbitMQ |
|---|---|
| Distributed **log**; consumers pull and track offsets | **Broker** pushes messages; deleted after ack |
| Replay old events, multiple independent consumers | Complex routing (exchanges), per-message acks, priorities |
| Very high throughput, event streaming, event sourcing | Task queues, RPC-style work distribution |

## 3. Spring setup
```xml
<dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId></dependency>
```
```properties
spring.kafka.bootstrap-servers=localhost:9092
# producer
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.producer.acks=all                       # wait for all in-sync replicas
spring.kafka.producer.properties.enable.idempotence=true   # no duplicates from producer retries (default in Kafka 3+)
# consumer
spring.kafka.consumer.group-id=payment-service
spring.kafka.consumer.auto-offset-reset=earliest     # new group starts from the beginning (latest = only new)
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.myapp.events
```
```java
@Bean
NewTopic ordersTopic() { return TopicBuilder.name("orders").partitions(6).replicas(3).build(); }
```

### Producer
```java
public record OrderPlaced(String orderId, Long userId, BigDecimal amount, Instant at) {}

@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderPlaced> kafka;
    OrderEventPublisher(KafkaTemplate<String, OrderPlaced> kafka) { this.kafka = kafka; }

    public void publish(OrderPlaced e) {
        kafka.send("orders", e.orderId(), e)                 // key = orderId → per-order ordering
             .whenComplete((result, ex) -> {                 // send() is async (CompletableFuture, see 59)
                 if (ex != null) log.error("Publish failed for {}", e.orderId(), ex);
                 else log.info("Sent {} to partition {} offset {}", e.orderId(),
                         result.getRecordMetadata().partition(), result.getRecordMetadata().offset());
             });
    }
}
```

### Consumer
```java
@Component
public class PaymentListener {
    @KafkaListener(topics = "orders", groupId = "payment-service", concurrency = "3")  // 3 threads ≤ partitions
    public void onOrder(OrderPlaced event,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset) {
        paymentService.charge(event);     // must be IDEMPOTENT (dedupe by orderId / eventId)
    }
}
```
By default Spring commits the offset **after** the listener returns successfully (`AckMode.BATCH`). Use `AckMode.MANUAL` + `Acknowledgment.acknowledge()` for manual control.

## 4. Delivery semantics
| Semantics | How | Risk |
|---|---|---|
| At-most-once | Commit the offset **before** processing | Crash → message lost |
| **At-least-once** (default, most common) | Process, **then** commit | Crash after processing → **redelivered** → duplicates |
| Exactly-once | Idempotent producer + **transactions** (`read-process-write` Kafka→Kafka), `isolation.level=read_committed` | Only end-to-end *within Kafka*; external side effects (DB, email) still need idempotency |
**Interview line:** "We use at-least-once delivery and make consumers idempotent (processed-events table or upserts keyed by event id)." See [69](69_Optimistic%20and%20Pessimistic%20Locking%20and%20Idempotency.md).

## 5. Error handling: retries and a dead letter topic
```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);           // → "orders.DLT" by default
    var handler = new DefaultErrorHandler(recoverer, new ExponentialBackOff(1000, 2.0) {{ setMaxElapsedTime(10_000); }});
    handler.addNotRetryableExceptions(ValidationException.class);          // bad data → straight to the DLT
    return handler;
}
// Non-blocking retries (retry topics) so one bad message doesn't block the partition:
@RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2))
@KafkaListener(topics = "orders")
public void onOrder(OrderPlaced e) { ... }
@DltHandler
public void dlt(OrderPlaced e) { log.error("Moved to DLT: {}", e.orderId()); }
```
- **Poison pill**: a message that always fails (bad JSON, invalid data). Without a DLT it blocks the partition forever (endless retry). Use `ErrorHandlingDeserializer` for deserialization failures.
- Blocking retries keep order but stall the partition; retry topics don't stall but **break ordering** for that key.

## 6. The dual-write problem and the transactional outbox
```java
@Transactional
public void placeOrder(Order o) {
    orderRepo.save(o);
    kafka.send("orders", ...);   // ❌ DB commit + Kafka send aren't atomic:
}                                //    DB rolls back after the send → phantom event; send fails after commit → lost event
```
**Outbox pattern:**
1. In the **same DB transaction**, insert the order **and** an `outbox` row (`event_id, aggregate_id, type, payload, created_at`).
2. A relay publishes outbox rows to Kafka and marks them sent: a poller (`@Scheduled` + `SELECT … FOR UPDATE SKIP LOCKED`, see [64](64_Async%20and%20Scheduled%20Tasks.md)) or **CDC** (Debezium reading the binlog).
3. Delivery is at-least-once, so consumers dedupe by `event_id`.
(`@TransactionalEventListener(phase = AFTER_COMMIT)` + send avoids phantom events but can still **lose** events on a crash. The outbox is the robust answer.)

## 7. Practical tips
- Choose the **partition count** for peak parallelism (it's hard to reduce later). Keys with skew (one huge customer) create hot partitions.
- Keep listener processing short, or tune `max.poll.interval.ms` / `max.poll.records`. A slow handler causes the consumer to be kicked from the group → rebalance storm.
- Version event schemas (Avro/Protobuf + Schema Registry, or JSON with additive changes only).
- Monitor **consumer lag** (how far behind the latest offset each group is) as the key health metric.
- Testing: `@EmbeddedKafka` (spring-kafka-test) or Testcontainers Kafka.

---

## Interview Q&A
**Q1. How does Kafka guarantee ordering?**
Only within a partition. Records with the same key go to the same partition, so they're ordered per key. There's no global ordering across partitions.

**Q2. What is a consumer group?**
A set of consumers sharing a topic's partitions. Each partition is consumed by one member, giving horizontal scaling. Separate groups each receive all messages.

**Q3. More consumers than partitions?**
The extra consumers stay idle. Parallelism is capped by the partition count.

**Q4. At-least-once vs exactly-once?**
Process-then-commit can redeliver after a crash, so you get duplicates. Exactly-once needs an idempotent producer + transactions + read_committed, and applies only to Kafka-to-Kafka flows. External effects still need idempotent consumers.

**Q5. What happens when a consumer crashes?**
Its session times out, the group rebalances, and its partitions move to other members, which resume from the last **committed** offset (so uncommitted work is reprocessed).

**Q6. How do you handle a message that keeps failing?**
Bounded retries with backoff (blocking or retry topics), then a dead letter topic, with alerting and replay tooling. Non-retryable errors go straight to the DLT.

**Q7. How do you reliably publish an event after saving to the DB?**
The transactional outbox (same-transaction outbox row + relay/CDC), with idempotent consumers.

**Q8. Kafka vs RabbitMQ?**
Kafka is a replayable, partitioned log with high throughput and multiple consumer groups. RabbitMQ is a broker with rich routing, per-message acks, and messages deleted after consumption.

**Q9. What does `acks=all` do?**
The producer waits until all in-sync replicas have the record. Combined with `min.insync.replicas=2`, it gives durability on broker failure.

## Quick revision
- Topic → partitions (ordered logs) → offsets; key → partition → per-key order.
- Consumer group: 1 partition ↔ 1 consumer; parallelism ≤ partitions; rebalances.
- `KafkaTemplate.send(topic, key, value)`; `@KafkaListener(groupId, concurrency)`.
- At-least-once + idempotent consumers; `DefaultErrorHandler` + DLT / `@RetryableTopic`.
- Outbox pattern for DB + event consistency; monitor consumer lag.
