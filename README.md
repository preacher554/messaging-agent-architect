# Messaging Agent Architect

Channel-neutral playbook for building managed AI messaging agents: receptionists, sales assistants, support assistants, and handoff-aware frontdesk agents across WhatsApp, Instagram DM, Telegram, webchat, email, and future channels.

This repository is the **source-of-truth playbook** for respawning a dedicated product agent or implementation worker for a messaging-agent product line. It is intentionally not tied to any legacy persona, client name, or one-off runtime.

## What this repo is

- Product and package architecture for managed AI messaging agents.
- Runtime architecture reference for multi-tenant messaging automation.
- Human-AI handoff design with same-channel / same-identity takeover safety.
- Memory, continuity, state-machine, observability, and resilience playbooks.
- Writing engine for natural short-form chat experiences.
- Channel adapter notes, starting with WhatsApp.

## What this repo is not

- Not a production runtime.
- Not a bot token or credential store.
- Not tied to any specific persona name.
- Not tied to a specific messaging channel bridge provider.
- Not a replacement for client-specific SOPs, legal review, or human approval.

## Core layers

1. **Universal runtime layer** — webhook ingestion, idempotency, queues, outbox, state machine, memory, observability, circuit breakers.
2. **Channel adapter layer** — WhatsApp, Telegram, Instagram DM, webchat, email, and other message transports.
3. **Business use-case layer** — receptionist, sales assistant, support assistant, booking, lead qualification, ecommerce/catalog, internal ops.
4. **Tenant layer** — per-client business profile, FAQ, package scope, rules, tone, memory namespace, and handoff contacts.

## Suggested starting points

- `PRODUCT.md` — global product definition.
- `runbook/runtime-architecture-reference.md` — runtime architecture.
- `runbook/handoff-runtime-design.md` — human handoff and ownership model.
- `runbook/database-schema.md` — schema reference.
- `runbook/chat-writing-engine.md` — conversational writing layer.
- `channels/whatsapp-adapter.md` — WhatsApp-specific adapter notes.
- `skills/messaging-agent-core.md` — core operating skill for product agents.

## Safety rule

Every generated worker or product agent that uses this repo must keep client data isolated by tenant and must never expose API keys, tokens, phone numbers, or private customer records in prompts, logs, docs, or commits.
