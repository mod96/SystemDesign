## Using Kafka at Each Stage (Service A -> Kafka -> Service B -> Kafka -> Service C)

When you introduce Kafka at each stage, it handles:

- Message Deduplication: Kafka's `idempotent producer` and `transactional API` ensure that even if a message is sent multiple times, it will be processed only once.

- Atomicity and Fault Tolerance: With transactions, you can ensure that messages are only delivered when processing is completed successfully, and you avoid partial failures.

By having Kafka at each step, you gain:

- Simplified exactly-once guarantees.
- Built-in durability, replayability, and fault tolerance.

This setup is common in microservice architectures where maintaining data integrity and consistency is crucial across distributed services.

## Idempotency at Each Service:

Ensure that each service (A, B, C) can handle receiving the same message more than once without causing unintended side effects. This usually involves maintaining a unique message ID and tracking processed IDs in a database to avoid reprocessing.

## Transactional Outbox Pattern:

Each service writes messages/events to an outbox table within the same database transaction as its data changes. A separate process then reads from this outbox table and sends messages to the next service or data store. This ensures that data changes and message production are atomic, avoiding inconsistencies.

## Distributed Transactions (2PC/XA Transactions):

Use distributed transaction protocols like Two-Phase Commit (2PC) if multiple services need to participate in a single atomic transaction. However, this approach can be complex, slower, and prone to bottlenecks, especially in large distributed systems. [link in the book](https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/)

## Event Sourcing and Change Data Capture (CDC):

Instead of pushing events/messages directly, services write their changes to a log or database that other services can consume in an exactly-once manner. Tools like Debezium can capture changes from databases and publish them to downstream systems, ensuring that data is processed exactly once.

---

# Trade-offs and Considerations

- Complexity: Implementing exactly-once semantics without Kafka requires careful handling of idempotency, transactional consistency, and failure scenarios. This can increase the complexity of your system significantly.

- Reliability: Kafka is designed for distributed, fault-tolerant messaging with built-in exactly-once support. Relying on custom implementations might introduce subtle bugs or corner cases that could affect data integrity.

- Performance: Distributed transactions (e.g., 2PC) can impact performance and scalability, making them less suitable for high-throughput scenarios.

# When to Use Kafka in the Middle vs. Not

- Use Kafka at Each Stage: If you need a robust, fault-tolerant, and scalable solution for exactly-once delivery, especially in data processing pipelines with multiple stages or microservices.

- Avoid Kafka at Each Stage: If you have strict latency requirements, lower throughput, or if introducing Kafka would add unnecessary complexity, but be prepared to handle idempotency and consistency yourself.

---

# How does the kafka transaction provides exactly once (in single cluster)

Kafka provides exactly-once semantics (EOS) through its combination of the idempotent producer, transactional API, and consumer isolation levels. Let’s break down how Kafka transactions achieve exactly-once delivery, even in complex scenarios involving message consumption and production.

## 1. The Role of the Idempotent Producer

When `enable.idempotence=true` is set on the Kafka producer, it ensures that each message sent to a partition is assigned a unique sequence number. The broker uses this sequence number along with the producer’s ID to track duplicates and prevent double writes if the producer retries.

This provides exactly-once semantics per partition in terms of production, as the broker will detect and _discard duplicate messages due to retries_.

## 2. Kafka Transactions and Exactly-Once Processing

The Kafka `transactional API` extends the idempotent producer to provide exactly-once semantics across multiple operations, _even in the presence of failures_. Here’s how it works:

### Producer Workflow with Transactions:

#### 1. Start a Transaction:

The producer starts a transaction using the `beginTransaction()` method. This marks the beginning of a group of operations that should be treated as a single atomic unit.

#### 2. Producing Messages Atomically:

During the transaction, the producer can send messages to multiple partitions or even multiple topics. These messages are temporarily invisible to consumers operating in `read_committed` mode until the transaction is completed.

#### 3. Commit or Abort:

If everything succeeds, the producer commits the transaction using `commitTransaction()`. This makes all the messages sent within this transaction visible to consumers.

If an error occurs, the producer can call `abortTransaction()`, which discards all the messages produced during that transaction, ensuring they are never delivered to consumers.

### Consumer Workflow with Transactions:

#### 1. Reading Committed Messages:

To achieve exactly-once semantics, consumers should operate in `read_committed` mode (`isolation.level=read_committed`). This setting ensures that the consumer reads only messages from fully committed transactions and ignores those from ongoing or aborted transactions.

#### 2. Offset Management:

When using transactions, _the producer can also commit consumer offsets_ as part of the transaction. This is crucial for exactly-once semantics because it ensures that:

- The offset commit and message production are treated as a single atomic operation.
- If a transaction is aborted, both the message production and offset commit are rolled back, preventing double-processing of messages.

## 3. End-to-End Exactly-Once Processing

The combination of transactional producers, read_committed consumers, and offset management enables true end-to-end exactly-once processing, even in scenarios where messages are consumed from one topic, processed, and then produced to another topic.

Here’s how it works in a typical consume-process-produce scenario:

- The consumer reads messages from the source topic in `read_committed` mode.
- The producer starts a transaction and processes the consumed messages.
- Within the same transaction, the producer sends the processed messages to the destination topic and commits the consumer's offsets.
- Finally, the producer commits the transaction. At this point, both the produced messages and the consumer's offset are atomically committed.

```java
producer.beginTransaction();
Object m = consumer.readMessage();
process(m);
producer.produce(message);
producer.sendOffsetsToTransaction(offsets, consumerGroupId); //
producer.commitTransaction();
```

If any failure occurs during this process (e.g., the producer crashes or network issues arise), the entire transaction can be retried or aborted without any risk of partial processing or duplication, ensuring exactly-once delivery.

### How sendOffsetsToTransaction() Works

The `sendOffsetsToTransaction()` method ties the offset commits to the ongoing transaction. When `commitTransaction()` is called, Kafka ensures that both the produced messages and the consumer offsets are committed together.

If the process crashes at any point before `commitTransaction()` is called, the entire transaction (including both the produced messages and the offset commit) is aborted.
