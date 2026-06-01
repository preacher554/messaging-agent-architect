# Client Agent Workspace Runbook

This runbook defines where spawned client Messaging/WA Agents live.

The mental model is:

```text
Operator sets up and monitors.
Client agent answers end users.
Runtime executes the client agent using tenant data.
Supabase is the live source of truth.
VPS workspace is the human-readable draft/backup workspace.
```

## 1. Key distinction

Do not create one Hermes profile per client agent.

Use this model instead:

```text
Hermes / VPS
├── internal operator profiles
├── messaging runtime
└── MCP tools

Supabase
├── tenants
├── tenant_agent_profiles
├── tenant_knowledge
├── conversations
└── messages

VPS workspace
└── /root/hermes-workspace/wa-agent/tenants/<tenant_key>/
```

Examples:

```text
Nara / Messaging Operator = internal operator
Rani = client WA Agent for Toko Baju B
Sinta = client WA Agent for Klinik Sehat
```

Rani and Sinta are not separate Hermes profiles. They are tenant-scoped agent profiles and knowledge records.

## 2. Source of truth

### Live production source of truth

Supabase is the live source of truth for active client agents.

The runtime should read live behavior from:

```text
tenants
tenant_agent_profiles
tenant_knowledge
conversations
messages
handoff_events
lead_summaries
```

### Human-readable workspace

The VPS workspace stores draft, backup, and reviewable files:

```text
/root/hermes-workspace/wa-agent/tenants/<tenant_key>/
```

The workspace is useful for:

- operator review,
- draft onboarding,
- human-readable backup,
- QA handover,
- debugging,
- change proposals,
- audit notes.

The runtime may read from workspace in dev mode, but production should prefer Supabase.

## 3. Standard workspace layout

Each client agent workspace must use this layout:

```text
/root/hermes-workspace/wa-agent/tenants/<tenant_key>/
├── tenant-profile.json
├── agent-profile.md
├── business-profile.md
├── faq.md
├── package-scope.md
├── handoff-rules.md
├── brand-voice.md
├── qa-plan.md
├── go-live-checklist.md
└── reports/
    ├── onboarding.md
    ├── qa-result.md
    └── incidents.md
```

Example:

```text
/root/hermes-workspace/wa-agent/tenants/toko_baju_b/
├── tenant-profile.json
├── agent-profile.md
├── faq.md
└── handoff-rules.md
```

## 4. File responsibilities

### tenant-profile.json

Stores the client/tenant identity and runtime mapping.

Example:

```json
{
  "tenant_key": "toko_baju_b",
  "business_name": "Toko Baju B",
  "package": "basic",
  "channel": "whatsapp",
  "whatsapp_instance": "evolution_toko_baju_b",
  "admin_handoff_id": "628xxxxxxxxxx@s.whatsapp.net",
  "ai_enabled": true
}
```

### agent-profile.md

Stores the client agent identity and behavior.

Example:

```md
# Agent Profile

Agent name: Rani
Business: Toko Baju B
Role: AI Frontdesk
Language: id
Tone: ramah, singkat, WhatsApp-native
Main task: jawab FAQ, bantu intake, dan handoff kalau butuh admin.
```

### business-profile.md

Stores business context:

- business category,
- offer,
- target customer,
- location,
- operating hours,
- service area,
- order flow.

### faq.md

Stores approved FAQ only. Do not allow the runtime to invent facts outside this approved knowledge.

### package-scope.md

Defines what the client package allows.

Examples:

- Basic: receptionist + FAQ + simple intake + handoff.
- Pro: lead qualification + CRM summary + richer memory.
- Custom: integrations, catalog, booking, payment, marketplace.

### handoff-rules.md

Defines when AI must pause and hand off to human/admin.

Examples:

- custom pricing,
- refund,
- angry complaint,
- unavailable stock,
- legal/medical/financial decision,
- explicit request for admin.

### brand-voice.md

Defines WhatsApp writing style:

- greeting style,
- emoji density,
- formality,
- banned phrases,
- bubble length,
- one-question-at-a-time rule.

### qa-plan.md

Defines tests before go-live:

- first greeting,
- FAQ answer,
- fragmented message buffering,
- handoff trigger,
- manual human takeover,
- AI resume,
- duplicate webhook,
- tenant isolation.

## 5. Spawn output

When an operator spawns a new client agent, the output must include both workspace files and Supabase rows.

For client `Toko Baju B` with agent `Rani`:

```text
Workspace:
/root/hermes-workspace/wa-agent/tenants/toko_baju_b/

Supabase:
tenants -> tenant_bbb
tenant_agent_profiles -> Rani
tenant_knowledge -> FAQ, handoff rules, business profile, brand voice
```

## 6. Runtime lookup flow

Live chat flow:

```text
Customer chats WhatsApp B
↓
Evolution API webhook reaches VPS
↓
Runtime maps whatsapp_instance to tenant_bbb
↓
Runtime loads tenant_bbb
↓
Runtime loads Rani from tenant_agent_profiles
↓
Runtime loads Toko Baju B knowledge from tenant_knowledge
↓
Runtime answers as Rani
↓
Runtime writes conversation/messages under tenant_bbb
```

The internal operator is not in the live chat loop. The operator sets up, monitors, and fixes.

## 7. Required Supabase tables

Minimum live tables for spawned client agents:

```text
tenants
tenant_agent_profiles
tenant_knowledge
conversations
messages
handoff_events
lead_summaries
```

Future/advanced tables:

```text
outbox_messages
message_buffers
runtime_events
customer_memories
social_memories
operational_memories
conversation_tags
scheduled_followups
```

## 8. Sync direction

Preferred sync direction:

```text
Workspace draft/review
↓ approved
Supabase live rows
↓ runtime reads
Client agent answers
```

Do not treat workspace files as the only production source once the system becomes SaaS.

## 9. Hard rules

- Every client agent must have a tenant.
- Every live record must include tenant scope.
- Do not create a Hermes profile per client agent.
- Do not mix tenant knowledge.
- Do not answer from another tenant's FAQ.
- Do not publish workspace drafts to production without review.
- Do not expose secrets in workspace files.
