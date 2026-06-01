# WhatsApp Channel Adapter

## Purpose

WhatsApp is a priority channel for managed messaging agents because many businesses already use it as their primary customer inbox.

This adapter defines WhatsApp-specific concerns while keeping the core architecture channel-neutral.

## Adapter responsibilities

- Receive WhatsApp message webhooks from the selected bridge provider.
- Normalize provider payloads into internal `message_events`.
- Preserve provider message IDs for idempotency.
- Detect customer inbound vs business-number outbound events.
- Support admin takeover when the human replies from the same business number.
- Send text/media replies through the bridge using an outbox worker.
- Avoid provider-specific assumptions in the core runtime.

## Provider abstraction

The runtime should depend on an interface, not a specific bridge implementation:

```text
WhatsAppBridge.send_text(instance_id, customer_channel_id, text)
WhatsAppBridge.mark_read(instance_id, message_id)
WhatsAppBridge.set_typing(instance_id, customer_channel_id, enabled)
WhatsAppBridge.normalize_webhook(payload) -> MessageEvent
```

A concrete adapter may use Evolution API, WhatsApp Cloud API, Baileys-based bridges, or another provider, but provider details must stay inside the adapter layer.

## Same-number handoff

If AI and human admin share the business number, outbound events from the business number are operationally meaningful.

Do not globally drop `fromMe: true` / business-outbound events.

Classifier:

```text
business outbound event
  ↓
matches pending/sent outbox row?
  ├─ yes → AI echo / delivery confirmation
  └─ no  → human admin takeover
```

When human takeover is detected:

- store the message as `source = human_admin`,
- transition conversation owner to `human`,
- start or extend the human window,
- prevent AI from sending until release/timeout policy allows.

## WhatsApp UX rules

- Short bubbles.
- One question at a time.
- Typing/read simulation where supported.
- Chunk long messages with delays.
- Avoid long markdown-heavy output.
- No spammy follow-up without explicit tenant policy.

## Required tenant configuration

- WhatsApp instance/channel ID.
- Business display name.
- Admin handoff destination.
- Package: Basic, Pro, or Custom.
- Business hours and SLA policy.
- Approved FAQ/knowledge.
- Escalation rules.
- Follow-up policy.
