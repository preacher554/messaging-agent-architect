# Continuity and Memory

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## The continuity gap

When a human admin takes over a conversation and the AI later resumes, the AI re-enters with its original system prompt context. It has no knowledge of:

- What the admin promised the end user
- What price or discount was negotiated
- What the end user's current emotional state is
- What the next action the end user is expecting

Without a continuity summary, the AI will contradict what the admin said. In business conversations where negotiation is common, this destroys end user trust immediately.

## Continuity summarizer — required on every resume

Before transitioning from `resume_pending` to `idle`, the runtime must:

```text
1. Collect all messages from the human_active session window:
   SELECT * FROM messages
   WHERE conversation_id = ?
     AND source = 'human_admin'
     AND received_at BETWEEN handoff_session.human_replied_at AND handoff_session.released_at
   ORDER BY received_at ASC

2. Call LLM with a continuity prompt:

   SYSTEM: You are a conversation continuity assistant.
   USER:
     The following messages were sent by a human admin while the AI was paused.
     Extract and summarize:
     1. Any commitments or promises made (price, discount, timeline, specific action)
     2. The end user's current emotional state
     3. What the end user is now waiting for or expecting
     4. Recommended first AI message after resuming

     Messages:
     {{human_session_messages}}

3. Write summary to conversation_summaries table

4. Transition conversation to idle, owner = ai

5. Inject summary into next AI context window as a special block:

   [HUMAN_ADMIN_SESSION]
   The following happened while AI was paused:
   {{summary.content}}
   Recommended action: {{summary.recommended_action}}
   [/HUMAN_ADMIN_SESSION]
```

## Memory types

The runtime maintains four distinct memory layers per end user, per tenant.

### 1. Conversation memory (session scope)

The message history of the current conversation. Included in every AI context window. Used for short-term continuity.

```
Stored as rows in the messages table.
Context builder selects last N messages (typically 20–40) ordered by received_at.
```

### 2. end user memory (persistent across conversations)

Key facts about the end user that persist between separate conversation sessions. Updated after each session ends or at significant events.

```sql
CREATE TABLE customer_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  memory_type TEXT NOT NULL,
  -- 'profile'     : name, business type, location, preferences
  -- 'lead'        : lead_status, package_interest, budget_signal, pain_point
  -- 'history'     : previous purchases, inquiries, resolved issues
  -- 'continuity'  : summaries of past human sessions
  content TEXT NOT NULL,
  version INT NOT NULL DEFAULT 1,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id, memory_type)
);

-- Key pattern (never global by phone number alone):
-- tenant_id + customer_channel_id together = isolated per tenant
```

### 3. Conversation summaries (cross-session summary)

A rolling summary generated at the end of longer sessions, used to compress context without losing important facts.

```sql
CREATE TABLE conversation_summaries (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  summary_type TEXT NOT NULL,
  -- 'session_end'      : generated when conversation goes idle for extended period
  -- 'human_session'    : generated on resume_pending transition (continuity use case)
  -- 'handoff_context'  : generated when handoff is triggered
  content TEXT NOT NULL,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 4. Business knowledge (tenant scope)

The tenant's product catalog, FAQ, SOP, sales script, objection responses. Loaded once per tenant and injected into every AI context as the base knowledge layer. Not per-end user.

Stored in tenant config files and loaded at conversation start.

## Context assembly order

When building AI context for a planning step, assemble layers in this order:

```text
1. System prompt (persona + package rules)
2. Tenant knowledge block (business profile, FAQ, approved prices, handoff rules)
3. end user memory block (profile, lead status, past interactions)
4. Human session continuity block (if resuming after human takeover)
5. Recent conversation history (last 20–40 messages)
6. Current merged turn (from buffer flush)
```

Each block is clearly labeled so the LLM can distinguish sources.

## Memory write timing

| Event | Memory Action |
|---|---|
| end user provides name | Update `customer_memories` type=profile |
| Lead status classified | Update `customer_memories` type=lead |
| Session idle for 2 hours | Write `conversation_summaries` type=session_end |
| Handoff triggered | Write `conversation_summaries` type=handoff_context |
| Human session ends | Write `conversation_summaries` type=human_session |
| Conversation archived | Update `customer_memories` type=history |

## Tenant isolation rule

Memory keys must always include `tenant_id`. Never look up end user memory by phone number alone. A end user calling two different businesses (tenants) must have completely separate memory records.

```text
WRONG:  customer_memories WHERE customer_channel_id = '+62812xxxx'
CORRECT: customer_memories WHERE tenant_id = 'tenant_xyz' AND customer_channel_id = '+62812xxxx'
```

## Silent observer rule

When `owner = 'human'` (states: `human_active`, `resume_pending`, `cooldown`):

- AI workers MAY read all inbound and outbound messages
- AI workers MAY update `customer_memories` and `conversation_signals`
- AI workers MUST NOT write to `outbox_messages`
- AI workers MUST NOT trigger any outbound send
- Any planned reply must be discarded, not queued

## Checklist

- [ ] Continuity summarizer runs before every resume_pending → idle transition
- [ ] Human session messages are collected from messages table, not reconstructed from memory
- [ ] Continuity summary is injected as a labeled block, not appended raw
- [ ] end user memory includes lead status, profile, and history separately
- [ ] Memory lookups always include tenant_id
- [ ] Context assembly follows a defined layer order
- [ ] Rolling summaries are written at session end to avoid context overflow
- [ ] Human session summaries are stored in conversation_summaries for audit
- [ ] Silent observer behavior is enforced at the outbox writer level, not only in the planning prompt
