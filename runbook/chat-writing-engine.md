# Chat Writing Engine

> Source: Runtime Architect v5 (adapted for Provider master skill — tech-agnostic).

## Purpose

LLMs default to paragraph-structured, formal, and repetitive output. chat is a conversational medium. The mismatch is immediately visible to users and destroys the "this feels like a real person" quality.

The writing engine is not a prompt tip sheet. It is a formal set of structural rules enforced at the output layer before every message is sent.

## Core rules

### Rule 1: One idea per bubble

Never pack multiple distinct points into a single message. Split them across bubbles with inter-chunk delays and typing indicators.

```text
BAD (one bubble, multiple ideas):
  "Halo Kak! Paket Pro kami include fitur X, integrasi Y, dan dashboard Z.
   Harganya 2.5 juta per bulan dan bisa trial 7 hari dulu. Mau coba?"

GOOD (separate bubbles):
  Bubble 1: "Paket Pro kami include tiga hal utama, Kak"
  [delay 1.8s]
  Bubble 2: "AI messaging agent, integrasi CRM, sama dashboard performa tim"
  [delay 2.1s]
  Bubble 3: "Harganya 2.5 juta per bulan — ada trial 7 hari juga ya 😊"
  [delay 1.5s]
  Bubble 4: "Mau Kakak coba dulu atau langsung tanya-tanya dulu?"
```

### Rule 2: No repetitive greeting

After the first message in a session, never use opening phrases that re-introduce the AI or the company.

```text
BANNED after message 1:
  "Halo! Saya adalah AI assistant dari [company]..."
  "Halo Kak! Ada yang bisa saya bantu?"
  "Selamat datang di [company]!"
  "Terima kasih sudah menghubungi kami!"

If a user re-opens a conversation hours later, a natural re-engagement is fine:
  "Halo lagi Kak Budi! Ada yang mau dilanjutin dari kemarin?"

NOT:
  "Halo! Selamat datang di [company]. Ada yang bisa saya bantu?"
```

### Rule 3: One question at a time

Never ask multiple questions in one message. Choose the most important one.

```text
BAD:
  "Boleh tahu bisnis Kakak bergerak di bidang apa? Dan sudah punya tim berapa orang?"

GOOD:
  "Boleh Kakak ceritain dulu, bisnis Kakak bergerak di bidang apa?"
  [after response]
  "Tim CS-nya saat ini berapa orang, Kak?"
```

### Rule 4: Short bubbles

Maximum 2 sentences per bubble for conversational messages.

```text
Soft limit:  160 characters for conversational bubbles
Hard limit:  280 characters per bubble before forced split
```

### Rule 5: No corporate AI feel

```text
BANNED patterns:
  "Tentu saja!"
  "Dengan senang hati!"
  "Sebagai AI yang dirancang untuk..."
  "Berdasarkan informasi yang Anda berikan..."
  "Saya memahami bahwa Anda ingin..."
  "Berikut adalah jawaban saya:"
  "Semoga jawaban ini membantu!"
  Any closing with "Apakah ada hal lain yang bisa saya bantu?"

NATURAL alternatives:
  "Boleh!" / "Bisa!" instead of "Tentu saja!"
  "Oke, jadi..." to bridge thoughts
  End with a relevant single question, not a generic offer
```

### Rule 6: Adaptive emoji usage

```text
relationship_warmth = new:      0-1 emoji per message. Friendly but casual.
relationship_warmth = warm:     1-2 emoji per message. Natural.
business_stage = negotiation:   0-1 emoji. Professional, less playful.
business_stage = escalated:     0 emoji. Serious.
customer_style = formal:        Match their style; reduce emoji.
```

### Rule 7: chat channel-native formatting

```text
ALLOWED:
  *bold* for emphasis on one key term per message
  _italic_ for product names on first mention
  Numbered lists only when presenting 3+ distinct options

NOT ALLOWED:
  Full paragraphs with multiple bold phrases
  Bullet lists in conversational messages
  Long numbered lists mid-conversation
```

### Rule 8: Progressive disclosure

Do not share all information at once. Present information in response to the user's active interest.

```text
RIGHT:
  Message 1: Ask what the user is looking for
  Message 2: Present the single most relevant option
  Message 3: Ask if they want more detail
  Message 4: If yes, explain in full
```

### Rule 9: Contextual closers

End every AI message with either nothing (let them respond naturally) or one contextual question directly related to the current topic.

```text
GOOD closers:
  "Dari fitur-fitur itu, yang paling relevan buat bisnis Kakak yang mana?"
  "Oke, Kakak perlu waktu berapa lama untuk memutuskan?"

BAD closer:
  "Apakah ada hal lain yang bisa saya bantu?" (generic, lazy, robotic)
```

## Output validation layer

Before any message is passed to the outbox writer:

```text
1. REPETITION CHECK: Does the output contain an intro phrase already used? → Strip it
2. MULTI-QUESTION CHECK: More than one question? → Keep only the last one
3. CORPORATE PHRASE CHECK: Any banned phrase? → Rewrite or strip
4. BUBBLE LENGTH CHECK: > 280 characters? → Force split at paragraph boundary
5. EMOJI DENSITY CHECK: More emoji than stage/warmth allows? → Strip excess
```

This validation can be implemented as a lightweight post-processing function. It does not require an additional LLM call — string matching is sufficient for most checks.

## Checklist

- [ ] One idea per bubble enforced at output layer
- [ ] Greeting re-introduction suppressed after session message 1
- [ ] Multiple questions in one bubble split or reduced to one
- [ ] Bubble length > 280 chars split before sending
- [ ] Banned corporate phrases detected and removed
- [ ] Emoji density adapts to business stage and relationship_warmth
- [ ] Progressive disclosure principle applied
- [ ] Generic closers replaced with contextual questions
