# Tenant Configuration

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## Philosophy

Tenant configuration separates tenant-specific runtime behavior from hardcoded defaults. Every value has a safe default so tenants without configuration still run correctly.

Configuration is loaded once at session start and injected into the AI context and all worker decision points. Never fetched mid-conversation — stale reads during a session are safer than inconsistent mid-conversation behavior changes.

## Full tenant.yaml schema

```yaml
# IDENTITY
tenant_id: tenant_xyz
business_name: "Toko Sample"
package: pro                              # basic | pro | custom
channel_instance: instance_001
admin_handoff_id: "6281234567890@channel-provider"

# LANGUAGE AND TONE
language: id                              # id | en | mix
formality: casual                         # formal | casual | auto
emoji_level: medium                       # low | medium | high
brand_voice: |
  Ramah, jujur, tidak bertele-tele.
  Gunakan bahasa sehari-hari yang nyaman.

# RESPONSE BEHAVIOR
response_delay_mode: adaptive             # adaptive | fixed | instant
fixed_delay_ms: 2000                      # only if response_delay_mode = fixed
buffer_window_ms: 1500
max_buffer_wait_ms: 8000
typing_indicator_enabled: true

# SALES BEHAVIOR
upsell_mode: gentle                       # disabled | gentle | aggressive
upsell_after_days: 30
max_discount_percent: 10                  # max AI can offer without human approval
max_discount_absolute: 250000             # absolute cap in local currency
auto_send_invoice: false
price_negotiation_floor_percent: 80       # % of base price

# BUSINESS HOURS
business_hours_enabled: false
business_hours:
  timezone: Asia/Jakarta
  monday:    { open: "08:00", close: "21:00" }
  tuesday:   { open: "08:00", close: "21:00" }
  wednesday: { open: "08:00", close: "21:00" }
  thursday:  { open: "08:00", close: "21:00" }
  friday:    { open: "08:00", close: "21:00" }
  saturday:  { open: "09:00", close: "18:00" }
  sunday:    closed

outside_hours_behavior: inform            # inform | queue | handoff
outside_hours_message: |
  Halo Kak! Saat ini kami sedang tutup.
  Kami buka lagi jam 08.00 WIB ya.
  Boleh tinggalkan pesan, kami balas segera besok 🙏

# HUMAN TAKEOVER
human_takeover_policy: same_number       # same_number | manual
inactivity_timeout_minutes: 30           # default 30, set 60 for stricter
resume_policy: timeout                   # timeout | manual | both
# manual_resume_command: "RESUME {conversation_id}" sent from admin_handoff_id

# FOLLOW-UP POLICY
followup_enabled: true
followup_payment_hours: 6
followup_payment_max_attempts: 2
followup_stalled_days: 3
followup_stalled_message: |
  Halo Kak [name]! Masih tertarik untuk lanjut?

# ESCALATION POLICY
escalation_auto_enabled: true
escalation_emotional_threshold: 2        # nth angry signal before auto-handoff
escalation_message: |
  Maaf Kak, biar lebih cepat terselesaikan, saya sambungkan ke tim kami ya 🙏

# MEMORY POLICY
memory_retention_days: 180
social_memory_enabled: true
operational_memory_enabled: true

# MODULES ENABLED (pro/custom features)
modules:
  lead_qualification: true
  followup_scheduling: true
  invoice_reminder: true
  onboarding_checklist: false
  payment_confirmation: false
  crm_sync: false

# AI BEHAVIOR LIMITS
scope_topics: []
out_of_scope_response: |
  Maaf Kak, untuk pertanyaan itu saya belum bisa bantu.
  Ada yang berkaitan dengan topik yang bisa saya bantu?
max_context_turns: 30
model: gpt-4o-mini                       # validated at deploy time
```

## Config loading rules

```text
1. Load at session start, cache for session duration
2. Do NOT reload mid-conversation
3. If config is missing or invalid: fail safe to package defaults
4. Always validate model string at worker startup
```

## Package-gated features

| Feature | Basic | Pro/Custom |
|---|---|---|
| Human takeover | ✅ | ✅ |
| Social memory | ✅ | ✅ |
| Business hours handling | ✅ | ✅ |
| AI response | ✅ | ✅ |
| Lead qualification | ❌ | ✅ |
| Follow-up scheduling | ❌ | ✅ |
| Invoice reminder | ❌ | ✅ |
| Auto escalation | ❌ | ✅ |
| Upsell mode | ❌ | ✅ |
| Negotiation (discount) | ❌ | ✅ |
| Onboarding checklist | ❌ | ✅ |

## Checklist

- [ ] Config loaded once at session start
- [ ] Config never fetched mid-conversation
- [ ] Model string validated at startup
- [ ] All workers read buffer/timeout from config, not hardcoded
- [ ] Package-gated features checked before enabling
- [ ] Config fallback to package defaults when keys missing
- [ ] Admin can update config; new value takes effect at next session
