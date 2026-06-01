# Business Modules

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## Purpose

Business modules are discrete, activatable functional units that extend the runtime with specific business operations. Each module has a defined trigger, execution flow, required permissions, and failure behavior.

Modules are not standalone services. They run inside the planning worker as tool calls, each gated by the tenant capability checker.

## Module registry

| Module | Package | Trigger | Output |
|---|---|---|---|
| Lead Qualification | Pro | Discovery complete | Updates lead_stage, writes lead_temperature signal |
| Follow-up Scheduler | Pro | Stage or payment state change | Creates scheduled_followups row |
| Invoice Reminder | Pro | waiting_payment > configured hours | Sends one reminder message |
| Onboarding Checklist | Pro | lead_stage → active_client | Sends structured onboarding |
| Payment Confirmation | Pro (webhook) | Payment webhook received | Updates status, transitions to active_client |
| Escalation Router | Basic + Pro | Escalation threshold exceeded | Triggers handoff, sends admin SOS |
| Business Hours Handler | Basic + Pro | Message outside business hours | Sends outside-hours message |
| Stall Detector | Pro | Inactivity > stall_threshold_days | Tags as Stalled, optional re-engagement |

## Module: Lead Qualification

**Trigger:** AI completes discovery stage.

1. Extract from context: business type, pain points, budget signals, timeline signals
2. Score against tenant's Ideal Client Profile (ICP)
3. Write result to operational_memory: `lead_temperature`, `lead_stage = qualified`
4. Update conversation_tags: Hot/Warm/Cold Lead

## Module: Escalation Router

**Trigger:** `escalation_risk = critical` OR manual handoff trigger.

1. Set conversation owner = `human`, business_stage = `escalated`
2. Assign tag: Escalated
3. Send non-generic acknowledgment to end user
4. Send SOS to admin_handoff_id
5. Write handoff_sessions row

## Module: Business Hours Handler

**Trigger:** Inbound message outside business_hours.

```text
behavior = inform:  Send outside_hours_message ONCE per session (not per message)
behavior = queue:  Buffer message, process when hours open
behavior = handoff: Trigger human escalation if admin reachable
```

Track `outside_hours_message_sent_at` per conversation to prevent repetition.

## Module: Stall Detector

**Trigger:** Scheduled daily check for conversations with inactivity > stall_threshold_days.

1. Assign tag: Stalled
2. If re-engagement configured: schedule one re-engagement message
3. If already sent and no response: transition to closed_lost (or leave for admin decision)

## Shared module rules

### Idempotency
Use SET NX per `(tenant_id, conversation_id, module, trigger_event_id)` to prevent duplicate module execution.

### Capability check
All modules check tenant capability before execution. Package-gated modules require minimum tier.

### Failure isolation
Module failures must not crash the planning worker. Wrap in try-catch, log to `runtime_events`.

## Database tables

```sql
CREATE TABLE scheduled_followups (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  action_type TEXT NOT NULL,
  scheduled_at TIMESTAMPTZ NOT NULL,
  context_hint TEXT,
  max_attempts INT NOT NULL DEFAULT 2,
  attempt_count INT NOT NULL DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'pending',
  last_attempt_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Checklist

- [ ] Every module is gated by capability check before execution
- [ ] Module failures are isolated and logged
- [ ] Scheduled follow-ups check current state before sending
- [ ] Follow-ups have max_attempts cap
- [ ] Outside-hours message sent once per session
- [ ] Stall detector runs on schedule, not on every webhook
- [ ] Escalation router sends non-generic acknowledgment
- [ ] Every module execution emits a runtime_event
