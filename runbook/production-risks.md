# Production Risks

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Critical

- AI and human both reply to the same end user (keyboard fighting).
- Duplicate webhook causes duplicate AI replies or duplicate tool execution.
- Webhook blocks on model call and times out, causing provider retries.
- Same-number human replies are ignored because `business_outbound` messages are dropped. This is deployment-blocking for same-channel handoff.
- Retry sends a newly generated answer instead of the original persisted answer.
- Tenant data leaks through global memory, vector search, prompt context, or logs.
- Tool executes payment, discount, booking, invoice, refund, or end user mutation without approval.
- Invalid model string causes every LLM call to fail after deployment.

## High

- No per-conversation locking, allowing concurrent workers to respond.
- No message buffer, causing premature replies to fragmented messaging channel messages.
- No outbox, making send status and retry behavior ambiguous.
- Handoff state can become stale forever.
- Model timeout fallback loses conversation continuity.
- Admin notification is sent but not audited.
- No dead-letter handling for failed webhooks or outbound sends.

## Medium

- Long replies are not chunked for messaging channel.
- AI reply timing is unnaturally fast (bot-feel).
- Emotional state is detected only by prompt, not tracked as runtime signal.
- Business hours and SLA rules are hard-coded.
- No tenant-level budget, rate limit, or model fallback policy.
- Media, captions, voice notes, and buttons are unsupported without explicit fallback.
- No no-UI admin release mechanism for v1 handoff recovery.

## Design principle

Production messaging channel AI is not primarily an intelligence benchmark. It is a distributed communication runtime. Prioritize ordering, ownership, memory continuity, idempotency, policy boundaries, and recovery.

## Humanization risks

- AI replies within 1 second of receiving a message. Users immediately identify this as a bot.
- No typing indicator is shown before message delivery.
- Long replies sent as a single unbroken wall of text.
- Multiple message chunks sent in rapid succession without inter-chunk delay.
- Pacing timer implemented with non-durable timers that don't survive worker restarts.

## Continuity risks

- AI resumes after human takeover with no knowledge of what was discussed or promised.
- Human admin negotiates a price or discount; AI resumes and contradicts it.
- Resume_pending transition goes directly to idle without writing continuity summary.
- Continuity summary is written but not injected into AI context window.

## Emotional and urgency risks

- Emotional state detected by LLM prompt in turn 3 is not available in turn 8.
- No runtime escalation threshold; AI continues trying to qualify an angry end user.
- Urgency not detected; AI buffers an urgent message for the full buffer window.
- No tenant analytics on emotional state; business cannot demonstrate AI value.

## Circuit breaker and LLM failure risks

- No retry backoff: LLM failure causes immediate re-queue, flooding the provider and exhausting the worker pool.
- No circuit breaker: all conversations stall when the LLM provider is down, instead of auto-escalating.
- Retry exhaustion produces silence: end user receives no message and admin receives no alert.
- Outbox retry regenerates from LLM instead of stored content, producing a different reply than the first attempt.
- Webhook handler blocks on LLM call: provider retries the webhook, causing duplicate processing even with idempotency.
- No stale lock recovery: conversation stuck in ai_planning for hours with no end user reply and no admin alert.

## Silent observer risks

- During human_active, AI worker still has write access to outbox_messages and triggers an outbound send.
- AI updates memory during human session with incorrect inferences, corrupting context before continuity summary runs.
- Cooldown state is skipped: AI resumes immediately after continuity summary is written, while admin is still typing a final message.
