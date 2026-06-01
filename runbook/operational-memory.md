# Operational Memory Layer

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## The business continuity problem

When a human admin negotiates a 10% discount on Monday, then the AI resumes on Tuesday and says "Halo Kak, mau tahu lebih soal Paket Pro?" — the business relationship is broken.

Operational memory tracks where the end user is in the **business process**: what they have been offered, what they agreed to, what is pending, what was decided by a human, and what the next expected action is.

This is the CRM layer of the runtime.

## Operational memory schema

```sql
CREATE TABLE operational_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,

  -- Sales pipeline state
  lead_stage TEXT NOT NULL DEFAULT 'new',
  -- new | contacted | qualified | interested | negotiating | payment_pending
  -- | active_client | churned | lost
  lead_temperature TEXT DEFAULT 'unknown',
  -- hot | warm | cold | unknown
  selected_package TEXT,
  quoted_price NUMERIC,
  negotiated_price NUMERIC,
  discount_given NUMERIC,
  discount_type TEXT,                   -- 'percent' | 'absolute'

  -- Payment state
  payment_status TEXT DEFAULT 'none',
  -- none | invoice_sent | pending | confirmed | failed | refunded
  invoice_sent BOOLEAN NOT NULL DEFAULT false,
  invoice_sent_at TIMESTAMPTZ,
  payment_confirmed_at TIMESTAMPTZ,
  payment_method TEXT,

  -- Onboarding and delivery
  onboarding_status TEXT DEFAULT 'not_started',
  -- not_started | in_progress | completed | stalled
  delivery_promised_at TIMESTAMPTZ,
  access_granted BOOLEAN NOT NULL DEFAULT false,

  -- Human decisions
  last_human_decision TEXT,
  last_human_decision_at TIMESTAMPTZ,
  next_expected_action TEXT,

  -- Loss tracking
  loss_reason TEXT,

  stage_entered_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id)
);
```

## Stage definitions for `lead_stage`

| Stage | Meaning | Typical next action |
|---|---|---|
| `new` | First contact, no qualification done | Begin discovery |
| `contacted` | Had initial exchange, intent unclear | Continue discovery |
| `qualified` | Intent confirmed, budget range known | Present relevant package |
| `interested` | Expressed interest in a specific offer | Move to negotiation or confirm |
| `negotiating` | Price or terms being discussed | Hold; do not pitch again |
| `payment_pending` | Deal agreed, waiting for payment | Send invoice, follow up once |
| `active_client` | Payment confirmed, service delivered | Support mode |
| `churned` | Was active but no longer engaged | Re-engagement if appropriate |
| `lost` | Explicitly will not proceed | Polite close |

## Integration with business stage engine

Operational memory and business stage are related but not the same:

- **Business stage** (on `conversations.business_stage`) = current stage of THIS conversation session
- **Operational memory** (on `operational_memories`) = cumulative CRM state across all sessions

A returning end user might start a new session (business stage = `greeting`) but their operational memory already says `lead_stage = payment_pending`. The AI must honor operational memory and skip re-qualification.

## Context injection format

```text
[OPERATIONAL_MEMORY]
Lead stage: negotiating
Interest: Pro package — already explained
Quoted price: 2,500,000
Counter-offer in prior session: 2,000,000
Negotiated price: not yet agreed
Discount authority: max 10% without admin approval (min 2,250,000)
Next action: Resume negotiation — end user was considering over the weekend
Payment status: none
[/OPERATIONAL_MEMORY]
```

## Discount authority enforcement

Before the AI proposes any price concession:

```text
proposed_discount > tenant's max_discount → trigger handoff, do not improvise
proposed_discount <= tenant's max_discount → AI may proceed, write to operational_memory
```

## Follow-up scheduling

When `next_expected_action` contains a time-sensitive action:

```sql
INSERT INTO scheduled_followups (
  tenant_id, conversation_id, customer_channel_id,
  scheduled_at, action_type, context_hint, max_attempts
);
```

The follow-up worker checks the current state before sending. If the situation has changed (e.g., payment already confirmed), the follow-up is skipped.

## Checklist

- [ ] Operational memory queried on every session start
- [ ] AI does not re-qualify an end user already in `negotiating` or later
- [ ] Price quotes and negotiations written to `quoted_price` and `negotiated_price`
- [ ] `last_human_decision` written after every human session
- [ ] Discount limit enforced from tenant config
- [ ] Payment status drives business stage override on session start
- [ ] Follow-up scheduling checks current state before sending
