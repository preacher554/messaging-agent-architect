# Runtime Capability Checklist

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

Use this checklist when auditing a Messaging Agent runtime. Each category maps to a minimum tier:

| Tier | Categories |
|---|---|
| Basic | Webhook ingestion, Conversation processing, Tenant isolation |
| Basic+ | Message buffering, Human takeover (basic) |
| Pro | Human takeover (full), Continuity, Humanization |
| Pro+ | Emotional signals, Circuit breaker, Observability |
| Custom | Tool safety, Kafka scale-up, Admin release |

## Webhook ingestion

- [ ] Normalizes provider payloads into one internal event shape
- [ ] Persists raw webhook payload before processing
- [ ] Uses stable idempotency keys `(provider, instance, message_id, event_type)`
- [ ] Returns quickly (< 200ms) and delegates work to a queue
- [ ] Handles duplicate, late, replayed, and out-of-order events
- [ ] Distinguishes one-to-one chats from groups, broadcasts, and unsupported channels
- [ ] Does not block on model calls or outbound sends
- [ ] Does not globally drop `business_outbound` events (need them for human takeover)

## Conversation processing

- [ ] Uses per-conversation FIFO queueing
- [ ] Uses conversation-level locks or optimistic state versioning
- [ ] Avoids concurrent AI replies for the same end user
- [ ] Supports stale lock recovery (ai_planning stuck > 30s, ai_sending stuck > 60s)
- [ ] Stores inbound, outbound, system, and human events immutably
- [ ] Keeps raw events immutable and derived state rebuildable from event log

## Message buffering

- [ ] Buffers short successive user messages
- [ ] Resets timer on new message
- [ ] Has a max wait cap (recommended: 8000ms)
- [ ] Flushes sooner for urgent or angry messages
- [ ] Merges fragments into one turn before AI planning
- [ ] Does not allow AI planning to start while buffer window is open

## Human takeover

- [ ] Supports same-channel human reply
- [ ] Processes `business_outbound` events (classifies: outbox echo vs human admin vs system)
- [ ] Maintains an outbound ledger to distinguish AI from human messages
- [ ] Sets owner to `human` after unrecognized same-channel outbound activity
- [ ] Prevents AI from sending while human owns the conversation (hard lock)
- [ ] 1-hour human window: no proactive send on timer expiry, only enables next-inbound eligibility
- [ ] Writes a continuity summary before AI resumes
- [ ] Handles cooldown state (grace window after continuity summary before AI acts)

## Tenant isolation

- [ ] Every data table has `tenant_id` where data could cross tenant boundaries
- [ ] Memory keys include tenant ID and end user ID (never phone number alone)
- [ ] Prompt/SOP/policy bundles are tenant-scoped
- [ ] Tool permissions are tenant-scoped
- [ ] Observability and billing are tenant-scoped
- [ ] RLS enabled before any admin UI is exposed

## Orchestration

- [ ] Separates webhook handler, planner, memory builder, policy engine, and sender
- [ ] Uses model routing per tenant with fallback model support
- [ ] Does not regenerate a new answer when only message sending failed (retry from outbox content)
- [ ] Stores AI decisions before executing side effects
- [ ] Uses durable delayed jobs for buffering, pacing, and retry timing

## Tool safety

- [ ] Tools have manifests with risk levels
- [ ] Inputs are schema-validated
- [ ] Dangerous actions (payment, discount, booking, refund, mutation) require approval
- [ ] Tool calls are idempotent
- [ ] Results are persisted and summarized

## Observability

- [ ] Emits structured runtime events (insert-only event log)
- [ ] Each event has per-conversation sequence number for time-travel debugging
- [ ] Tracks state transitions and ownership changes
- [ ] Tracks model latency, cost, and errors per tenant
- [ ] Tracks outbox send status and retry counts
- [ ] Has dead-letter review and replay
- [ ] Alerts on conversations stuck in ai_planning, ai_sending, waiting_human, or failed_recoverable

## Humanization and pacing

- [ ] Response delay is calculated from message length, not a constant
- [ ] Urgency signals shorten delay; angry signals add a small increase
- [ ] Typing indicator before every outbound message
- [ ] Long messages split at paragraph boundaries (280 char threshold)
- [ ] Pacing waits use durable delayed jobs, not ephemeral timers

## Continuity on AI resume

- [ ] Continuity summarizer runs before every resume to active transition
- [ ] Human session messages collected from messages table (not reconstructed)
- [ ] Summary injected as labeled block in AI context window
- [ ] Context assembly follows defined layer order (system → tenant → end user → continuity → history → turn)

## Business stage engine (V5 — Pro/Custom)

- [ ] Business stage is separate from runtime state (two dimensions)
- [ ] Business stage persists through runtime changes and human takeovers
- [ ] Stage transitions emit runtime events
- [ ] AI context includes current stage and stage-appropriate directive
- [ ] AI does not re-pitch or re-qualify when stage shows the end user already passed that point
- [ ] Admin can manually override stage
- [ ] Tag auto-assignment fires on stage transitions

## messaging channel writing engine (V5 — Pro/Custom)

- [ ] One idea per bubble enforced at output layer
- [ ] Repetitive greeting suppressed after session message 1
- [ ] One question per bubble (multiple questions split)
- [ ] Bubble length capped at 280 characters
- [ ] Banned corporate phrases detected and removed
- [ ] Emoji density adapts to business stage and relationship warmth
- [ ] Generic closers replaced with contextual questions

## Business modules (V5 — Pro/Custom)

- [ ] Module capability check before execution
- [ ] Module failures isolated from planning worker
- [ ] Scheduled follow-ups check current state before sending
- [ ] Follow-ups have max_attempts cap
- [ ] outside-hours message sent once per session only
- [ ] Stall detector runs on schedule

## Social and operational memory (V5 — Pro/Custom)

- [ ] Social memory queried before every AI planning step
- [ ] AI does not re-introduce company if already introduced
- [ ] explained_topics prevents full re-explanation
- [ ] relationship_warmth controls greeting style
- [ ] Operational memory queried on session start to determine initial stage
- [ ] Price quotes and negotiations written to operational memory
- [ ] Discount limit enforced from tenant config
- [ ] Payment status drives business stage override on session start

## CRM tagging and pipeline (V5 — Pro/Custom)

- [ ] Conflicting tags removed before assigning replacement
- [ ] Tags injected into AI context as [ACTIVE_TAGS] block
- [ ] Behavior matrix enforced (no pitch to Escalated/Complaint)
- [ ] Admin-assigned tags never auto-removed
- [ ] Pipeline queries available for admin view

## Circuit breaker and resilience

- [ ] LLM failures use exponential backoff, not immediate retry flood
- [ ] Retry exhaustion auto-escalates to human with SOS notification
- [ ] end user receives apologetic message on AI failure — never silence
- [ ] Admin receives SOS notification on system failure
- [ ] Stale lock recovery worker runs on schedule
