# Message Buffer Design

> Source: Runtime Architect v4 (adapted for Provider master skill — tech-agnostic).

## Why this matters

Indonesian messaging channel users frequently send messages in rapid fragments before finishing their thought. The runtime must buffer these fragments, wait for a natural pause, then merge them into one coherent turn before AI planning begins.

Without buffering:
- User sends "Kak" → AI plans and replies to a single-word greeting
- User sends "mau tanya soal paket" → AI plans and replies again
- User sends "yang pro itu gimana" → AI plans and replies a third time
- end user receives 3 fragmented AI replies to one incomplete question

With buffering:
- All three fragments are held in the buffer
- After the buffer window expires, they are merged into: "Kak mau tanya soal paket yang pro itu gimana"
- AI plans once, replies once, response is coherent

## Buffer window algorithm

```text
on message.received for conversation_id:

  1. Check for active buffer row where flushed_at IS NULL

  2a. If NO active buffer:
      INSERT buffer row:
        conversation_id, tenant_id
        messages = [current_message]
        flush_at = now() + buffer_window_ms
      Schedule delayed flush job
      Emit: message.buffered
      Return

  2b. If ACTIVE buffer exists:
      APPEND current_message to buffer.messages
      UPDATE buffer.flush_at = now() + buffer_window_ms
      Cancel existing delayed job
      Re-schedule with new delay
      Emit: message.buffered
      Return

3. When flush job fires (no new message arrived):
   UPDATE buffer SET flushed_at = now()
   Merge messages into one turn (see merge rules)
   Emit: buffer.flushed
   Enqueue conversation for AI planning
```

## Timing values

| Parameter | Value | Notes |
|---|---|---|
| `buffer_window_ms` | 1500 | Reset on each new message |
| `max_wait_ms` | 8000 | Hard cap; flush even if messages keep arriving |
| `urgent_flush_ms` | 0 | Flush immediately on urgency signal |

The `max_wait_ms` cap prevents a chatty user from blocking the AI indefinitely.

## Fragment merge rules

```text
1. Sort messages by received_at ascending
2. Join with single space: " ".join(messages)
3. Preserve original capitalization and punctuation
4. Do not add conjunctions or filler words
5. Treat the merged result as a single inbound turn

Example:
  ["Kak", "mau tanya", "soal paket pro"] → "Kak mau tanya soal paket pro"
```

## Flush override conditions

Flush the buffer immediately and skip the remaining wait when:
- The buffered content contains a handoff trigger keyword (darurat, bayar, komplain, angry patterns)
- The buffer has exceeded `max_wait_ms`
- The conversation transitions to handoff from another event

## State machine integration

```text
conversation.state = 'user_buffering' while buffer window is open
conversation.state = 'ai_planning'   when buffer.flushed fires
```

Do not allow AI planning to start while state is `user_buffering`. If a new message arrives during `ai_planning`, it creates a new buffer for the next turn rather than interrupting the current one.

## Durability requirements

- Buffer state must be persisted (database), not in-memory only
- Flush scheduling must use a durable delayed job, NOT an in-memory timer
- In-memory timers (setTimeout) do not survive worker restarts, causing buffer leaks
- If using a queue system: stable job ID per conversation (`buffer_flush:{conversation_id}`) enables cancel and re-schedule on timer reset

## Checklist

- [ ] Timer resets on every new message within the window
- [ ] Hard cap prevents indefinite blocking
- [ ] Stable job ID prevents duplicate flush jobs
- [ ] Urgency keywords trigger immediate flush
- [ ] Merged content is stored in the buffer row for audit
- [ ] State transitions to user_buffering when buffer opens
- [ ] State transitions to ai_planning only after buffer flushed
- [ ] Buffer survives worker restarts (persistent storage, not in-memory)
