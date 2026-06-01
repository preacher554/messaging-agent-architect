# Handoff Runtime Design v2

> Upgraded from v1 after production validation. Adds `business_outbound` classifier, outbox echo handling, 1-hour human window with context-preserving resume.

## Default model

Messaging Agent uses **admin notification + same-channel human reply**.

The client's end user continues chatting with the business messaging channel number. When the AI cannot or should not answer, it pauses and notifies the tenant admin on the admin's private messaging channel number.

The admin then replies to the end user using the same business messaging channel/agent number, so the experience stays inside one official business identity.

## Flow (v2)

```txt
1. End user sends message to business messaging channel number.
2. Agent receives message through webhook.
3. Agent dedupes, checks package scope, tenant knowledge, confidence, and escalation rules.
4. If no escalation is needed, AI replies normally via outbox.
5. If escalation is needed:
   a. AI generates short handoff reply → enqueue outbox → send to end user.
   b. Runtime creates handoff summary.
   c. Runtime sends notification to admin_handoff_destination.
   d. Runtime transitions state: waiting_human, owner: human.
   e. AI stops auto-replying to that conversation.
6. Admin replies to end user from the same business/number.
   a. Webhook arrives with key.business_outbound = true.
   b. Runtime checks: is this an outbox echo (AI's own outbound) or human admin message?
   c. If human admin message: persist, transition state → human_active, record last_human_activity_at.
7. During 1-hour human window:
   a. End user inbounds within 1 hour → stored, no AI reply.
   b. New business_outbound:true events reset the 1-hour timer.
   c. Timer expiry does NOT trigger a proactive AI message.
8. Resume on next end user inbound after 1-hour window:
   a. Runtime transitions state → ai_active, owner: ai.
   b. Resume note injected into model prompt (continue from context, not restart).
   c. Human outbound messages labeled as "Admin [TenantName]" in history, not as AI replies.
```

## Same-number business_outbound classifier

Critical mechanism: admin and AI share one messaging channel number. The bridge emits sent-message events with `key.business_outbound: true`.

```txt
business_outbound: true received
  ↓
Match against outbox_messages (pending/sent)?
  ├─ Yes → AI's own outbound echo → mark outbox sent, no action
  └─ No → Human admin message
       ├─ Persist as outbound (source: human_admin)
       ├─ Transition waiting_human → human_active
       ├─ Update last_human_activity_at
       ├─ Emit ownership.changed → human
       └─ Return without AI generation
```

Pitfall: do not globally drop `business_outbound: true` before conversation lookup. The event is operationally meaningful for state transitions.

## Ownership model

| Owner | Meaning |
|---|---|
| `ai` | AI is allowed to generate replies |
| `human` | Human admin has taken over; AI must stay silent |
| `system` | System-level actions (init, cleanup, scheduled resume) |

Ownership enforces hard boundaries: when `owner: human`, no model call is made regardless of inbound volume.

## Conversation states (v2)

```yaml
ai_active:
  description: AI can respond normally. Default state.

waiting_human:
  description: Handoff notification sent. AI must not answer client except waiting messages.

human_active:
  description: Human admin has taken over via business_outbound:true. AI stays silent. 1-hour window starts.

resolved:
  description: Human/admin closed the issue. AI can resume if released or timeout policy allows.

idle:
  description: Conversation exists but no active processing.

user_buffering:
  description: Accumulating short successive messages before AI planning.

ai_planning:
  description: Model call in progress for reply generation.

ai_sending:
  description: Reply generated, sending via outbox worker.

failed_recoverable:
  description: Transient failure (model timeout, network). Will retry.

suspended:
  description: Admin or system policy suspended this conversation.
```

## 1-hour human window + resume policy

1. AI escalates and sets conversation to `waiting_human`.
2. Admin replies manually from the same messaging channel Business number.
3. Runtime detects `business_outbound: true`, transitions to `human_active`, records `last_human_activity_at`.
4. During the 1-hour window, end user messages are stored but AI stays silent.
5. Timer expiry does NOT trigger an outbound message. It only enables AI on next end user inbound.
6. When end user sends a new message after the 1-hour window:
   - Runtime transitions `ai_active`, owner: `ai`
   - Resume context injected into prompt
7. Any new `business_outbound: true` resets the 1-hour timer.
8. Resume should feel natural — not a fresh greeting, but a contextual continuation.

## Handoff keyword detection

- Use **word boundary matching** to avoid false positives.
- Keyword `custom` must NOT match inside `client` or similar words.
- Keyword list defined per tenant in `handoff-rules.md`.

## Admin notification payload

```yaml
event_type: handoff_requested
tenant_id:
conversation_id:
client_name:
client_phone:
escalation_reason:
lead_status:
last_message:
summary:
recommended_next_action:
ai_state: waiting_human
reply_instruction: "Balas dari nomor messaging channel bisnis/agent yang sama."
```

## Resume prompt engineering

When AI resumes after human takeover, the history fed to the model must:

1. **Distinguish human admin messages from AI messages**: Label `business_outbound: true` outbound as `Admin [TenantName]`, not as the AI agent.
2. **Inject resume context**: Operational note explaining the resume situation.
3. **Preserve full conversation window**: Include the human admin's messages.
4. **Avoid restart loops**: If history contains 3+ messages from the human window, acknowledge them naturally.

## Why v2 over v1

| Aspect | v1 | v2 |
|---|---|---|
| business_outbound handling | Global drop | Conversational classifier with outbox matching |
| Timer behavior | Manual release or vague timeout | 1-hour fixed window, no proactive send |
| Resume quality | Restart risk | Context-preserving with prompt engineering |
| Ownership | Implicit in state | Explicit Owner dimension |
| State coverage | 4 states | 10 lifecycle states |

## MVP requirement (v2)

Every Messaging Agent runtime must support:

- per-tenant `admin_handoff_destination`
- per-conversation lifecycle state + ownership
- `business_outbound: true` classification (outbox echo vs human admin)
- AI pause on handoff with ownership lock
- admin notification message
- 1-hour human window with no-proactive-send semantics
- context-preserving resume on next inbound
- word-boundary handoff keyword detection
