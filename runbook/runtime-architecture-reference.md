# Runtime Architecture Reference

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Purpose

Define the module separation, queue topology, and message lifecycle that every Messaging Agent runtime should follow. Technology choices (Node.js, Python, Go) are implementation details — the architecture patterns are universal.

## Module responsibilities

### Webhook gateway (entry point)

The webhook handler must do nothing except:
1. Parse and validate the incoming payload
2. Run the idempotency check
3. Classify `business_outbound` events (outbox echo vs human admin)
4. Enqueue the normalized event
5. Return 200

It must NOT call the LLM, call the messaging channel bridge, or do business logic.

### Queue / worker layer

Each worker owns one phase of the message lifecycle. Workers must be idempotent — running twice produces the same result as running once.

```text
message-worker:  webhook event → buffer open/append/flush
planning-worker: buffer flushed → context build → LLM call → signal extract → outbox write
pacing-worker:   outbox created → typing delay → send trigger
outbox-worker:   send result → status update → retry or dead-letter
```

### State machine

Every state transition must:
1. Verify the transition is valid from the current state
2. Update conversation state and timestamp
3. Emit `state.transitioned` to the event log

No module may update conversation state directly. All updates go through the state machine.

### AI planning layer

The planner orchestrates but does not implement:
1. Context assembly (system prompt + tenant knowledge + end user memory + continuity + history + current turn)
2. LLM call
3. Signal extraction from response
4. AI decision persistence
5. Signal writes
6. Outbox row creation

The planner never calls the messaging channel bridge directly. The outbox worker handles all sends.

### Memory layer

Four memory layers with defined read/write timing:
- Conversation memory (session messages)
- end user memory (persistent profiles, leads, history)
- Conversation summaries (cross-session compression)
- Business knowledge (tenant scope: FAQ, catalog, SOP)

### Tenant isolation layer

Every runtime row and event must include `tenant_id`. Never key memory globally by phone number alone. Isolate messaging channel instance, SOP, prompt bundle, catalog, end user data, policy, tools, logs, and billing per tenant.

### Outbox (safe send pattern)

Never call the messaging channel bridge inline in the webhook handler:
1. Generate AI reply text
2. Insert into outbox with status `pending`
3. Return webhook response immediately
4. Worker picks up pending rows, sends via bridge
5. On success → update status to `sent`
6. On failure → increment attempt, schedule retry
7. On exhaustion → dead-letter

### Human-AI collaboration layer

Handoff trigger → admin notification → same-channel human activity detection → pause → continuity summary → resume. AI must never fight for keyboard. If owner is `human`, AI may update memory but must not send.

### Observability layer

Event log as source of truth (insert-only). Per-conversation sequence numbers for deterministic replay. Track state transitions, ownership changes, model latency/cost, outbox status, duplicate rate, dead letters.

## Queue topology

```text
Queue: message-events      FIFO per conversation_id    (inbound webhook processing)
Queue: buffer-flush        Delayed jobs per conversation_id  (buffer timer)
Queue: planning            FIFO per conversation_id    (LLM call + signal extraction)
Queue: outbox-send         FIFO per conversation_id, delayed by pacing  (messaging channel send)
Queue: handoff-notify      Standard, low volume       (admin notifications)
Queue: stale-recovery      Scheduled, periodic        (lock timeout recovery)
Queue: circuit-check       Scheduled, periodic        (circuit breaker state)

Dead-letter: exhausted retries → dead_letters table
```

## Message lifecycle (end-to-end)

```text
messaging channel → Bridge webhook
  ↓
Gateway: parse, validate, idempotency check, business_outbound classify
  ↓
Enqueue: message-events queue
  ↓
message-worker: open/append/flush buffer
  ↓
buffer.flushed event
  ↓
Enqueue: planning queue
  ↓
planning-worker: context build → LLM → signals → outbox write
  ↓
Enqueue: outbox-send queue (delayed by pacing)
  ↓
pacing-worker: typing indicator → wait → send via bridge → update outbox
  ↓
messaging channel delivers to end user
```

## Technology guidance

| Concern | v1 Recommendation | Scale Alternative |
|---|---|---|
| messaging channel bridge | channel bridge provider v2 (Baileys) | Official Meta Cloud API |
| Persistence | Postgres-compatible database | Self-hosted Postgres + PgBouncer |
| Queue + delayed jobs | Any durable queue with delayed job support | Apache Kafka (>50 concurrent) |
| Session cache | Redis or in-memory per worker | Redis Cluster |
| LLM | Any OpenAI-compatible provider | Multi-model via router |
| Runtime | Python (FastAPI) or Node.js (Express) | — |
| Infra | VPS + systemd or Docker | Managed k8s at scale |

## Scale-up path

When active tenant count exceeds ~50 simultaneous conversations, consider migrating the inbound pipeline to a distributed log (e.g., Kafka):

- Partition key: `phone_number` (ensures FIFO per end user across all workers)
- Removes need for per-conversation queue locks at scale
- Significant operational complexity — do not migrate before BullMQ/queue v1 is stable

Do not add infrastructure complexity before the correctness work is complete (state machine, dedupe, handoff, business_outbound classifier).
