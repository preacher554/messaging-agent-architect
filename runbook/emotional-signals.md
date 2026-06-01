# Emotional and Urgency Signals

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Why prompt-only detection is not enough

Putting emotional detection only inside the LLM prompt has these limitations:

- Emotional state detected in turn 3 is not tracked between turns unless explicitly restated
- After a human takeover and AI resume, emotional state from before the handoff is lost
- No runtime routing decisions can use emotional state (e.g., skip lead scoring for angry customers)
- No analytics on emotional distribution across tenants
- No escalation threshold logic based on accumulating frustration

Emotional state must be a runtime record in the database, not just an instruction in a prompt.

## Signal types

| signal_type | signal_value options | When to write |
|---|---|---|
| `emotional_state` | angry, frustrated, confused, neutral, positive, urgent | After each AI planning step when reasoning contains emotional inference |
| `urgency` | immediate, high, medium, low | When message contains time-sensitive language |
| `lead_temperature` | hot, warm, cold, unknown | After lead qualification step (Pro package) |
| `escalation_risk` | critical, elevated, normal | When multiple negative signals accumulate |

## Database table

```sql
CREATE TABLE conversation_signals (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  signal_type TEXT NOT NULL,
  signal_value TEXT NOT NULL,
  confidence FLOAT,                -- 0.0 to 1.0
  detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  detected_by TEXT NOT NULL,       -- 'llm', 'rule', 'human_admin'
  raw_evidence TEXT                -- the message or pattern that triggered it
);

CREATE INDEX ON conversation_signals (conversation_id, signal_type, detected_at DESC);
```

## Extraction flow

After each AI planning step, extract signals from the LLM reasoning output:

```text
1. LLM returns reply and optional structured signal block
2. Planner reads signal block
3. For each detected signal:
   INSERT INTO conversation_signals (tenant_id, conversation_id, signal_type, signal_value, confidence, detected_by='llm', raw_evidence)
4. Check for escalation threshold:
   IF recent angry/frustrated signals count >= escalation_threshold:
     set escalation_risk = 'critical'
     auto-trigger handoff without waiting for next AI turn
```

## Prompt instruction for signal extraction

Ask the LLM to return signals as a structured block alongside the end user reply:

```text
After your reply to the end user, output a JSON block:

<signals>
{
  "emotional_state": "neutral | confused | frustrated | angry | positive | urgent",
  "emotional_confidence": 0.0–1.0,
  "urgency": "low | medium | high | immediate",
  "escalation_recommended": true | false,
  "escalation_reason": "string or null"
}
</signals>

Base this only on the end user's actual messages. Do not infer from your own replies.
```

## Rule-based signals (no LLM required for common cases)

Some signals can be detected by keyword rules before the LLM is called, saving latency for common cases:

```text
Urgency patterns (Bahasa Indonesia + English):
  darurat, segera, cepat, emergency, asap, urgent, buru-buru, sekarang juga

Angry patterns:
  kecewa, marah, komplain, tidak puas, menipu, tipu, bohong, parah, buruk banget

Confused patterns:
  bingung, tidak mengerti, gimana, maksudnya apa, tidak paham, bisa dijelasin
```

These fire on message receipt before buffering. If an urgency keyword is detected, set urgency signal immediately and shorten buffer flush time to 0ms.

## Routing decisions that use signals

| Signal | Runtime action |
|---|---|
| `emotional_state = angry` AND `escalation_risk = critical` | Auto-trigger handoff, skip normal handoff threshold |
| `urgency = immediate` | Flush buffer immediately, skip pacing delay |
| `lead_temperature = hot` | Flag for admin notification even without explicit handoff trigger |
| `emotional_state = confused` for 3+ consecutive turns | Suggest simpler language in next AI reply |

## Observability use

Signals stored in `conversation_signals` enable tenant-level analytics:
- Average emotional state distribution per week
- Escalation trigger frequency
- Lead temperature conversion rates
- Time-to-frustration patterns by business type

These become value metrics for the managed service — tenants see that the AI is tracking lead quality and end user sentiment.

## Checklist

- [ ] Signals are written to database, not only used in prompt
- [ ] LLM is prompted to output a structured signal block alongside every reply
- [ ] Rule-based keyword detection fires before LLM call for high-urgency cases
- [ ] Urgency signal triggers immediate buffer flush
- [ ] Escalation threshold check happens after every signal write
- [ ] Tenant dashboard can query signal history
- [ ] Human-detected signals (set by admin) are stored with detected_by='human_admin'
