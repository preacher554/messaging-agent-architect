# Business Stage Engine

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## Two-layer state architecture

V5 separates runtime state from business stage. These are orthogonal dimensions.

```
RUNTIME STATE — what the system is doing right now
  idle | user_buffering | ai_planning | ai_typing | ai_sending |
  handoff_requested | waiting_human | human_active | resume_pending |
  cooldown | failed_recoverable | retrying | suspended | archived

BUSINESS STAGE — where the end user is in the sales or service journey
  greeting | discovery | qualification | explanation | negotiation |
  waiting_payment | active_client | followup | closed_won | closed_lost |
  escalated
```

A conversation can be in runtime state `user_buffering` AND business stage `negotiation` simultaneously. The runtime state drives what the system does mechanically. The business stage drives how the AI behaves, speaks, and what it focuses on.

## Business stage definitions

### `greeting`
End user has just initiated contact. No prior qualification has happened.
- Warm, natural first message. Ask for name if not known.
- Do not re-introduce the company if `social_memory.introduced_company = true`.
- Do not list all products; focus on opening a discovery conversation.

### `discovery`
AI is learning what the end user needs. No specific product pitch yet.
- Ask open discovery questions (one at a time).
- Listen and classify business type and pain points.
- Do not pitch specific packages yet.

### `qualification`
AI is assessing whether the end user fits the offer, their budget range, and readiness.
- Explore budget sensitivity gently.
- Clarify timeline and decision-making ability.
- Update `lead_temperature` signal. Classify as hot, warm, or cold.

### `explanation`
AI is presenting the relevant product or package in detail.
- Present only the most relevant option based on qualification findings.
- Use the end user's own language (reflect back their stated problem).
- Do not re-explain topics already covered per `social_memory.explained_topics`.

### `negotiation`
End user is interested and discussing price, terms, or conditions.
- Do not reduce price beyond tenant-configured discount limit without approval.
- Record any counter-offers or price points.
- If end user pushes beyond approved discount: trigger handoff, do not improvise.

### `waiting_payment`
End user has agreed to purchase but payment has not been confirmed.
- Send payment instructions once, clearly.
- Follow up once after configured waiting period.
- Do not repeatedly ask about payment (annoying and pushy).

### `active_client`
End user has paid and is receiving or using the service.
- Shift from sales to support and relationship management tone.
- Answer service or usage questions.
- Do not pitch new products too early.

### `followup`
End user was previously qualified but did not convert, or needs re-engagement.
- Reference the prior conversation context; do not start from scratch.
- Use a softer re-engagement message, not a hard pitch.
- Do not repeat the same pitch that did not convert before.

### `escalated`
End user has a complaint, critical issue, or emotional state requiring special handling.
- Do not pitch or qualify.
- Acknowledge the issue explicitly and empathetically.
- Escalate to human immediately if `escalation_risk = critical`.

### `closed_won`
End user converted. Deal is complete.
- Warm, appreciative close. Set clear next steps.
- No further active sales behavior needed.

### `closed_lost`
End user confirmed they will not proceed.
- Polite, non-pushy close. Leave door open for future re-engagement.
- Do not argue or offer last-ditch discounts without approval.

## Stage persistence

Business stage is stored per conversation and persists across runtime state changes, worker restarts, and human takeovers.

```sql
ALTER TABLE conversations ADD COLUMN business_stage TEXT NOT NULL DEFAULT 'greeting';
ALTER TABLE conversations ADD COLUMN stage_updated_at TIMESTAMPTZ;
ALTER TABLE conversations ADD COLUMN stage_history JSONB NOT NULL DEFAULT '[]';
```

## Stage transition rules

```text
Transitions driven by:
  1. AI planner detecting a qualifying event
  2. A runtime event (e.g., payment confirmed → active_client)
  3. Manual override by admin

Each transition must:
  1. Update conversations.business_stage
  2. Append to conversations.stage_history
  3. Emit: stage.transitioned event
  4. Write milestone to memory (e.g., CLOSED_WON, ACTIVE_CLIENT)
```

## Context injection by stage

When building AI context, include the current business stage and a stage-appropriate directive:

```text
[BUSINESS_STAGE]
Current stage: negotiation
End user has expressed interest in the Pro package.
Focus on confirming the deal. Do not reduce price below configured minimum.
[/BUSINESS_STAGE]
```

## Stage-to-tag mapping

| Stage | Auto-assigned tag |
|---|---|
| `qualification` (hot) | Hot Lead |
| `negotiation` | Negotiation |
| `waiting_payment` | Waiting Payment |
| `active_client` | Active Client |
| `escalated` | Escalated |
| `followup` | Followup |
| `closed_won` | Won |
| `closed_lost` | Lost |

## Stage-specific writing directives

| Stage | Writing directive |
|---|---|
| `greeting` | Warm, brief, curious. One question to open discovery. |
| `discovery` | Open-ended, patient. Explore without leading. |
| `qualification` | Direct but gentle. Questions about intent and budget. |
| `explanation` | Structured but conversational. Progressive disclosure. |
| `negotiation` | Confident, calm, clear. No filler or hesitation. |
| `waiting_payment` | Short and operational. Logistics only. |
| `active_client` | Supportive, friendly. Less formal. |
| `followup` | Reference the past. Be specific. Do not restart. |
| `escalated` | Serious, empathetic. Acknowledge before anything else. |

## Checklist

- [ ] Business stage is separate from runtime state (two dimensions)
- [ ] Business stage persists through runtime changes and human takeovers
- [ ] Stage transitions emit runtime events and append to stage_history
- [ ] AI context includes current stage and stage-appropriate directive
- [ ] AI does not re-pitch or re-qualify when stage shows user already passed that point
- [ ] Admin can manually override stage
- [ ] Tag auto-assignment fires on stage transitions
