# Human Handoff SOP (v2)

## Goal

Ensure the AI stops and escalates when human authority, missing information, or custom judgment is required — and resumes intelligently after human takeover.

## Handoff triggers

Always handoff when the end user asks for:

- final custom price,
- discount approval,
- refund/cancellation,
- complaint/escalation,
- legal/medical/financial decision,
- partnership/custom deal,
- unavailable/missing business info,
- angry messages,
- integration/custom workflow,
- payment issue.

## Handoff keyword detection

- Use **word boundary matching** to prevent false positives.
- Keyword `custom` must NOT match inside `client`, `customize`, or similar.
- Compile keyword list per tenant from `handoff-rules.md`.

## Basic package handoff

Basic should handoff early when conversation moves beyond FAQ or simple booking/order intake.

## Pro package handoff

Pro may qualify the lead first, then handoff with summary when:

- lead is hot,
- end user requests next step,
- custom pricing is needed,
- end user asks for human confirmation,
- AI confidence is low.

## Primary handoff model (v2)

```txt
end user chats business messaging channel number
↓
AI detects handoff trigger
↓
AI tells end user it will forward to admin
↓
AI sends handoff reply via outbox → state: waiting_human → owner: human
↓
System sends handoff notification to admin's private messaging channel number
↓
Admin replies from the same business/number
↓
business_outbound: true → runtime classifies as human admin message
↓
Runtime persists, transitions → human_active
↓
last_human_activity_at = now()
↓
end user inbounds within 1 hour → stored, no AI reply
↓
end user inbound after 1 hour → ai_active, context-preserving resume
↓
any new business_outbound:true resets the 1-hour timer
```

## Handoff message to end user

```txt
Baik Kak, untuk bagian itu aku bantu teruskan ke admin ya supaya jawabannya lebih tepat. Mohon tunggu sebentar.
```

Include business hours if relevant. Don't ask when the user is free.

## Admin notification message

```txt
[Messaging Agent Handoff]
Tenant: {{tenant_id}}
Client: {{client_name}} / {{client_phone}}
Reason: {{escalation_reason}}
Lead status: {{lead_status}}
Last message: {{last_message}}

Recommended next action:
{{recommended_next_action}}

AI sudah pause untuk chat ini. Silakan balas dari nomor WA bisnis/agent yang sama.
```

## Conversation states + ownership (v2)

**Lifecycle states:**
```txt
ai_active       = AI may respond normally
waiting_human   = escalation sent; AI must not answer except waiting message
human_active    = admin replying; AI stays silent, 1-hour window active
resolved        = issue closed; AI may resume on next inbound
```

**Ownership:**
```txt
owner: ai       = AI is allowed to act
owner: human    = Human has taken over; AI hard-locked out
owner: system   = System operations only
```

When `owner: human`, NO model call is made. Hard boundary regardless of inbound volume.

## 1-hour human window — critical rules

1. **No proactive send on timer expiry.** The 1-hour window only enables AI reply on the next end user inbound.
2. **Timer resets on human activity.** Any new `business_outbound: true` resets the clock.
3. **End user messages during the window are stored** for context continuity.
4. **Resume is context-preserving** — the next end user inbound after the window transitions to ai_active with resume note injected.

## Resume prompt engineering (v2)

1. Label human admin messages as `Admin [TenantName]`, NOT as the AI agent.
2. Inject operational note:
   ```
   Percakapan ini baru di-resume otomatis setelah admin mengambil alih.
   Balas sebagai Messaging Agent yang aktif kembali. Jangan ulang dari awal; lanjutkan natural.
   ```
3. Include full conversation window including human admin messages.
4. Never ask name/business again if history is non-empty.

## Outbox echo handling

AI's own outbound messages arrive as `business_outbound: true` when sent-message events are delivered. The runtime distinguishes:

```txt
business_outbound: true
  ↓
Matches a pending outbox row?
  ├─ Yes → Mark as sent (echo) → no state change
  └─ No → Human admin message → persist + state transition
```

## QA checklist

- [ ] Handoff keyword `custom` does NOT match `client`
- [ ] AI stops replying after handoff (state: waiting_human)
- [ ] Admin notification sent to private WA
- [ ] business_outbound: true correctly classified (echo vs human)
- [ ] 1-hour window: end user messages stored, no AI reply
- [ ] Timer expiry: NO proactive send
- [ ] Resume after timer: contextual, not a restart
- [ ] New business_outbound: true resets timer
- [ ] Human outbound messages labeled as "Admin" in model history
- [ ] Outbox echo does not trigger human_active transition
