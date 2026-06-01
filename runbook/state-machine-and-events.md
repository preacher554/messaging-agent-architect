# State Machine and Events

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Conversation lifecycle states

```text
idle
user_buffering
ai_planning
ai_typing
ai_sending
handoff_requested
waiting_human
human_active
resume_pending
cooldown
failed_recoverable
retrying
suspended
archived
```

State descriptions:
- `idle` — No active processing. AI allowed to reply when next message arrives.
- `user_buffering` — Buffer window open. Collecting fragments. AI must not plan yet.
- `ai_planning` — LLM call in progress. Conversation locked by AI worker.
- `ai_typing` — Decision ready. Typing indicator active. Pacing delay in progress.
- `ai_sending` — Outbox send in progress.
- `handoff_requested` — AI triggered handoff. Admin notification sent. AI silent.
- `waiting_human` — Human takeover initiated, waiting for admin first reply.
- `human_active` — Admin has sent from business number. AI locked out. Silent observer mode.
- `resume_pending` — Inactivity or release detected. Building continuity summary.
- `cooldown` — 30-second grace window after continuity summary written. If admin sends during cooldown, return to human_active.
- `failed_recoverable` — LLM or send failure. Scheduled for retry with exponential backoff.
- `retrying` — Retry in progress.
- `suspended` — Retries exhausted. Auto-escalated to human. Manual intervention required.
- `archived` — Conversation closed.

## Ownership states

```text
owner = ai | human | system
```

Lifecycle state says what is happening. Ownership says who is allowed to act.

## Core transitions

```text
idle + message.received -> user_buffering
user_buffering + buffer.flush_due -> ai_planning
ai_planning + ai.decision_ready -> ai_typing
ai_typing + pacing.elapsed -> ai_sending
ai_sending + outbox.sent -> idle

handoff.requested -> waiting_human
human.outbound_detected -> human_active
human.inactivity_timeout -> resume_pending
admin.release_requested -> resume_pending
continuity.summary_written -> cooldown
cooldown + cooldown.elapsed (30s) -> idle
cooldown + human.outbound_detected -> human_active  // late admin message

send.failed -> failed_recoverable
retry.scheduled -> retrying
retry.succeeded -> idle
retry.exhausted -> suspended
suspended + system.failure.escalated -> waiting_human  // auto-escalate to human
```

## Silent observer rule

When `owner = 'human'` (states: `human_active`, `resume_pending`, `cooldown`):
- AI workers MAY read all inbound and outbound messages
- AI workers MAY update customer_memories and conversation_signals
- AI workers MUST NOT write to outbox_messages
- AI workers MUST NOT trigger any outbound send
- Any planned reply must be discarded, not queued

## Required runtime events

```text
webhook.received
webhook.duplicate_ignored
message.normalized
message.persisted
message.buffered
buffer.flushed
ownership.lock_acquired
ownership.lock_rejected
ownership.changed
handoff.requested
handoff.notified
human.outbound_detected
ai.plan_started
ai.plan_completed
ai.plan_failed
tool.proposed
tool.approval_requested
tool.approved
tool.rejected
tool.executed
outbox.created
outbox.send_started
outbox.sent
outbox.failed
state.transitioned
memory.summary_updated
deadletter.created
system.failure.escalated
session.lock.expired
cooldown.started
cooldown.elapsed
circuit.opened
circuit.half_opened
circuit.closed
```

## Minimal table concepts

```text
runtime_events          -- insert-only event log (source of truth)
conversations           -- per-conversation state + ownership
messages                -- immutable message log
outbox_messages         -- pending/sent/failed outbound messages
message_buffers         -- in-progress fragment accumulation
webhook_events          -- idempotency log
handoff_sessions        -- handoff lifecycle records
ai_decisions            -- what the AI decided (before execution)
tool_calls              -- tool execution records
tool_approvals          -- human approval records
conversation_summaries  -- continuity + session summaries
customer_memories       -- cross-session end user profiles
dead_letters            -- exhausted retry records
```

## Same-number human detection

When the messaging channel bridge emits `business_outbound=true`:

1. Check whether the message matches a runtime outbox row by provider message ID, request ID, or content_hash/time window
2. If matched, classify as `ai` or `system` outbound
3. If not matched, classify as human admin outbound
4. Set ownership to `human`
5. Record `last_human_activity_at`
6. Keep AI silent until manual release or inactivity policy

Never implement same-channel takeover by dropping every `business_outbound=true` event. That prevents the runtime from detecting human admin activity.
