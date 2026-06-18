---
tags: [fundamentals, semantics]
---

# Delivery Semantics

> **Delivery semantics** is the guarantee about how many times a message gets processed by the consumer: zero-or-one, one-or-more, or exactly one.

## Interview-ready answer

Delivery semantics is the contract between the broker and the consumer about how many times a message will actually be processed. There are three choices: at-most-once (zero or one), at-least-once (one or more), and exactly-once (one and only one). The reason this even matters is that the network is unreliable — acks get lost, consumers crash. So the system has to pick a tradeoff between losing messages and processing duplicates. Most production systems use at-least-once with an idempotent consumer, because true exactly-once is slow, fragile, and rarely worth the cost.

## Why does this exist?

Networks fail. Programs crash. Acks get lost.

Imagine: the consumer pulls a message, processes it (charges a credit card), and **right before** it sends the second ack to the broker, it crashes.

The broker never got the ack.

From the broker's view: *"no ack = work not done = redeliver."*

But the work **was** done. The card was charged. If the broker redelivers, the card gets charged again.

The broker can't tell the difference between:
- Consumer crashed *before* doing the work
- Consumer crashed *after* doing the work but before acking

So the system designer has to **pick a tradeoff**. That choice is the delivery semantic.

## The three options

### 1. At-most-once

**Rule:** The message is delivered zero or one times. Never more.

**How it's built:** Don't retry. Producer sends, broker stores, consumer pulls. If anything goes wrong, the broker shrugs and moves on.

**Tradeoff:** No duplicates, but messages can be lost.

**When to use it:**
- Live telemetry / sensor readings
- Real-time video and audio streams
- Heartbeats and health pings
- High-volume data where the next message replaces the last one

The mental model: data is **continuous, time-sensitive, and self-correcting**. Losing one is fine because another arrives in a second.

### 2. At-least-once

**Rule:** The message is delivered one or more times. Never zero.

**How it's built:** The broker keeps redelivering until it gets the second ack. The crashed-consumer problem from above? The broker just retries. The message only gets deleted after a real ack.

**Tradeoff:** No lost messages, but duplicates are possible.

**When to use it:**
- Payment processing
- Order placement
- Bank transfers
- Email sending
- Most business workflows

The mental model: **at-least-once + idempotent consumer = safe and reliable.** This is what most real production systems actually use.

> [[Idempotency]] is what makes at-least-once safe. Without it, duplicates become real bugs.

### 3. Exactly-once

**Rule (the dream):** The message is processed exactly one time. No losses, no duplicates.

**The catch:** This is *nearly impossible* to achieve at the network level. The "broker can't tell what happened" problem doesn't go away.

**How systems fake it:**

1. **At-least-once + idempotent consumer.** Duplicates do arrive, but the consumer's dedup logic makes them *effectively* exactly-once. Most "exactly-once" systems are really this.

2. **Transactional commits.** The broker and the consumer's database participate in a **shared transaction**. The work and the ack are bound together — either *both* happen or *neither* does.

   ```
   start transaction
     write to DB
     ack the broker
   commit
   ```

   If the consumer crashes mid-way, the transaction rolls back. The DB write is undone, and the broker redelivers — but there's no duplicate, because the DB was rolled back.

**Why not always pick exactly-once?**

- **Slow** — atomic transactions need multiple round-trips (prepare phase, commit phase). Two-phase commit (2PC) adds latency to every single message.
- **Tight coupling** — broker and consumer's database have to speak the same transaction protocol. Most systems don't.
- **Locks and blocking** — transactions hold locks. One slow participant blocks everyone.
- **In-doubt transactions** — if the broker commits but the DB times out, you have stuck transactions that need manual recovery.

## The tradeoff table

| Semantic | Lose messages? | Duplicates? | When to use |
|---|---|---|---|
| At-most-once | Yes | No | Streaming, telemetry, throwaway data |
| At-least-once | No | Yes | Most business work (with [[Idempotency]]) |
| Exactly-once | No | No | Rare; expensive; mostly fake |

## A real example

Order placement system:

- **Auth service → Order service:** at-least-once. Losing an order = lost money. Duplicate orders are caught by an `orderId` check.
- **Order service → Email service:** at-least-once. A duplicate welcome email is fine; a missing one means the user never confirms.
- **App metrics dashboard:** at-most-once. Losing one CPU reading is invisible; a duplicated reading creates a fake spike.

Same system. Different semantics on different paths. The choice depends on **what hurts more — losing or duplicating.**

## Mental model

**Mailing a letter.**

- **At-most-once** is regular post — drop it in the box, hope it arrives, if it doesn't you'll never know. Cheap, fast, occasionally lost.
- **At-least-once** is registered mail with retry — the post office keeps trying, you might get the letter delivered twice if a previous "failed" attempt actually went through. You'll definitely get it; you might also get a duplicate.
- **Exactly-once** is courier with a notarized chain of custody. It's possible, but it's slow, expensive, and the steps to make it bulletproof are why most people just use registered mail and tell the recipient to ignore duplicates.

## Common misconceptions

- **"Exactly-once delivery is what you want by default."** No. It is rarely worth the cost. Most production systems use at-least-once with [[Idempotency|idempotent consumers]] and call that "effectively exactly-once."
- **"At-most-once means messages are unsafe."** Only if losing one matters. For sensor readings or live telemetry, at-most-once is exactly the right choice — losing one reading is invisible, and a duplicate would create a fake spike.
- **"Service Bus gives you exactly-once if you ask for it."** Service Bus gives you at-least-once with strong dedup tools (duplicate detection, sessions, transactions). True end-to-end exactly-once still requires the consumer to be idempotent — the broker alone cannot guarantee it.
- **"The choice is per-system."** No. It's **per message path**. The same system can use at-most-once for metrics and at-least-once for orders. Pick by what hurts more: losing or duplicating.

## See also

- [[Producer]]
- [[Broker]]
- [[Consumer]]
- [[Idempotency]]

## Index

[[Messaging Fundamentals]]
