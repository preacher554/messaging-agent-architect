# Social Memory Layer

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## The social continuity problem

A messaging channel AI that re-introduces the company on the third message feels broken. A messaging channel AI that asks "Halo, ada yang bisa dibantu?" to someone it already greeted two hours ago feels like a machine with amnesia.

Social memory tracks what has been **socially established** — what has been communicated, explained, and mutually acknowledged. Not what was purchased (that's operational memory), but what is socially known.

## Social memory schema

```sql
CREATE TABLE social_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  -- Social state
  introduced_company BOOLEAN NOT NULL DEFAULT false,
  known_customer_name TEXT,
  known_customer_role TEXT,              -- owner, staff, buyer
  known_business_type TEXT,
  relationship_warmth TEXT NOT NULL DEFAULT 'new',
  -- new | acquainted | warm | returning | loyal
  explained_topics JSONB NOT NULL DEFAULT '[]',
  -- [{topic, explained_at, depth: brief|detailed}]
  known_customer_intent TEXT,
  known_customer_pain_points JSONB NOT NULL DEFAULT '[]',
  prior_questions JSONB NOT NULL DEFAULT '[]',
  -- [{question, asked_at, was_answered}]
  interaction_count INT NOT NULL DEFAULT 0,
  last_session_summary TEXT,
  preferred_language TEXT DEFAULT 'id',
  preferred_formality TEXT DEFAULT 'auto',
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id)
);
```

## Key fields

### `relationship_warmth`

Controls AI greeting and opening style:

| Value | AI behavior |
|---|---|
| `new` | Full warm introduction, ask for name, explain who you are |
| `acquainted` | Use name, skip company intro, reference prior topic |
| `warm` | Casual re-engagement, pick up where conversation left off |
| `returning` | Acknowledge return, ask what's new, gentle context refresh |
| `loyal` | Minimal formalities, treat as familiar contact |

Warmth upgrades: new → acquainted (first exchange) → warm (3+ conversations) → returning (30+ days inactive then re-engage) → loyal (multiple transactions).

### `explained_topics`

Prevents re-explaining what the end user already heard. Before explaining any topic, check this list. If `depth: detailed`, skip full explanation and ask if they want a refresh.

### `prior_questions`

When an end user asks the same question twice, the AI knows the previous answer did not satisfy them. Triggers a different response approach — more direct, more specific — not the same explanation repeated.

## Anti-repetition rules

```text
[SOCIAL_MEMORY_RULES]
1. Do NOT re-introduce the company if introduced_company = true.
2. Do NOT use "Halo! Saya adalah AI dari [company]" if interaction_count > 0.
3. Do NOT repeat any topic from explained_topics with depth=detailed.
   Instead: "Kita udah bahas soal X sebelumnya ya, Kak. Ada yang mau dikonfirmasi?"
4. Always use the end user's name if known_customer_name is set.
5. If relationship_warmth = warm or loyal: use casual tone, skip formal openers.
6. If prior_questions contains the same question being asked now:
   respond with a different framing, not the same answer.
[/SOCIAL_MEMORY_RULES]
```

## Write timing

| Event | Social memory update |
|---|---|
| AI sends greeting and end user responds with name | `known_customer_name`, `relationship_warmth = acquainted` |
| AI finishes explaining a topic | Append to `explained_topics` |
| End user asks a question | Append to `prior_questions` |
| Session ends (idle > 2h) | Increment `interaction_count`, write `last_session_summary` |
| End user re-engages after 30+ days | Set `relationship_warmth = returning` |

## Context injection format

```text
[SOCIAL_MEMORY]
End user name: Budi Santoso
Known business type: online fashion seller
Relationship warmth: warm — use casual, skip formal intro
Company already introduced: yes
Topics already explained: paket_pro, cara_integrasi
Stated intent: wants to automate messaging channel for their end users
Prior unanswered question: "Bisa dipakai untuk 2 nomor sekaligus?"
[/SOCIAL_MEMORY]
```

## Checklist

- [ ] Social memory is queried before every AI planning step
- [ ] AI does not re-introduce the company if already introduced
- [ ] `explained_topics` prevents full re-explanation
- [ ] `prior_questions` triggers different response, not repetition
- [ ] `relationship_warmth` controls greeting style
- [ ] Updated after session ends (not mid-session, to avoid partial writes)
- [ ] Isolated per `(tenant_id, customer_channel_id)` — never global by phone
