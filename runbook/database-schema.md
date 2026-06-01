# Core Database Schema

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

Use this as the reference schema for a Messaging Agent runtime. Every table that holds tenant data must include `tenant_id`. Apply RLS when any admin UI or multi-tenant API is added.

## Tenants

```sql
CREATE TABLE tenants (
  id TEXT PRIMARY KEY,
  business_name TEXT NOT NULL,
  package TEXT NOT NULL CHECK (package IN ('basic', 'pro', 'custom')),
  channel_instance TEXT NOT NULL,
  admin_handoff_id TEXT,
  resume_policy TEXT NOT NULL DEFAULT 'timeout' CHECK (resume_policy IN ('timeout', 'manual', 'both')),
  resume_timeout_minutes INT NOT NULL DEFAULT 60,
  buffer_window_ms INT NOT NULL DEFAULT 1500,
  max_buffer_wait_ms INT NOT NULL DEFAULT 8000,
  memory_retention_days INT NOT NULL DEFAULT 180,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Conversations

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL REFERENCES tenants(id),
  customer_channel_id TEXT NOT NULL,
  customer_name TEXT,
  state TEXT NOT NULL DEFAULT 'idle',
  owner TEXT NOT NULL DEFAULT 'ai' CHECK (owner IN ('ai', 'human', 'system')),
  last_message_at TIMESTAMPTZ,
  last_human_activity_at TIMESTAMPTZ,
  state_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON conversations (tenant_id, state);
CREATE INDEX ON conversations (tenant_id, customer_channel_id);
```

## Messages (immutable event log)

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  direction TEXT NOT NULL CHECK (direction IN ('inbound', 'outbound', 'system')),
  source TEXT NOT NULL CHECK (source IN ('end_user', 'ai', 'human_admin', 'system')),
  content TEXT NOT NULL,
  provider_message_id TEXT,
  outbox_id UUID,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON messages (conversation_id, received_at);
CREATE INDEX ON messages (tenant_id, received_at);
```

`direction` = direction relative to business channel.  
`source` = actor who produced the message.

| Case | direction | source |
|---|---|---|
| Customer message | inbound | end_user |
| AI reply | outbound | ai |
| Human admin same-number reply | outbound | human_admin |
| System/internal note | system | system |

For context-preserving resume after handoff, include rows with `source = 'human_admin'` in model history and label them as `Admin [TenantName]`, not as AI.
This source enum is aligned with `wa-agent-dashboard` issue #2 migration contract.

## Outbox messages

```sql
CREATE TABLE outbox_messages (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT NOT NULL,        -- for business_outbound matching
  source TEXT NOT NULL DEFAULT 'ai' CHECK (source IN ('ai', 'system')),
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'sending', 'sent', 'failed', 'dead')),
  provider_message_id TEXT,
  attempt_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON outbox_messages (conversation_id, status);
CREATE INDEX ON outbox_messages (status, next_retry_at) WHERE status IN ('pending', 'failed');
```

## Message buffers

```sql
CREATE TABLE message_buffers (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  messages JSONB NOT NULL DEFAULT '[]',
  flush_at TIMESTAMPTZ NOT NULL,
  flushed_at TIMESTAMPTZ,
  merged_content TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON message_buffers (conversation_id) WHERE flushed_at IS NULL;
```

## Webhook events (idempotency log)

```sql
CREATE TABLE webhook_events (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  provider TEXT NOT NULL,
  instance_id TEXT NOT NULL,
  message_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  raw_payload JSONB NOT NULL,
  processing_status TEXT NOT NULL DEFAULT 'received'
    CHECK (processing_status IN ('received', 'processing', 'processed', 'failed')),
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(provider, instance_id, message_id, event_type)
);
```

## Runtime events (observability, insert-only)

```sql
CREATE TABLE runtime_events (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT,
  sequence BIGINT NOT NULL,          -- per-conversation sequence for deterministic replay
  event_type TEXT NOT NULL,
  payload JSONB NOT NULL DEFAULT '{}',
  emitted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON runtime_events (tenant_id, conversation_id, sequence ASC);
CREATE INDEX ON runtime_events (tenant_id, conversation_id, emitted_at DESC);
CREATE INDEX ON runtime_events (event_type, emitted_at DESC);
```

### Time-travel debugging

With sequence numbers, replaying a conversation to any point in time is deterministic:

```sql
-- Reconstruct conversation state up to event sequence N:
SELECT * FROM runtime_events
WHERE conversation_id = 'tenant_x:+62812xxx'
  AND sequence <= N
ORDER BY sequence ASC;

-- Find the exact moment AI and human collided:
SELECT sequence, event_type, payload, emitted_at
FROM runtime_events
WHERE conversation_id = ?
  AND event_type IN ('human.outbound_detected', 'outbox.sent', 'ownership.changed')
ORDER BY sequence ASC;
```

## Handoff sessions

```sql
CREATE TABLE handoff_sessions (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  reason TEXT NOT NULL,
  lead_status TEXT,
  summary TEXT,
  admin_notified_at TIMESTAMPTZ,
  human_replied_at TIMESTAMPTZ,
  released_at TIMESTAMPTZ,
  release_reason TEXT CHECK (release_reason IN ('timeout', 'manual', 'admin_command')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON handoff_sessions (conversation_id, created_at DESC);
```

## Conversation signals

```sql
CREATE TABLE conversation_signals (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  signal_type TEXT NOT NULL,
  signal_value TEXT NOT NULL,
  confidence FLOAT,
  detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  detected_by TEXT NOT NULL CHECK (detected_by IN ('llm', 'rule', 'human_admin')),
  raw_evidence TEXT
);

CREATE INDEX ON conversation_signals (conversation_id, signal_type, detected_at DESC);
```

## Client memories (cross-session)
> Note: Table name `customer_memories` is retained for backward compatibility. In the center documentation use "client".

```sql
CREATE TABLE customer_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  memory_type TEXT NOT NULL,
  -- 'profile' | 'lead' | 'history' | 'continuity'
  content TEXT NOT NULL,
  version INT NOT NULL DEFAULT 1,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id, memory_type)
);

CREATE INDEX ON customer_memories (tenant_id, customer_channel_id);
```

## Conversation summaries

```sql
CREATE TABLE conversation_summaries (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  summary_type TEXT NOT NULL,
  -- 'session_end' | 'human_session' | 'handoff_context'
  content TEXT NOT NULL,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON conversation_summaries (conversation_id, summary_type, created_at DESC);
```

## Dead letters

```sql
CREATE TABLE dead_letters (
  id UUID PRIMARY KEY,
  tenant_id TEXT,
  source_type TEXT NOT NULL CHECK (source_type IN ('webhook', 'outbox', 'planning', 'buffer')),
  source_id TEXT,
  error_message TEXT NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_at TIMESTAMPTZ,
  resolution TEXT
);
```

## Row Level Security

Enable RLS when any admin UI, tenant API, or multi-user access is added.

```sql
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE outbox_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation_signals ENABLE ROW LEVEL SECURITY;

-- Pattern: runtime uses a session variable or JWT claim for tenant_id
CREATE POLICY tenant_isolation ON conversations
  USING (tenant_id = current_setting('app.current_tenant_id', true));

-- Repeat for each table
```

For the managed v1 model where Provider controls all DB access directly (no tenant self-service portal), RLS is Medium priority. It becomes Critical before any multi-tenant admin UI is exposed.

## Social memories (V5)
> New in Runtime Architect v5. Tracks social continuity — what has been established with the end user across sessions.

```sql
CREATE TABLE social_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  introduced_company BOOLEAN NOT NULL DEFAULT false,
  known_customer_name TEXT,
  known_customer_role TEXT,
  known_business_type TEXT,
  relationship_warmth TEXT NOT NULL DEFAULT 'new',
  -- new | acquainted | warm | returning | loyal
  explained_topics JSONB NOT NULL DEFAULT '[]',
  known_customer_intent TEXT,
  known_customer_pain_points JSONB NOT NULL DEFAULT '[]',
  prior_questions JSONB NOT NULL DEFAULT '[]',
  interaction_count INT NOT NULL DEFAULT 0,
  last_session_summary TEXT,
  preferred_language TEXT DEFAULT 'id',
  preferred_formality TEXT DEFAULT 'auto',
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id)
);
```

## Operational memories (V5)
> New in Runtime Architect v5. Tracks business pipeline state — pricing, payment, onboarding progress.

```sql
CREATE TABLE operational_memories (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  lead_stage TEXT NOT NULL DEFAULT 'new',
  lead_temperature TEXT DEFAULT 'unknown',
  selected_package TEXT,
  quoted_price NUMERIC,
  negotiated_price NUMERIC,
  discount_given NUMERIC,
  discount_type TEXT,
  payment_status TEXT DEFAULT 'none',
  invoice_sent BOOLEAN NOT NULL DEFAULT false,
  invoice_sent_at TIMESTAMPTZ,
  payment_confirmed_at TIMESTAMPTZ,
  payment_method TEXT,
  onboarding_status TEXT DEFAULT 'not_started',
  delivery_promised_at TIMESTAMPTZ,
  access_granted BOOLEAN NOT NULL DEFAULT false,
  last_human_decision TEXT,
  last_human_decision_at TIMESTAMPTZ,
  next_expected_action TEXT,
  loss_reason TEXT,
  stage_entered_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, customer_channel_id)
);
```

## Scheduled follow-ups (V5)
> New in Runtime Architect v5. Queued outbound follow-up messages (payment reminders, re-engagement).

```sql
CREATE TABLE scheduled_followups (
  id UUID PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  customer_channel_id TEXT NOT NULL,
  action_type TEXT NOT NULL,
  -- payment_reminder | stall_reengagement | custom
  scheduled_at TIMESTAMPTZ NOT NULL,
  context_hint TEXT,
  max_attempts INT NOT NULL DEFAULT 2,
  attempt_count INT NOT NULL DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'pending',
  last_attempt_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON scheduled_followups (tenant_id, scheduled_at) WHERE status = 'pending';
```

## Conversation tags (V5)
> New in Runtime Architect v5. Behavioral markers that influence AI behavior and enable pipeline views.

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

## Schema checklist

- [ ] Every tenant-scoped table has `tenant_id` column
- [ ] `webhook_events` has UNIQUE constraint on `(provider, instance_id, message_id, event_type)`
- [ ] `runtime_events` is insert-only; no UPDATE on this table
- [ ] `outbox_messages` has `content_hash` for business_outbound matching
- [ ] `message_buffers` has index on `conversation_id WHERE flushed_at IS NULL`
- [ ] `customer_memories` has UNIQUE on `(tenant_id, customer_channel_id, memory_type)`
- [ ] RLS is enabled before any admin UI is shipped
