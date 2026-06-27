# 05 — Kafka vs RabbitMQ

> **05-Messaging Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why This Comparison Matters

Kafka and RabbitMQ are the two most commonly discussed messaging tools in system design interviews. They are often described as alternatives, but they are built on fundamentally different models and solve different problems. Choosing between them is not about preference — it's about matching the tool to the architecture you need.

> **The short answer:**
> - RabbitMQ is a **message broker** — it routes and delivers messages, then discards them.
> - Kafka is a **distributed event log** — it stores events permanently and lets any consumer read from any point.

The right question isn't "which is better?" — it's "which model does my use case need?"

---

## 2. Fundamental Model Difference

This is the most important thing to understand. Everything else follows from it.

### RabbitMQ — The Smart Broker

RabbitMQ is designed around **routing and delivery**. Producers send messages to exchanges; exchanges route messages to queues based on rules; consumers read from queues; messages are deleted after ACK.

```
Producer → Exchange → (routing rules) → Queue A → Consumer 1
                                      → Queue B → Consumer 2
                                      → Queue C → Consumer 3

Message is deleted from the queue after the consumer ACKs.
```

RabbitMQ is "smart broker, dumb consumer" — the broker does the heavy lifting of routing and delivery logic.

### Kafka — The Dumb Broker, Smart Consumer

Kafka is designed around **a distributed log**. Producers append events to a topic; events are stored persistently; consumers read by tracking their own position (offset). The broker does minimal routing — it just stores events in order.

```
Producer → Topic (immutable log) → Consumer A reads at offset 5
                                 → Consumer B reads at offset 2
                                 → Consumer C reads at offset 0 (replaying)

Events are NOT deleted after consumption. They're retained for days/weeks.
```

Kafka is "dumb broker, smart consumer" — the broker stores; the consumer manages its own position and does complex processing.

---

## 3. Side-by-Side Comparison

| Dimension | RabbitMQ | Kafka |
|---|---|---|
| **Model** | Message queue | Distributed event log |
| **Message retention** | Deleted after ACK | Retained for configured period (days/weeks) |
| **Message ordering** | Per queue | Per partition |
| **Replay** | ❌ Not possible | ✅ Yes — reset offset |
| **Multiple consumers** | Via pub/sub exchanges (copies made) | Via consumer groups (same log, different offsets) |
| **Throughput** | High (tens of thousands/sec per queue) | Very high (millions/sec per cluster) |
| **Latency** | Very low (sub-millisecond) | Low (milliseconds) |
| **Message routing** | Rich — direct, topic, fanout, headers exchanges | Simple — by topic + partition key |
| **Consumer model** | Push (broker pushes to consumer) | Pull (consumer polls broker) |
| **Protocol** | AMQP (and others) | Custom binary protocol |
| **Ordering guarantee** | Per queue, strict | Per partition, strict |
| **Horizontal scale** | Sharding (limited); federation | Partitions (unlimited; built-in) |
| **Ecosystem** | General-purpose messaging | Data pipeline, event streaming |
| **Operational complexity** | Medium | High |

---

## 4. When to Choose RabbitMQ

RabbitMQ shines when you need **flexible routing, immediate consumption, and low latency** — and you don't need event replay.

### Use RabbitMQ when:

**Task queue / work distribution:** Background jobs that each need to be done exactly once by exactly one worker. Email sending, image processing, report generation.

```
Web Server → [Queue] → Worker Pool (resize images, send emails, generate PDFs)
Each job done once, then gone.
```

**Complex routing logic:** Route messages to different queues based on content — different message types to different services.

```
Exchange routing:
  order.created  → Payment Queue + Inventory Queue
  order.shipped  → Notification Queue only
  order.error    → Alert Queue + DLQ
```

**Request-Reply pattern:** One service needs a synchronous-feeling response from another service.

**Lowest possible latency:** RabbitMQ delivers messages faster than Kafka when throughput requirements are moderate. Sub-millisecond delivery is achievable.

**Mixed delivery guarantees:** Some messages need at-most-once (fire and forget); some need at-least-once; RabbitMQ supports both flexibly.

**Short-lived messages:** Messages are only useful for a brief window. A task queue for live chat message delivery — old messages are irrelevant.

---

## 5. When to Choose Kafka

Kafka shines when you need **high throughput, event replay, multiple independent consumers, and long-term event storage.**

### Use Kafka when:

**Multiple independent consumers of the same events:** Different services need to process the same events independently without any coupling.

```
"payment-completed" event:
  → Accounting service (books the revenue)
  → Analytics service (updates dashboard)
  → Fraud detection (analyses pattern)
  → Loyalty service (awards points)

All four read from the same Kafka topic independently, at their own pace.
In RabbitMQ, you'd need to copy the message to four queues.
```

**Event replay / reprocessing:** Bug in a consumer? Reset offset and reprocess historical events. Onboarding a new service that needs historical data? Read from offset 0.

**Very high throughput:** Millions of events per second. Kafka's partitioned architecture scales horizontally without limit. RabbitMQ has a throughput ceiling per broker.

**Event sourcing:** Kafka's immutable log is a natural fit for event sourcing architectures.

**Stream processing:** Real-time aggregation, transformation, joining of event streams (with Kafka Streams, Flink).

**Audit log / compliance:** All events retained, ordered, and replayable. Regulatory compliance often requires this.

**Data pipeline / ETL:** Move data from operational systems to data warehouses, analytical systems, ML pipelines.

---

## 6. The Decision Framework

```
Q1: Do consumers need to replay events?
    YES → Kafka (RabbitMQ cannot replay)
    NO  → continue

Q2: Do multiple independent consumer groups need the same events?
    YES → Kafka (native; RabbitMQ requires message copying)
    NO  → continue

Q3: Is throughput > 100K messages/second sustained?
    YES → Kafka (built for this scale)
    NO  → continue

Q4: Do you need complex routing (by type, header, pattern)?
    YES → RabbitMQ (rich exchange routing)
    NO  → continue

Q5: Do you need sub-millisecond delivery latency?
    YES → RabbitMQ (lower latency than Kafka at moderate volume)
    NO  → continue

Q6: Is this a task queue / work queue pattern?
    YES → RabbitMQ (natural fit; messages consumed and deleted)
    NO  → Kafka (default for event-driven architectures)
```

---

## 7. Amazon SQS — The Third Option

In AWS environments, Amazon SQS is often the right answer — especially when you don't need Kafka's advanced features.

| | SQS Standard | SQS FIFO | Kafka |
|---|---|---|---|
| **Ordering** | Best-effort | Strict FIFO | Per partition |
| **Delivery** | At least once | Exactly once | Configurable |
| **Throughput** | Unlimited | 3,000 msg/sec per queue | Millions/sec |
| **Retention** | Max 14 days | Max 14 days | Configurable (unlimited) |
| **Replay** | ❌ | ❌ | ✅ |
| **Operations** | Zero (managed) | Zero (managed) | High (self-hosted) or Medium (MSK) |
| **Cost** | Pay per request | Pay per request | Infrastructure cost |

**Choose SQS when:**
- You're on AWS and want zero operational overhead
- Task queues, job processing, decoupling microservices
- You don't need replay or high-throughput streaming

**Choose Kafka (or Amazon MSK) when:**
- You need replay, high throughput, or complex stream processing
- Multiple consumer groups with independent offsets
- Event sourcing or audit trail requirements

---

## 8. Common Interview Scenarios

**"Design a notification system"**
→ RabbitMQ or SQS. Each notification is a discrete task (send this email/push). Consumed once, deleted. Low-to-moderate volume. Complex routing by notification type.

**"Design a real-time analytics pipeline"**
→ Kafka. High-volume event stream. Multiple consumers (dashboard, alerting, ML). Replay for backfill. Stream processing with aggregations.

**"Design a ride-sharing system"**
→ Kafka for driver location streams (millions of location updates/second). SQS/RabbitMQ for ride request jobs (discrete tasks, one consumer per request).

**"Design a payment system"**
→ Both. Kafka for payment events (audit log, multiple consumers, replay). SQS for payment processing jobs (at-most-once or exactly-once FIFO queue).

**"Design a social media feed"**
→ Kafka for activity events (post created, liked, followed). Fan-out to follower feeds via consumer groups. Replay for new feature rollout.

---

## 9. How Large Companies Made This Choice

| Company | Choice | Reason | Source |
|---|---|---|---|
| **LinkedIn** | Kafka (built it) | High-throughput activity streams; multiple consumers; replay | LinkedIn Eng Blog |
| **Slack** | RabbitMQ (early) → Kafka (at scale) | Started with task queues; moved to streaming at scale | Public talks |
| **Robinhood** | RabbitMQ | Financial order processing; strict delivery guarantees; moderate volume | Public talks |
| **Netflix** | Kafka | Playback events, monitoring, multiple downstream consumers | Netflix Tech Blog |
| **Most startups** | SQS | Zero ops overhead; good enough for most task queue needs | Common pattern |

> **Inferred:** Internal specifics vary; the migration patterns and reasoning are well documented publicly.

---

## 10. Best Practices

- **Use RabbitMQ for task queues where each message is consumed once and routing matters.**
- **Use Kafka for event streams where replay, fan-out, and high throughput are needed.**
- **Use SQS on AWS when you want managed infrastructure and don't need Kafka's advanced features.**
- **Don't default to Kafka for everything** — it has significantly higher operational complexity.
- **Kafka and RabbitMQ are not mutually exclusive** — many systems use both for different purposes.
- **Match the tool to the pattern:** task queue → RabbitMQ/SQS; event streaming → Kafka.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Using Kafka for simple task queues | Unnecessary complexity; RabbitMQ or SQS is simpler and sufficient | Match tool to pattern |
| Using RabbitMQ when replay is needed | Can't replay; data gone after consumption | Use Kafka for any replay requirement |
| Self-hosting Kafka for small scale | Heavy operational burden for minimal benefit | Use managed Kafka (MSK, Confluent) or SQS |
| Expecting global ordering from Kafka | Kafka orders within partitions only | Use partition key to ensure per-entity ordering |
| Treating them as interchangeable | Wrong tool for the job; architectural debt | Understand the model; choose deliberately |

---

## 12. Interview Questions

1. What is the fundamental difference between Kafka and RabbitMQ?
2. When would you choose RabbitMQ over Kafka?
3. When would you choose Kafka over RabbitMQ?
4. Why can Kafka support multiple consumer groups while a RabbitMQ queue cannot easily?
5. Your team needs to replay the last 7 days of events after fixing a bug. Which tool supports this?
6. You need to route messages differently based on their type. Which tool is more naturally suited?
7. For a notification system sending emails — Kafka or RabbitMQ? Why?

---

## 13. Summary

| | RabbitMQ | Kafka |
|---|---|---|
| **Model** | Smart broker; message routing | Dumb broker; event log |
| **After consumption** | Deleted | Retained |
| **Replay** | ❌ | ✅ |
| **Routing** | Rich (exchanges) | Simple (topic + key) |
| **Fan-out** | Via exchange copies | Via consumer groups (efficient) |
| **Latency** | Sub-millisecond | Milliseconds |
| **Throughput** | Tens of thousands/sec | Millions/sec |
| **Best for** | Task queues, complex routing | Event streaming, high throughput, replay |

---

## 14. Cross References

**Prerequisites:** 01-messaging-fundamentals.md · 02-message-queues.md · 03-event-streaming.md · 04-messaging-patterns.md

**Related Topics:** Scalability (NFR #3) · Fault Tolerance (NFR #6) · Database Selection (DB #9)

**What to Learn Next:** 06-Distributed-Systems section

---

*System Design Engineering Handbook — 05-Messaging Series*

---

> **05-Messaging series complete.**
> Covered: Fundamentals · Message Queues · Event Streaming · Messaging Patterns · Kafka vs RabbitMQ