# CRM Tagging and Pipeline

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## Purpose

Tags are runtime-visible markers that influence AI behavior, escalation logic, pacing, and admin prioritization. They bridge the gap between the AI's internal state and an admin's operational view of the pipeline.

Tags are not just labels for reporting. They are behavioral signals that the runtime actively reads.

## Tag definitions

| Tag | Meaning | Applied when |
|---|---|---|
| `New Lead` | First contact, no qualification | `lead_stage = new` |
| `Hot Lead` | High interest, budget fit | `lead_temperature = hot` |
| `Warm Lead` | Interest shown but not ready | `lead_temperature = warm` |
| `Cold Lead` | Minimal engagement | `lead_temperature = cold` |
| `Negotiation` | Price or terms being discussed | `business_stage = negotiation` |
| `Waiting Payment` | Invoice sent, awaiting confirmation | `payment_status = pending` |
| `Active Client` | Paid and onboarded | `lead_stage = active_client` |
| `Followup` | Needs re-engagement | `business_stage = followup` |
| `Escalated` | Complaint or critical emotional state | `escalation_risk = critical` |
| `VIP` | Manually assigned by admin | Admin-only |
| `Priority` | Needs faster response | Admin or SLA auto-assigned |
| `Complaint` | Has raised a service complaint | Keyword or admin |
| `Stalled` | No progress despite follow-up | Auto from inactivity policy |
| `Won` | Deal completed | Payment confirmed |
| `Lost` | Explicitly declined | Stage = lost |

## Tag schema

```sql
CREATE TABLE conversation_tags (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  tag TEXT NOT NULL,
  assigned_by TEXT NOT NULL,           -- 'runtime' | 'admin' | 'system'
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  removed_at TIMESTAMPTZ,
  removal_reason TEXT
);

CREATE INDEX ON conversation_tags (tenant_id, customer_channel_id) WHERE removed_at IS NULL;
CREATE INDEX ON conversation_tags (tenant_id, tag) WHERE removed_at IS NULL;
```

Only one active instance of each tag per conversation per tenant.

## Tag influence on AI behavior

```text
[ACTIVE_TAGS]
Hot Lead — showed strong interest, budget confirmed
Negotiation — do not restart discovery or pitch; focus on closing
[/ACTIVE_TAGS]
```

| Active tag | AI behavior change |
|---|---|
| `Hot Lead` | Prioritize closing; reduce exploratory questions |
| `Waiting Payment` | No new pitches; address payment logistics only |
| `Escalated` | Disable sales; switch to empathy + resolution |
| `VIP/Priority` | Shorten pacing delay by 30% |
| `Complaint` | Acknowledge first in every message; no upsell |
| `Active Client` | Support mode; qualify only when appropriate |
| `Stalled` | Re-engagement approach; reference last interaction |
| `Cold Lead` | Reduce aggression; nurture rather than close |

## Pacing overrides by tag

| Tag | Effect |
|---|---|
| Priority / VIP | T_base reduction: -30%, max delay: 4000ms |
| Escalated / Complaint | T_base: 2.5s, no urgency shortcut |
| Waiting Payment | Follow-up once after configured hours, max 2 attempts |

## Tag auto-assignment rules

```
lead_temperature → hot:        Assign Hot Lead (remove Warm, Cold)
business_stage → negotiation:  Assign Negotiation (remove Hot, Warm)
payment_status → invoice_sent: Assign Waiting Payment (remove Negotiation)
lead_stage → active_client:    Assign Active Client, Won (remove all sales tags)
escalation_risk → critical:   Assign Escalated (override; admin clears)
inactivity > stall threshold:  Assign Stalled
lead_stage → lost:            Assign Lost (remove all active sales tags)
```

## Admin-assigned tags

Admins can assign `VIP`, `Priority`, `Complaint`, and custom tags:

```
TAG <customer_channel_id> VIP
UNTAG <customer_channel_id> STALLED
```

Admin-assigned tags are **never auto-removed** by the runtime.

## Pipeline view

Tags enable a CRM-style pipeline view:

```sql
-- Hot leads right now:
SELECT c.customer_name, c.customer_channel_id, om.lead_stage
FROM conversations c
JOIN conversation_tags ct ON ct.conversation_id = c.id AND ct.removed_at IS NULL
WHERE ct.tenant_id = ? AND ct.tag = 'Hot Lead'
ORDER BY c.last_message_at DESC;
```

## Checklist

- [ ] Conflicting tags removed before assigning replacement
- [ ] Tags injected into AI context as `[ACTIVE_TAGS]` block
- [ ] Behavior matrix enforced (no pitch to Escalated/Complaint)
- [ ] Pacing overrides apply for Priority and VIP
- [ ] Auto-assignment fires on state transitions
- [ ] Admin-assigned tags never auto-removed
- [ ] Pipeline queries available for admin view
