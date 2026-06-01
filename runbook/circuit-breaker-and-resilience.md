# Circuit Breaker and Resilience

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Purpose

Define the resilience patterns every Messaging Agent runtime must implement to survive production failures — LLM provider outages, channel bridge provider unreliability, webhook floods, and worker crashes.

## Exponential backoff on LLM failure

When a planning worker fails (model timeout, HTTP 429, HTTP 500):

```text
attempt_1: retry after  2^1 × 1000ms = 2s
attempt_2: retry after  2^2 × 1000ms = 4s
attempt_3: retry after  2^3 × 1000ms = 8s
max_attempts: 3
```

Queue configuration:
```
attempts: 3
backoff: { type: 'exponential', delay: 1000 }
```

## Auto-escalation on retry exhaustion

When a job reaches `attempts = max_attempts` and still fails:

```text
1. Move to dead-letter queue
2. Write row to dead_letters table:
     source_type = 'planning'
     error_message = last error
     payload = original job data
3. Emit event: system.failure.escalated
4. Force conversation state to: waiting_human (owner = 'human')
5. Send SOS notification to admin private JID:
     "[SYSTEM ALERT] AI gagal memproses percakapan dengan {{customer_name}}.
      Tolong tangani manual dari nomor bisnis. Error: {{error_summary}}"
6. Send apologetic template message to end user:
     "Maaf Kak, sistem kami sedang mengalami gangguan teknis sementara.
      Tim kami akan segera melanjutkan percakapan ini. Mohon tunggu sebentar 🙏"
```

This ensures the end user is never left in silence and the admin is always aware of system failures.

## Circuit breaker states (provider-level)

For high-volume deployments, implement a circuit breaker per LLM provider:

```text
CLOSED   — Normal operation. Requests flow through.
OPEN     — Provider is failing. All requests immediately routed to:
             (a) fallback model if configured, or
             (b) auto-escalation flow above
HALF_OPEN — After recovery window (60s), allow one test request.
             If successful: transition to CLOSED
             If failed: return to OPEN, extend recovery window
```

Track the circuit state in a low-latency store (Redis or equivalent):
```
key: circuit:llm:{provider}:{tenant_id}
value: CLOSED | OPEN | HALF_OPEN
TTL: 60s for OPEN (auto-reset to HALF_OPEN after window)
```

For single-tenant managed deployments, a full per-tenant circuit breaker is overkill. Implement the retry + auto-escalation flow first. Add circuit breaker state tracking when serving 20+ concurrent active tenants.

## Outbox send failure handling

When the messaging channel bridge rejects or times out a send:

```text
attempt_1: retry after 3s
attempt_2: retry after 9s
attempt_3: retry after 27s (exponential)

On exhaustion:
  - Update outbox_messages.status = 'dead'
  - Write to dead_letters (source_type = 'outbox')
  - Do NOT regenerate from LLM (the content is already in outbox_messages.content)
  - Notify admin: "Pesan tidak berhasil dikirim ke {{customer_name}}. Kirim manual jika perlu."
```

The critical rule: **retries always resend from the outbox row content, never regenerate from LLM.** Regenerating produces a different reply, which breaks conversation continuity.

## Webhook retry flood protection

When the runtime's webhook endpoint has high latency, the messaging channel bridge will re-deliver the same webhook. Combined with idempotency, this is safe. But under extreme load, the retry flood itself can overwhelm the system.

```text
Ingress gateway hardening:
  1. Idempotency check must complete in < 5ms
  2. Webhook endpoint must return HTTP 200 in < 200ms always
  3. Any logic beyond idempotency check goes into the queue
  4. If queue is backing up: shed load by returning 200 without queuing
     (idempotency log already recorded the event; it can be replayed from dead-letter)
```

Idempotency guard pattern (using SET NX with TTL):
```
key:   idempotency:wamid:{provider}:{instance_id}:{wamid}
value: "1"
NX:    true (only set if not exists)
EX:    86400 (24 hours; covers any reasonable retry window)

if SET returns nil → duplicate, return 200 immediately, do nothing
if SET returns "OK" → new event, proceed to enqueue
```

## Stale lock recovery

Conversations can get stuck in `ai_planning`, `ai_sending`, or `waiting_human` indefinitely if a worker crashes mid-execution.

```text
Stale recovery worker (runs every 5 minutes):

For ai_planning stuck > 30s:
  → Transition to failed_recoverable
  → Schedule retry

For ai_sending stuck > 60s:
  → Check outbox row status
  → If outbox shows sent: transition to idle (worker missed the callback)
  → If outbox shows pending: transition to failed_recoverable

For waiting_human stuck > resume_timeout_minutes (tenant config):
  → Transition to resume_pending
  → Build continuity summary
  → Resume AI

For human_active with last_human_activity_at > inactivity_threshold:
  → Transition to resume_pending
  → Build continuity summary
  → Resume AI
```

## Resilience checklist

- [ ] LLM failure triggers exponential backoff, not immediate retry flood
- [ ] Max retry exhaustion auto-escalates to human with SOS notification
- [ ] end user receives apologetic message when AI fails — never silence
- [ ] Outbox retries resend from stored content, never regenerate
- [ ] Idempotency check completes in < 5ms
- [ ] Webhook handler returns 200 in < 200ms regardless of downstream state
- [ ] Stale lock recovery worker runs on schedule
- [ ] Circuit breaker state tracked per provider for scale deployments
- [ ] Dead-letter rows include enough payload for manual replay
- [ ] Admin is notified on system failures, not only on handoff triggers
