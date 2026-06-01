# Scale-Up Path

> Reference for future scaling. Not a requirement for v1.

## When to consider scaling

| Signal | Threshold |
|---|---|
| Active concurrent conversations | > 50 |
| Webhook events per second | > 100 |
| Queue worker contention | Observable under load |
| Single-worker FIFO violations | Detected in event log |

Do NOT scale infrastructure before Phase 0 and Phase 1 correctness work is complete: state machine, dedupe, handoff, business_outbound classifier, and outbox pattern must be stable first.

## Kafka migration path

When active tenant count exceeds ~50 simultaneous conversations, or throughput exceeds the reliable operating range of the current queue system, consider migrating the inbound pipeline to a distributed log:

### Topic design

```text
Topic: wa-raw-events
  Partition key: null (round-robin; ingress is stateless)
  Purpose: holds raw webhook payloads directly from the messaging channel bridge

Topic: wa-session-events
  Partition key: phone_number (CRITICAL: ensures FIFO per end user across all workers)
  Purpose: holds aggregated, deduplicated, normalized events ready for FSM processing
```

### Consumer group

```text
Consumer group: session-actor-workers
  - Each worker processes one partition
  - FIFO per end user guaranteed by partition key
  - No per-conversation queue lock needed at this scale
```

### Why Kafka over queues at scale

| Concern | Queue (BullMQ/Redis) | Distributed Log (Kafka) |
|---|---|---|
| FIFO guarantee | Application-level, can drift under contention | Broker-enforced by partition |
| Per-conversation ordering | Requires explicit lock | Partition key = phone_number |
| Replay capability | Dead-letter only | Full topic replay from any offset |
| Operational complexity | Low | Significant |
| Infrastructure cost | Redis on existing VPS | Dedicated Kafka cluster |

### Migration checklist

- [ ] Current queue system is stable and correctness-tested
- [ ] Event schema is versioned (forward-compatible)
- [ ] Idempotency keys are durable across replay
- [ ] State machine transitions are deterministic from event sequence
- [ ] Monitoring and alerting covers consumer lag
- [ ] Rollback plan exists (queue system can be re-activated)

### Alternatives to Kafka

| Option | When to consider |
|---|---|
| Redis Streams | Simpler log pattern, survives crashes, but no topic partitioning |
| Database queue + SKIP LOCKED | Works for moderate scale without new infrastructure |
| Self-hosted Kafka | Full control, higher ops burden |
| Managed Kafka (Confluent/Redshift) | Offload ops, higher cost |

## General scaling principles

1. Scale correctness before scale volume — fix state machine and dedupe before adding Kafka
2. Scale horizontally before vertically — add workers before upgrading the server
3. Scale observability first — you cannot tune what you cannot measure
4. Avoid distributed transactions — single-service ownership per conversation is the safest pattern
5. Tenant-level rate limiting prevents one noisy tenant from affecting others
