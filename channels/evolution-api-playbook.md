# Evolution API Runtime Playbook

This playbook explains how Evolution API fits into the Messaging/WA Agent provider architecture.

## 1. Simple mental model

```text
Evolution API = WhatsApp bridge
WA Runtime = AI agent engine
Supabase = tenant/profile/knowledge/message database
Port = runtime server door
Instance = WhatsApp number/session
API key = access key to Evolution API server
```

## 2. Message flow

```text
Customer chats a client WhatsApp number
↓
Evolution API receives the WhatsApp event
↓
Evolution API sends webhook to WA Runtime
↓
WA Runtime maps instance to tenant_id
↓
WA Runtime loads tenant agent profile and knowledge from Supabase
↓
AI answers as the client agent
↓
WA Runtime sends reply through Evolution API
↓
Reply reaches customer WhatsApp
```

Example:

```text
Customer chats Toko Baju B
↓
Evolution instance: evolution_toko_baju_b
↓
Runtime maps evolution_toko_baju_b -> tenant_bbb
↓
Runtime loads agent profile: Rani
↓
Runtime loads Toko Baju B FAQ
↓
Runtime answers as Rani
```

## 3. Runtime port

A port is a server door.

Example:

```text
WA Runtime: port 3300
Evolution API: port 8080
Hermes Gateway: port 8642
Dashboard dev server: port 5173 or 3000
```

For provider architecture, do not create one runtime port per client by default.

Preferred model:

```text
WA Runtime port 3300
├── tenant A / NusaAI
├── tenant B / Toko Baju
├── tenant C / Klinik
└── tenant D / Bengkel
```

A single runtime endpoint can serve many tenants because the runtime maps each incoming webhook using instance/channel identity.

## 4. Evolution API instance

One WhatsApp number/session should be represented by one Evolution API instance.

Examples:

```text
instance_nusaai      -> NusaAI WhatsApp number
instance_toko_baju   -> Toko Baju B WhatsApp number
instance_klinik      -> Klinik Sehat WhatsApp number
```

The tenant table must store the mapping:

```text
tenants.whatsapp_instance = instance_toko_baju
```

When a webhook arrives, runtime resolves:

```text
payload.instance_id
↓
tenants.whatsapp_instance
↓
tenant_id
```

## 5. Evolution API key

The Evolution API key is the access credential for the Evolution API server.

It is not the same thing as a runtime port.
It is not necessarily one key per WhatsApp number.

Common self-hosted provider model:

```text
1 Evolution API server
1 API key
many WhatsApp instances/numbers
```

The API key should be stored only in backend/runtime environment variables.

Never store Evolution API keys in:

- frontend code,
- public GitHub files,
- tenant workspace files,
- client-visible dashboard state.

## 6. Multi-tenant mapping

Required Supabase tenant fields:

```text
tenant_id
tenant_key
business_name
package
whatsapp_instance
admin_handoff_id
ai_enabled
```

Runtime lookup:

```text
incoming instance_id
↓
select tenant where whatsapp_instance = instance_id
↓
load tenant_agent_profiles
↓
load tenant_knowledge
↓
process message under tenant_id
```

Every conversation, message, handoff, memory, and runtime event must be tenant-scoped.

## 7. One runtime, many agents

Client agents such as Rani, Sinta, or Aulia are not separate runtime ports.

They are tenant-scoped profiles stored in Supabase:

```text
tenant_agent_profiles
```

The same runtime can run many client agents by loading the correct tenant profile and knowledge.

## 8. When to add another runtime port

Do not add a new port just because a new client is added.

Add another runtime/port only for reasons such as:

- staging environment,
- experimental runtime version,
- enterprise client isolation,
- heavy traffic isolation,
- special network/auth requirement,
- dedicated model/provider environment,
- high-SLA dedicated deployment.

Example:

```text
3300 = shared production runtime
3301 = staging runtime
3310 = dedicated enterprise runtime
```

Do not use this pattern for normal clients:

```text
client A = 3301
client B = 3302
client C = 3303
```

That becomes hard to maintain.

## 9. Model/provider routing

Different AI models do not require different ports by default.

Preferred approach:

```text
One runtime port 3300
├── tenant A -> GPT-5.2
├── tenant B -> Owl Alpha
├── tenant C -> Gemini
└── tenant D -> fallback model
```

Model routing should be stored as tenant configuration, for example:

```text
model_provider
model_name
fallback_provider
fallback_model
model_timeout
```

Add a dedicated runtime only when the model/provider requires special isolation, API key ownership, network settings, or SLA.

## 10. Webhook endpoint standard

Recommended runtime webhook endpoint:

```text
POST /webhook/evolution
```

Provider/public example:

```text
https://api.example.com/webhook/evolution
```

Local/VPS example:

```text
http://127.0.0.1:3300/webhook/evolution
```

The webhook handler should be fast:

```text
parse payload
validate shape
resolve tenant
idempotency check
enqueue work
return 200
```

Do not do heavy LLM work directly inside webhook request handling.

## 11. Required webhook safeguards

Runtime must support:

- idempotency using provider message ID,
- tenant resolution using instance ID,
- inbound vs business outbound detection,
- safe handling of `fromMe` / business outbound events,
- message buffering for fragmented chats,
- outbox-based sending,
- retry/dead-letter for failed sends,
- runtime event logging.

## 12. Business outbound / same-number handoff

If AI and admin use the same business WhatsApp number, outbound events are meaningful.

Do not globally drop `fromMe: true` events.

Classify them:

```text
business outbound event
↓
Does it match an outbox message?
├── yes -> AI echo / delivery confirmation
└── no  -> human admin message / takeover
```

When human admin takeover is detected:

```text
store message as human_admin
set conversation owner = human
set state = human_active
set last_human_activity_at = now
prevent AI generation until release/manual toggle/timeout policy allows
```

## 13. Setup checklist for a new client WhatsApp number

```text
1. Create tenant row.
2. Choose tenant_key.
3. Create tenant agent profile.
4. Add tenant knowledge/FAQ/handoff rules.
5. Create or pair Evolution API instance.
6. Store instance name in tenants.whatsapp_instance.
7. Configure Evolution webhook to runtime endpoint.
8. Send test inbound message.
9. Confirm runtime resolves correct tenant.
10. Confirm response uses correct agent profile.
11. Confirm messages are stored under correct tenant_id.
12. Confirm handoff works.
13. Confirm manual Human Agent toggle pauses/resumes AI.
14. Activate tenant.
```

## 14. Scaling notes

One port can serve many tenants, but capacity depends on:

- runtime worker count,
- queue/backpressure,
- Supabase query/index performance,
- Evolution API capacity,
- number of connected WhatsApp instances,
- model latency and rate limits,
- server CPU/RAM.

When scale grows:

```text
public endpoint
↓
load balancer / reverse proxy
↓
multiple runtime instances
↓
queue
↓
workers
```

From the outside it can still look like one webhook endpoint.

## 15. Hard rules

- Do not create one port per normal client.
- Do not expose Evolution API key in frontend.
- Do not store Evolution API key in tenant workspace files.
- Always map instance to tenant before processing.
- Always store messages under tenant_id.
- Do not let AI answer if tenant AI is disabled.
- Do not let AI answer if conversation owner is human.
- Do not globally ignore business outbound events.
