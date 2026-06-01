# Client Agent Schema Extension

This document extends the core Messaging Agent database schema for provider-managed client agents.

It answers this question:

```text
When an operator spawns a client WA Agent, where does the live agent profile and client knowledge live?
```

Answer:

```text
Supabase is the live source of truth.
VPS workspace is draft/backup/human-readable state.
```

For live runtime, each client agent needs at least:

```text
tenants
tenant_agent_profiles
tenant_knowledge
```

## 1. tenant_agent_profiles

Stores the client agent identity and behavior.

Example:

```text
Tenant: Toko Baju B
Agent: Rani
Role: AI Frontdesk
Tone: ramah, singkat, WhatsApp-native
```

Recommended SQL:

```sql
CREATE TABLE IF NOT EXISTS tenant_agent_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  agent_name TEXT NOT NULL DEFAULT 'Aulia',
  role TEXT NOT NULL DEFAULT 'AI Frontdesk',
  language TEXT NOT NULL DEFAULT 'id',
  tone TEXT,
  greeting_style TEXT,
  bubble_style TEXT,
  fallback_message TEXT,
  system_prompt TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (tenant_id)
);

CREATE INDEX IF NOT EXISTS tenant_agent_profiles_tenant_idx
  ON tenant_agent_profiles (tenant_id);
```

### Field notes

- `agent_name`: public-facing client agent name, e.g. Rani, Sinta, Aulia.
- `role`: what this agent does for the client.
- `language`: default response language.
- `tone`: brand voice summary.
- `greeting_style`: approved opening style.
- `bubble_style`: WhatsApp formatting rules.
- `fallback_message`: safe reply when handoff is needed.
- `system_prompt`: optional generated prompt layer for this tenant.
- `is_active`: allows disabling a profile without deleting it.

## 2. tenant_knowledge

Stores approved business knowledge, FAQ, handoff rules, package scope, and brand voice.

Recommended SQL:

```sql
CREATE TABLE IF NOT EXISTS tenant_knowledge (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  kind TEXT NOT NULL CHECK (
    kind IN (
      'business_profile',
      'faq',
      'pricing',
      'policy',
      'handoff_rule',
      'greeting',
      'brand_voice',
      'package_scope',
      'custom'
    )
  ),
  title TEXT,
  content TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS tenant_knowledge_tenant_kind_idx
  ON tenant_knowledge (tenant_id, kind)
  WHERE is_active = true;
```

### Field notes

- `kind = 'business_profile'`: client business context.
- `kind = 'faq'`: approved FAQ entries.
- `kind = 'pricing'`: approved public pricing or pricing rules.
- `kind = 'policy'`: business rules and limits.
- `kind = 'handoff_rule'`: when AI must pause and route to human/admin.
- `kind = 'greeting'`: approved greeting/opening message.
- `kind = 'brand_voice'`: writing style and tone.
- `kind = 'package_scope'`: Basic/Pro/Custom boundaries.
- `kind = 'custom'`: temporary extension for client-specific data.

## 3. Runtime lookup

When a WhatsApp message arrives:

```text
Incoming webhook
↓
Runtime maps whatsapp_instance to tenant_id
↓
Runtime loads tenants row
↓
Runtime loads active tenant_agent_profiles row
↓
Runtime loads active tenant_knowledge rows
↓
Runtime builds prompt/context
↓
Runtime answers as the client agent
↓
Runtime writes messages under the same tenant_id
```

## 4. Workspace-to-Supabase sync

Workspace files should map to Supabase rows like this:

| Workspace file | Supabase target |
|---|---|
| `tenant-profile.json` | `tenants` |
| `agent-profile.md` | `tenant_agent_profiles` |
| `business-profile.md` | `tenant_knowledge.kind = business_profile` |
| `faq.md` | `tenant_knowledge.kind = faq` |
| `handoff-rules.md` | `tenant_knowledge.kind = handoff_rule` |
| `brand-voice.md` | `tenant_knowledge.kind = brand_voice` |
| `package-scope.md` | `tenant_knowledge.kind = package_scope` |
| `qa-plan.md` | workspace/report only unless QA is later formalized in DB |

Preferred direction:

```text
workspace draft
↓ approval
Supabase live rows
↓ runtime reads
client agent answers
```

## 5. Example: Toko Baju B / Rani

### tenant_agent_profiles

```text
tenant_id: tenant_bbb
agent_name: Rani
role: AI Frontdesk Toko Baju B
language: id
tone: ramah, singkat, WhatsApp-native
fallback_message: Aku teruskan ke admin ya Kak.
is_active: true
```

### tenant_knowledge

```text
tenant_id: tenant_bbb
kind: faq
title: Cara order
content: Customer bisa order dengan mengirim nama produk, ukuran, warna, dan alamat.
is_active: true
```

```text
tenant_id: tenant_bbb
kind: handoff_rule
title: Stok tidak tersedia
content: Kalau stok tidak ada di data, teruskan ke admin.
is_active: true
```

## 6. SaaS note

When the product becomes a self-serve SaaS, add:

```text
profiles
tenant_members
```

Then RLS policies should use `tenant_members` so each logged-in user can only read/write tenant rows they belong to.

Provider admin can be handled by a separate role/policy.

## 7. Hard rules

- Do not store client agent profiles as one Hermes profile per client.
- Do not let runtime answer without resolving tenant_id first.
- Do not mix knowledge between tenants.
- Do not allow frontend service_role keys.
- Do not expose these tables publicly without Auth + RLS.
