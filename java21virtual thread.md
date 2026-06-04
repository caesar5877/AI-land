你提出的方向是合理的，但需要稍微调整表述：

Java 21 Virtual Threads 可以帮助 notification platform 提高 I/O-bound message processing 的并发能力，从而缓解 consumer lag。
但是，它不是 Kafka consumer lag 的完整解决方案，也不应该直接替换 Kafka consumer poll thread。

对于每分钟几十万到几百万 events 的 notification platform，更好的方案是：Kafka partition-level horizontal scaling + poll/processing 解耦 + Virtual Thread worker + bounded concurrency + durable outbox + downstream isolation。

⸻

1. 先明确 Virtual Thread 能解决什么问题

Virtual Thread 适合大量 concurrent tasks 等待 blocking I/O，例如：

* 查询 Cassandra
* 调用 GKS
* 调用 REST / Feign downstream API
* 调用 webhook endpoint
* 等待 partner response
* 写入数据库
* 等待 Kafka producer acknowledgement

当 Virtual Thread 遇到 blocking I/O 时，JVM 通常可以暂时释放底层 carrier thread，让它处理其他任务。因此，同样数量的 OS threads 可以支撑更多 concurrent tasks。Oracle 也明确说明：Virtual Threads 适合大量 waiting tasks，不适合 long-running CPU-intensive operations；它们提高的是 throughput，而不是让单个任务执行得更快。 

可以用一个简单模型估算需要多少并发任务：

Required Concurrency ≈ Events Per Second × Average Processing Time

例如，每分钟 100 万条 events，大约是每秒 16,700 条：

每条 event 的平均处理时间	需要支撑的并发任务数
50 ms	约 835
100 ms	约 1,670
500 ms	约 8,350
1 second	约 16,700

如果平台大量等待 downstream I/O，Virtual Threads 很有价值。
但是，如果每条 event 主要做 JSON parsing、复杂计算、加解密或 CPU-heavy transformation，Virtual Threads 不会明显改善 throughput。

⸻

2. 不要把 Kafka Consumer 本身改成大量 Virtual Threads

Kafka Consumer 不是 thread-safe。一个 consumer instance 不能被多个 Virtual Threads 同时调用。Apache Kafka 官方文档明确说明，多线程并发访问 consumer 可能产生 ConcurrentModificationException。 

因此，不应该这样设计：

Virtual Thread 1 -> consumer.poll()
Virtual Thread 2 -> consumer.poll()
Virtual Thread 3 -> consumer.commitSync()
Virtual Thread 4 -> consumer.pause()

推荐使用 hybrid model：

Platform Thread
    |
    |-- Kafka consumer.poll()
    |-- partition assignment
    |-- pause / resume
    |-- offset commit
    |
    +----> Virtual Thread 1 -> process Event A
    +----> Virtual Thread 2 -> process Event B
    +----> Virtual Thread 3 -> process Event C
    +----> Virtual Thread N -> process Event N

Kafka 官方文档本身也推荐：当 message processing time 波动较大时，可以把 processing 移到其他 threads，让 consumer 继续调用 poll()；但是必须谨慎处理 offset commit，并在必要时 pause partition，防止 offset 提前提交。 

⸻

3. Notification Platform 的推荐架构

对于你们这种大流量平台，我不建议让一个 consumer 同时完成所有事情：

Consume Kafka event
  -> Cassandra
  -> GKS
  -> webhook
  -> partner response
  -> commit offset

如果 partner webhook 变慢，整个 consumer pipeline 会累积 backlog，consumer lag 会迅速升高。

更好的方式是拆成几个独立 stages：

                  ┌──────────────────────────┐
                  │  Source Kafka Topic      │
                  │  Notification Events     │
                  └─────────────┬────────────┘
                                │
                                ▼
        ┌───────────────────────────────────────────┐
        │ Event Processor                           │
        │                                           │
        │ Platform Threads: Kafka poll loop         │
        │ Virtual Threads: enrichment / I/O work    │
        │                                           │
        │ - Validate event                          │
        │ - Deduplicate by event ID                 │
        │ - Query GKS / Cassandra if needed         │
        │ - Write durable partner event / outbox    │
        └──────────────────────┬────────────────────┘
                               │
                               ▼
        ┌───────────────────────────────────────────┐
        │ Channel-Specific Outbox Topics            │
        │                                           │
        │ - push-notification-outbox                │
        │ - webhook-outbox                          │
        │ - email-outbox                            │
        │ - sms-outbox                              │
        └───────┬───────────────┬───────────────────┘
                │               │
                ▼               ▼
       ┌────────────────┐  ┌────────────────────┐
       │ Push Workers   │  │ Webhook Workers    │
       │ Virtual Thread │  │ Virtual Thread     │
       │ per delivery   │  │ per delivery       │
       └───────┬────────┘  └─────────┬──────────┘
               │                     │
               ▼                     ▼
       ┌───────────────────────────────────────────┐
       │ Delivery Status / Retry Topic / DLQ       │
       │ Response Log / Reconciliation             │
       └───────────────────────────────────────────┘

这种设计有几个好处：

* Kafka ingestion 不会被 partner webhook latency 拖慢。
* Push、webhook、email、SMS 可以独立扩容。
* 不同 downstream 可以配置不同的 concurrency limit。
* retry 不会长期占用正常 processing capacity。
* 你们已有的 Cassandra、GKS、webhook 和 SAF replay 机制可以自然地接入这个模型。
* consumer lag 可以分 stage 监控，更容易找到真正的瓶颈。

⸻

4. 第一层扩容能力仍然是 Kafka Partitions

Virtual Threads 不能替代 Kafka partitions。

Kafka consumer group 会把 partitions 分配给 consumers；同一个 consumer group 内，一个 partition 在同一时间只能分配给一个 consumer。Kafka 通过这个机制实现水平扩展和 fault tolerance。 

例如：

Topic: notification-events
Partitions: 96
Application instances: 12
Consumers per instance: 8
Total consumers: 96

这时，每个 consumer 处理一个 partition，Kafka 可以充分利用 96-way parallelism。

但是，如果 topic 只有 12 个 partitions：

Topic partitions: 12
Application consumers: 96

真正工作的最多仍然只有 12 个 consumers，其余 consumers 没有 partition 可以处理。

所以你的 capacity planning 应该先回答：

1. Peak event rate 是多少？
2. 每个 partition 能处理多少 events/sec？
3. 每个 consumer instance 能稳定处理多少 partitions？
4. 是否需要增加 partitions？
5. 是否存在 hot partition？
6. Partition key 应该使用 accountId、notificationId，还是 partnerId？

如果相同 customer 或 account 的消息需要保持顺序，应该使用稳定的 partition key。Kafka 对同一个 partition 内的消息提供顺序保证。 

⸻

5. 第二层才是 Virtual Thread Worker

推荐把每一个独立的 notification-processing task 交给一个 Virtual Thread：

private final ExecutorService workers =
        Executors.newVirtualThreadPerTaskExecutor();
public void dispatch(NotificationEvent event) {
    workers.submit(() -> processEvent(event));
}

Oracle 的 adoption guide 不建议创建 Virtual Thread pool。Virtual Thread 很轻量，正确模型是 one task, one virtual thread。如果需要限制下游并发，应使用 Semaphore，而不是把 Virtual Threads 放入固定大小的 thread pool。 

例如，webhook partner 最多只能承受 500 个 concurrent requests：

private final Semaphore webhookPermits = new Semaphore(500);
public void deliverWebhook(Notification notification) {
    webhookPermits.acquireUninterruptibly();
    try {
        webhookClient.send(notification);
    } finally {
        webhookPermits.release();
    }
}

不同 downstream 使用独立限流：

Cassandra concurrency limit:  1,000
GKS concurrency limit:          500
Webhook Partner A:              300
Webhook Partner B:            1,000
Push notification provider:   2,000

这非常重要。Virtual Threads 可以轻松创建大量 concurrent tasks，但 Cassandra、HTTP connection pool、GKS 和 webhook partner 的容量仍然有限。Virtual Threads 的目标不是无限制增加并发，而是在资源允许范围内减少 OS-thread bottleneck。Oracle 也指出，数据库 connection pool 本身已经相当于一个 semaphore。 

⸻

6. 必须增加 Backpressure

对于百万级 event pipeline，最危险的做法是：

poll 10,000 records
  -> create 10,000 Virtual Threads
  -> downstream 变慢
  -> poll another 10,000 records
  -> create another 10,000 Virtual Threads
  -> memory continues growing

Virtual Threads 虽然轻量，但并不是免费的。必须设置：

Global in-flight limit
Per-partition in-flight limit
Per-downstream concurrency limit
High watermark
Low watermark
Pause / resume rule

推荐流程：

1. Consumer poll records
2. Register records as in-flight
3. Submit processing tasks to Virtual Threads
4. If in-flight count reaches high watermark:
       pause affected partitions
5. Virtual Threads complete processing
6. Poll thread drains completion queue
7. Commit completed offsets
8. If in-flight count drops below low watermark:
       resume affected partitions

Spring Kafka 提供 listener container 的 pause() 和 resume()。暂停后 container 仍然继续调用 poll()，从而避免不必要的 rebalance，但不会继续拉取新 records。 

⸻

7. Offset Commit 必须使用 “Highest Contiguous Completed Offset”

并发处理时，event 完成顺序可能与 Kafka offset 顺序不同：

Partition 0:
Offset 100 -> completed
Offset 101 -> still processing
Offset 102 -> completed
Offset 103 -> completed

此时只能 commit 到 offset 101 之前。不能直接 commit 104，否则应用 crash 后，offset 101 对应的 event 可能丢失。

推荐为每个 partition 建立 offset tracker：

Partition 0:
  completed: 100, 102, 103
  pending:   101
Safe commit position:
  101

当 101 完成以后，才可以向前 commit：

Safe commit position:
  104

Kafka 官方文档也强调：将 processing 移到其他 thread 后，需要关闭 auto commit，只在实际完成处理之后提交 offset，防止 committed offset 超过真正完成的位置。 

如果你们使用 Spring Kafka，可以评估 asyncAcks。它允许 records 以不同顺序 acknowledge，并延迟 offset commit，直到前面的 gap 被补齐；但失败后出现 duplicate delivery 的可能性会提高。 

⸻

8. Offset 不要等到 Webhook 真正发送成功后才提交

对于 notification platform，建议把 durable boundary 放在 outbox：

Consume source event
  -> validate
  -> enrich
  -> write durable partner event / delivery outbox
  -> commit Kafka offset

然后由另外的 delivery workers 异步完成：

Read outbox event
  -> call webhook / push provider
  -> write response log
  -> mark delivery status
  -> retry or DLQ when necessary

这与你们已有的 Cassandra event store、SAF replay 和 webhook response log 思路是一致的。

核心原则是：

Kafka ingestion consumer 只需要确保 event 已经 durable，不应该等待外部 partner 完成最终 delivery。

这样，即使 partner webhook 慢、timeout 或暂时不可用，source topic 的 consumer lag 也不会无限增长。

如果 workflow 只是 Kafka topic A 转换后写入 Kafka topic B，可以评估 Kafka transactions。Kafka 官方文档说明，transactions 可以支持向多个 Kafka topics 和 partitions 原子写入，并由 read_committed consumer 只读取成功 commit 的消息。 

但是，如果中间还需要写 Cassandra，Kafka transaction 无法覆盖 Cassandra。此时更实际的是：

At-least-once delivery
+ durable outbox
+ eventId deduplication
+ replay
+ reconciliation

⸻

9. Java 21 必须增加 Pinning Guardrails

Java 21 的 Virtual Threads 已经是正式特性，但是仍然存在 pinning 风险。

当 blocking I/O 发生在以下场景时，Virtual Thread 可能无法释放 carrier thread：

synchronized block / synchronized method
native method
foreign function

Pinning 不一定造成功能错误，但是频繁且耗时较长的 pinning 会降低 scalability。Oracle 建议使用 JFR 的 jdk.VirtualThreadPinned event 或 -Djdk.tracePinnedThreads=full 定位问题；对于频繁且耗时较长的 synchronized I/O，可以有针对性地替换为 ReentrantLock。 

建议在 Java 21 上默认开启监控：

-Djdk.tracePinnedThreads=full

在 performance environment 运行 JFR：

jfr print \
  --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd,jdk.VirtualThreadPinned,jdk.VirtualThreadSubmitFailed \
  recording.jfr

Java 24 的 JEP 491 已经改善了 synchronized 导致的 pinning 问题，因此 Java 25 rollout 仍然应该保留在 roadmap 中。 

⸻

10. 建议的实施 Milestones

Milestone 1 — Bottleneck Analysis

先确认 consumer lag 主要来自哪里：

Kafka partition 数量不足？
Consumer instance 数量不足？
Platform thread pool saturation？
Cassandra latency？
GKS latency？
Webhook latency？
Connection pool wait time？
CPU saturation？
GC pause？
Hot partition？
Retry storm？

只有当 thread saturation + blocking I/O 是明显瓶颈时，Virtual Threads 才是高优先级优化。

⸻

Milestone 2 — Partition and Batch Optimization

先建立稳定 baseline：

- Tune topic partition count
- Tune consumer instance count
- Tune consumers per instance
- Tune max.poll.records
- Evaluate batch processing
- Measure hot partitions
- Separate normal flow, retry flow and DLQ

Kafka 提供 max.poll.records 控制每次 poll() 返回的最大 records 数量，也提供 max.poll.interval.ms 控制 polling 的 liveness 边界。 

⸻

Milestone 3 — Hybrid Virtual Thread PoC

只替换 processing layer：

Keep:
  Kafka poll thread = platform thread
Change:
  Message-processing task = virtual thread per task
Add:
  in-flight limit
  semaphore per downstream
  pause / resume
  offset tracker
  feature toggle

不要一开始修改 topic partition、consumer count、batch size 和 Virtual Threads。一次只改变一组变量，否则 benchmark 无法说明真正的 improvement 来源。

⸻

Milestone 4 — Split Durable Processing and Delivery

把 webhook、push notification 和 partner delivery 移出 source-event consumer path：

Source consumer
  -> durable outbox
  -> commit
Delivery workers
  -> external call
  -> status
  -> retry / DLQ

对于 notification platform，这一步通常比单独开启 Virtual Threads 更重要。

⸻

Milestone 5 — Canary and Java 25 Upgrade

Java 21 上先做：

lower environment
-> representative load test
-> JFR pinning analysis
-> canary traffic
-> rollback toggle

Java 25 upgrade 完成以后，再扩大 production rollout 范围。

⸻

11. 需要监控的指标

不要只观察总体 throughput。建议建立四组 dashboard。

Category	Metrics
Kafka	consumer lag、records-consumed-rate、records-lag-max、rebalance count、commit latency
Processing	processed events/sec、p50/p95/p99 processing time、in-flight tasks、paused partitions、retry count
JVM	CPU、heap、GC pause、platform thread count、virtual thread count、VirtualThreadPinned、VirtualThreadSubmitFailed
Downstream	Cassandra pool wait、GKS latency、webhook latency、timeout rate、partner error rate、DLQ volume

Kafka Java client metrics 可以通过 JMX 暴露，官方文档也列出了 records-consumed-rate 及其对应的累计指标。 

⸻

12. 可以拿去和 Architect 讨论的结论

你可以这样表达：

For this notification platform, I don’t think Virtual Threads should be positioned as a direct Kafka optimization. The Kafka poll loop should remain on platform threads because Kafka Consumer is not thread-safe.

The better design is a hybrid model: scale Kafka consumption through partitions and consumer instances, then use one Virtual Thread per independent I/O-bound processing task. We also need bounded in-flight processing, per-downstream semaphores, partition pause and resume, and contiguous offset tracking.

For the notification workflow, I recommend separating durable event ingestion from final webhook or push delivery. Once the event is written to the durable outbox, the source offset can be committed. Delivery workers can then use Virtual Threads independently, with retries, DLQ, replay, and reconciliation.

On Java 21, we should run JFR pinning analysis and keep a feature toggle. Java 25 remains the preferred target for broader rollout because it includes the JEP 491 pinning improvement.

最简化的一句话是：

Use Virtual Threads to scale blocking delivery work, not to replace Kafka partitioning, consumer-group scaling, backpressure, or durable outbox design.