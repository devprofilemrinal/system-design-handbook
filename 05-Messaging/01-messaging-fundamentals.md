# 01 — Messaging Fundamentals

> **05-Messaging Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Messaging?

Messaging is a way for different parts of a system to communicate by sending and receiving **messages** — discrete units of data — through an intermediary called a **message broker**. Instead of one service calling another directly, it sends a message and moves on.

Think of it like the difference between a phone call and a text message:
- **Phone call (synchronous):** You call someone. You wait until they pick up. You talk. You wait for their reply. If they don't answer, you're stuck.
- **Text message (asynchronous):** You send a message and carry on with your day. They read and respond when they're ready. If they're busy, your message waits — it's not lost.

Messaging in software works the same way. The sender doesn't wait for the receiver. An intermediary holds the message until the receiver is ready.

```
SYNCHRONOUS (direct call):
Service A ──calls──→ Service B
           ←waits──
           (blocked until B responds)
           ←response──

ASYNCHRONOUS (messaging):
Service A ──message──→ [Broker] ──message──→ Service B
          (continues immediately)             (processes when ready)
```

---

## 2. Why Messaging Exists — The Problems It Solves

Synchronous communication is simple and intuitive, but it creates problems at scale.

**Tight coupling:** Service A must know Service B's address, protocol, and must be available at the same time. If B is slow, A is slow. If B is down, A fails.

**Scalability ceiling:** If B can only handle 100 requests/second but A sends 500, A must slow down or start failing. There's no buffer.

**No tolerance for spikes:** A sudden traffic spike hits every downstream service simultaneously. The weakest link collapses first, then cascades.

Messaging solves all three:

| Problem | Messaging Solution |
|---|---|
| Tight coupling | Services don't know about each other; only know about the broker |
| Speed mismatch | Broker buffers messages; consumer processes at its own pace |
| Traffic spikes | Queue absorbs the spike; workers drain it gradually |
| Downstream failure | Messages wait in queue; consumer catches up when it recovers |
| Single point of failure | Producers and consumers are independent; one failure doesn't cascade |

> **Core principle:** A queue decouples the *rate of producing* work from the *rate of consuming* it. This is the highest-leverage use of messaging in system design.

---

## 3. Core Terminology

Before going further, establish the vocabulary — these terms appear in every messaging conversation.

| Term | Definition |
|---|---|
| **Producer** | The service that creates and sends messages |
| **Consumer** | The service that receives and processes messages |
| **Broker** | The intermediary that stores and routes messages (Kafka, RabbitMQ, SQS) |
| **Queue** | A data structure in the broker that holds messages until consumed |
| **Topic** | A named channel (in streaming systems like Kafka) that messages are published to |
| **Message** | The unit of data sent from producer to consumer (payload + metadata) |
| **Acknowledgement (ACK)** | A signal from the consumer that it has successfully processed a message |
| **Dead Letter Queue (DLQ)** | Where messages go when they repeatedly fail processing |
| **Offset** | The position of a message in a partition (Kafka concept) |
| **Consumer Group** | A set of consumers that collectively process messages from a topic |

---

## 4. Synchronous vs Asynchronous Communication

This is the foundational distinction. Everything in messaging builds from it.

### Synchronous Communication

The caller blocks until the callee responds. The two services are coupled in time.

```
Caller → Request → Callee
Caller ← Response ← Callee
(Caller is blocked during this entire time)
```

**When it's right:**
- You need the result before you can continue (login check, payment validation)
- The operation is fast and the callee is reliable
- The user is waiting for an immediate answer

**When it hurts:**
- Callee is slow → caller is slow
- Callee is down → caller fails
- High traffic → callee overwhelmed → caller backs up

### Asynchronous Communication

The sender sends and continues. The receiver processes independently.

```
Producer → Message → [Broker] → Message → Consumer
(Producer continues immediately; Consumer processes at its own pace)
```

**When it's right:**
- The result isn't needed immediately (send email, process image, update analytics)
- Operations that can be delayed without user impact
- High-volume operations that must survive traffic spikes
- Operations across multiple services that should be decoupled

**When it hurts:**
- You genuinely need the result before continuing (use sync instead)
- Debugging is harder — tracing an async flow requires distributed tracing
- Eventual consistency — the consumer hasn't processed yet when you check

---

## 5. Queues vs Streams — The Two Models

Messaging systems come in two fundamental models. They look similar but behave very differently.

### Queue Model (Traditional Message Queue)

Each message is delivered to **one consumer** and then removed. Messages are consumed, not stored long-term.

```
Producer → [Q: M1, M2, M3, M4] → Consumer A gets M1 (deleted)
                                 → Consumer B gets M2 (deleted)
                                 → Consumer A gets M3 (deleted)
                                 → Consumer B gets M4 (deleted)

M1, M2, M3, M4 are gone after consumption.
```

Think of a queue like a task list — you tick off tasks and they're done.

**Tools:** RabbitMQ, Amazon SQS, ActiveMQ

**Best for:** Work queues, task distribution, background jobs, any "do this once" operation.

### Stream Model (Event Stream)

Messages are stored in a **log**. Multiple consumers can read independently, at their own position. Messages are not deleted after consumption — they're retained for a configurable period.

```
Producer → [Log: M1, M2, M3, M4, M5...]
                    ↑           ↑
             Consumer A     Consumer B
             (at offset 2)  (at offset 4)
             "I'm reading   "I'm reading
              M2 now"        M4 now"

Both consumers read independently. M1–M5 remain in the log.
```

Think of a stream like a newspaper — the paper exists regardless of how many people read it, and you can go back and re-read yesterday's edition.

**Tools:** Apache Kafka, Amazon Kinesis, Apache Pulsar

**Best for:** Event sourcing, audit logs, analytics pipelines, fan-out to multiple consumers, data replayability.

| | Queue | Stream |
|---|---|---|
| **Message delivery** | One consumer only | Many consumers independently |
| **After consumption** | Message deleted | Message retained (log) |
| **Replay** | Not possible | Possible (read from any offset) |
| **Ordering** | Per queue | Per partition |
| **Scale** | Scale consumers | Scale partitions |
| **Best for** | Task processing | Event-driven architecture |

---

## 6. Message Delivery Guarantees

Not all messaging systems deliver messages with the same guarantees. Understanding these prevents subtle production bugs.

| Guarantee | Meaning | Risk |
|---|---|---|
| **At most once** | Message delivered zero or one times | May be lost; never duplicated |
| **At least once** | Message delivered one or more times | Never lost; may be duplicated |
| **Exactly once** | Message delivered exactly one time | Never lost, never duplicated (hardest to achieve) |

```
At most once:  Producer sends → broker ACKs → consumer might not receive
               (producer doesn't retry on failure)
               Risk: message loss

At least once: Producer retries until broker ACKs → consumer may receive twice
               Risk: duplicate processing
               Fix: make consumers IDEMPOTENT

Exactly once:  Distributed transaction across producer, broker, consumer
               Very expensive; most systems approximate this
```

> **At least once is the practical default.** It's simpler to implement than exactly once, and the risk (duplicates) is manageable if consumers are idempotent. Make your consumers idempotent and at-least-once delivery becomes effectively exactly-once behaviour.

**Idempotent consumer:** Processing the same message twice has the same result as processing it once.

```
Idempotent:     "Set user 123 email to alice@mail.com"
                → run twice → same result. Safe.

Non-idempotent: "Add £100 to account 123"
                → run twice → £200 added. Dangerous.
                Fix: "Set account 123 balance to £550 if current is £450"
                     (conditional update = idempotent)
```

---

## 7. The Message Lifecycle

Understanding what happens to a message from creation to completion.

```
1. PRODUCED
   Producer creates message with payload + metadata
   Sends to broker

2. STORED
   Broker receives message
   Persists to queue/log (durable storage)
   Acknowledges receipt to producer

3. DELIVERED
   Broker delivers message to consumer
   Message marked "in-flight" (not yet ACK'd)

4. PROCESSED
   Consumer processes the message
   Consumer sends ACK to broker

5. ACKNOWLEDGED / COMPLETED
   Broker marks message as consumed
   Queue: message deleted
   Stream: consumer's offset advanced; message stays in log

FAILURE PATH:
   Consumer fails to process → no ACK sent
   Broker timeout → redelivers message to same or different consumer
   Repeated failures → message moves to Dead Letter Queue (DLQ)
```

The DLQ is critical — it catches messages that repeatedly fail so they don't block the main queue and can be inspected and replayed manually.

---

## 8. Backpressure

Backpressure is what happens when consumers can't keep up with producers. Messages accumulate in the queue faster than they're processed.

```
Producer rate: 10,000 messages/second
Consumer rate: 1,000 messages/second

After 1 second:  9,000 messages queued
After 10 seconds: 90,000 messages queued
After 1 minute:  540,000 messages queued → memory exhaustion → broker dies
```

**Handling backpressure:**

| Approach | Mechanism |
|---|---|
| **Scale consumers** | Add more consumer instances → consume faster |
| **Rate limit producers** | Slow down the source of messages |
| **Drop low-priority messages** | Discard non-critical messages when queue is full |
| **Prioritise** | High-priority messages processed first; low-priority can wait |

> **Queues absorb temporary spikes; they do not solve structural mismatches.** If consumers are consistently slower than producers, no queue size will save you — you must scale the consumers.

---

## 9. How Large Companies Use Messaging

| Company | Tool | Application | Source |
|---|---|---|---|
| **LinkedIn** | Kafka (created it) | Activity streams, metrics pipeline, log aggregation | LinkedIn Eng Blog (public) |
| **Uber** | Kafka | Trip events, surge pricing signals, driver location streams | Uber Eng Blog (public) |
| **Netflix** | Kafka | Playback events, recommendations pipeline, monitoring | Netflix Tech Blog (public) |
| **Amazon** | SQS | Decouples hundreds of services; powers async workflows across AWS | AWS public docs |

---

## 10. Best Practices

- **Make consumers idempotent** — with at-least-once delivery, duplicate messages are inevitable.
- **Always configure a Dead Letter Queue** — messages that repeatedly fail must go somewhere inspectable.
- **Monitor queue depth** — growing queue = consumer can't keep up; alert before it becomes a crisis.
- **Set message TTL** — messages that are too old to be useful should be discarded, not processed.
- **Don't use messaging when you need an immediate response** — async is the wrong tool for synchronous workflows.
- **Keep messages small** — put large payloads in object storage; put the reference in the message.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Non-idempotent consumers | Duplicate messages cause duplicate processing (double charges, double sends) | Design consumers to handle duplicates safely |
| No Dead Letter Queue | Failing messages block the queue or are silently lost | Always configure DLQ |
| Messages too large | Broker memory exhausted; slow transmission | Store payload in S3; send reference URL in message |
| Ignoring queue depth | Silent backlog grows until broker crashes | Monitor and alert on queue depth |
| Async when sync needed | User waits indefinitely for a response that isn't coming | Use sync for operations requiring immediate return |

---

## 12. Interview Questions

1. What is a message broker and what problem does it solve?
2. What is the difference between synchronous and asynchronous communication? When would you use each?
3. What is the difference between a queue model and a stream model?
4. Explain the three message delivery guarantees. Which is most practical?
5. What does "idempotent consumer" mean? Why is it important?
6. What is a Dead Letter Queue and why must every queue have one?
7. What is backpressure and how do you handle it?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Messaging** | Async communication via a broker. Decouples producer and consumer. |
| **Sync vs Async** | Sync = wait for response. Async = fire and continue. |
| **Queue** | One consumer per message. Message deleted after processing. Task model. |
| **Stream** | Many consumers independently. Messages retained. Event model. |
| **At-least-once** | The practical default. Make consumers idempotent. |
| **DLQ** | Catches repeatedly-failing messages. Mandatory. |
| **Backpressure** | Queue absorbs spikes; doesn't fix structural consumer shortfall. |

---

## 14. Cross References

**Prerequisites:** System Design Fundamentals · Scalability (NFR #3) · Fault Tolerance (NFR #6)

**Related Topics:** API Gateway (BB #2) · Consistency (NFR #5)

**What to Learn Next:** 02-message-queues.md · 03-event-streaming.md

---

*System Design Engineering Handbook — 05-Messaging Series*