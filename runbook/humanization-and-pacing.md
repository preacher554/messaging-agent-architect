# Humanization and Pacing

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Purpose

messaging channel users notice inhuman timing immediately. A reply arriving within one second marks the interaction as a bot and reduces trust and compliance, regardless of how good the content is. This reference defines the layers required to make AI responses feel like they came from a real person.

Every Messaging Agent runtime — Basic, Pro, or Custom — SHOULD implement Layer 1 and Layer 2. Layer 3+ are Pro/Custom tier differentiators.

## Layer 1 — Response delay calculation

Calculate a pacing delay before sending every outbound message. Do not use a fixed constant.

### Mathematical formula

```
T_delay = T_base + (N_char / WPM_char) × 60

Where:
  T_delay   = total delay in seconds before message is sent
  T_base    = perceived read time for the incoming message (seconds)
  N_char    = character count of the AI response being sent
  WPM_char  = simulated human typing speed = 250 characters per minute

Example:
  Response of 180 characters, T_base = 2.0s:
  T_delay = 2.0 + (180 / 250) × 60 = 2.0 + 43.2 = 45.2s → capped at 8s
  T_delay = min(T_delay_calculated, T_max) = 8s
```

Cap: `T_max = 8.0 seconds`. Never delay longer than this regardless of response length.

### T_base: sentiment-aware read time

`T_base` is based on the complexity and sentiment of the **incoming** end user message. The AI simulates "thinking about what was just said."

```text
incoming message is simple greeting only (Halo, Hi, Selamat pagi):
  T_base = 1.0s

incoming message is normal inquiry:
  T_base = 1.5s (default)

incoming message contains complex multi-part question:
  T_base = 2.0s

incoming message contains urgency signal (darurat, segera, emergency, buru):
  T_base = 1.0s  // read fast, respond faster

incoming message contains angry/frustrated signal:
  T_base = 2.5s  // take a beat; rushing feels dismissive
```

## Layer 2 — Typing indicator (presence simulation)

The messaging channel bridge API supports composing/paused presence. Use this before every outbound message.

```text
1. AI decision ready, outbox row created
2. Outbox worker starts:
   a. call sendPresence('composing', customer_channel_id)
   b. wait calculated_delay milliseconds
   c. call sendText(content, customer_channel_id)
   d. call sendPresence('paused', customer_channel_id)
   e. update outbox status = sent
```

Do not call `sendPresence` after message delivery. The paused state is set before moving to the next action.

## Layer 3 — Long message chunking

messaging channel is a messaging app, not an email client. Long walls of text reduce engagement and feel robotic.

```text
chunk_threshold_chars = 280

if response_length <= chunk_threshold_chars:
  send as single message

if response_length > chunk_threshold_chars:
  split at natural paragraph breaks, not mid-sentence
  max 3 chunks per response
  for each chunk:
    send typing indicator
    wait 1200–1800ms
    send chunk
    pause presence
```

Splitting rules:
- Split at double newline first
- If no double newline exists, split at sentence boundary near the midpoint
- Never split mid-word or mid-sentence
- Never produce a chunk shorter than 40 characters unless it is the final one

## Layer 4 — Read receipt awareness

Optionally use read receipt to reduce delay for already-open conversations, since the end user is actively waiting.

```text
if customer_has_open_chat:
  calculated_delay *= 0.8
```

This is optional. Do not implement it before Layer 1 and Layer 2 are stable.

## Layer 5 — Message burst protection

Do not send multiple AI messages in rapid succession even when chunking. Rapid delivery looks like a spammer.

```text
minimum_inter_message_gap_ms = 1000

Always enforce this gap between consecutive outbound messages to the same end user,
regardless of whether they are chunks of one response or separate planned replies.
```

## Implementation notes

- Use a durable delayed job for the pacing wait, not an in-memory timer. Timers that don't survive worker restarts cause messages to be lost or sent too fast.
- Humanization applies to all outbound messages, not only AI replies (system messages, notifications, handoff confirmations).

## Checklist

- [ ] Response delay is calculated from message length, not a constant
- [ ] Urgency override reduces delay
- [ ] Typing indicator is set before send, released after
- [ ] Long messages are chunked at paragraph boundaries
- [ ] Inter-chunk delay uses typing indicator
- [ ] Pacing is implemented as a durable delayed job, not ephemeral timer
- [ ] Minimum inter-message gap is enforced
- [ ] Humanization applies to all outbound messages
