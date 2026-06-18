---
tags: [fundamentals, semantics]
---

# Idempotency

> An operation is **idempotent** if applying it once or many times gives the same result. In messaging, it's what makes [[Delivery Semantics|at-least-once]] safe.

## Interview-ready answer

Idempotency means an operation can be safely run multiple times without changing the result beyond the first run. It is the partner of at-least-once delivery — the broker may redeliver the same message many times, so the consumer must handle duplicates without doing the work twice. The two ways to build it are: design naturally idempotent operations (e.g., "set status to SHIPPED" instead of "advance status"), or track a message/order ID and skip if already processed. The dedup check and the work must happen in one atomic transaction, otherwise a crash between them brings the duplicate problem right back.

## Why does this exist?

Remember the at-least-once rule from [[Delivery Semantics]] — the broker keeps redelivering until it gets the second ack. That means the **same message can arrive multiple times.**

If the consumer just blindly does the work each time:

- `ChargeOrder { orderId: 456, amount: $50 }` arrives twice
- Card gets charged $100 instead of $50
- Customer is unhappy

Idempotency is the fix. It lets the consumer say *"I've already processed this — ignore the duplicate"* without breaking anything.

## The word itself

Idempotent comes from math. **Apply it once or many times → same result.**

Math examples that **are** idempotent:

- `abs(-7) = 7`, `abs(abs(-7)) = 7`, `abs(abs(abs(-7))) = 7`
- `{1,2,3} ∪ {1,2,3} = {1,2,3}` (set union with itself)
- `x = 5` — running it again keeps `x` at 5

Math examples that are **not** idempotent:

- `count++` — each run increases the value
- `balance + 50` — keeps growing

The pattern: idempotent operations have a **fixed point**. Once you've reached the answer, applying again keeps you there. Non-idempotent operations **accumulate**.

## Why it pairs with at-least-once

| Semantic | Failure mode | Fix |
|---|---|---|
| At-most-once | Loses messages | Retry on producer side, or accept the loss |
| At-least-once | Delivers duplicates | **Idempotency** |
| Exactly-once | (Mostly fake) | Built on top of at-least-once + idempotency anyway |

Match the safety mechanism to the failure mode. At-least-once fails by duplicating, so the consumer has to absorb duplicates. That's idempotency.

At-most-once doesn't need it — the broker never retries, so there's nothing to deduplicate.

## Two ways to build it

### 1. Naturally idempotent operations

Design the operation so doing it twice has **no effect**.

| Not idempotent | Idempotent version |
|---|---|
| "Add $50 to balance" | "Set balance to $100" |
| "Advance order to next stage" | "Set status to SHIPPED" |
| "Increment retry count" | "Set retry count to 3" |

Replace **deltas** with **absolute values**. Replace **steps** with **target states**.

### 2. Dedup with a key

Track which message IDs (or order IDs) have already been processed.

```python
def handle_charge(msg):
    if db.has_processed(msg.orderId):
        return  # already done, skip
    
    charge_card(msg.orderId, msg.amount)
    db.mark_processed(msg.orderId)
```

If the broker redelivers:
- First time: not in DB → charge → mark
- Second time: already in DB → skip

Card charged once.

## The hidden race condition

There's a subtle bug in the simple version above.

What if the consumer crashes **between** `charge_card()` and `mark_processed()`?

- Card is charged
- DB doesn't know
- Broker redelivers
- Consumer checks DB → "not processed" → charges *again*

Same duplicate problem, just moved one step later.

**The fix:** put the check, the work, and the mark in **one atomic transaction.**

```python
def handle_charge(msg):
    with db.transaction():
        if db.has_processed(msg.orderId):
            return
        charge_card(msg.orderId, msg.amount)
        db.mark_processed(msg.orderId)
```

If the consumer crashes mid-way, the transaction rolls back — *both* the charge and the mark are undone. The broker redelivers safely.

This is the same atomicity idea from [[Delivery Semantics|exactly-once via transactional commits]].

## What about external systems?

If the work touches an external system you can't roll back — e.g., charging a real credit card via Stripe — the database transaction won't help. Stripe doesn't know about your DB.

The fix: **push the dedup down to the external system.**

Most payment APIs support an **idempotency key** — a unique ID you send with the request. Stripe stores it. If you send the same request twice with the same key, Stripe refuses the second one.

```python
stripe.Charge.create(
    amount=5000,
    currency="usd",
    idempotency_key=msg.orderId,  # Stripe dedups on this
)
```

The consumer becomes idempotent because the **external service** is now idempotent.

## A real example

Order placement:

1. Producer sends `ChargeOrder { orderId: 456, amount: $50 }`.
2. Broker stores it.
3. Consumer pulls it. Inside a transaction:
    - Check: has `orderId: 456` been processed? No.
    - Call Stripe with `idempotency_key=456`. Stripe charges the card.
    - Mark `orderId: 456` as processed in DB.
    - Commit transaction.
    - Send second ack to broker.
4. Broker deletes the message.

If anything fails in step 3 — DB rolls back, broker redelivers. On the retry:
- Either the DB check catches it (already processed), OR
- Stripe's idempotency key catches it (already charged)

Charged exactly once. Effectively exactly-once delivery, built from at-least-once + idempotency.

## Mental model

**A light switch.**

Flip the switch up — light turns on. Flip it up again — still on. Flip it up ten more times — still on. The state is *"light is on,"* and re-applying the same action doesn't change it further. That's idempotency. Compare with a counter that goes up by one each press — same input, different result every time. Messaging operations should be light switches, not counters.

## Common misconceptions

- **"Idempotency means the consumer never gets duplicates."** No. The consumer absolutely gets duplicates. Idempotency means duplicates **don't cause harm**.
- **"If my code is idempotent, I don't need transactions."** Watch out — without atomicity between the dedup check and the work, a crash in between brings the duplicate problem back. The check, the work, and the mark must commit together.
- **"GET is idempotent, POST is not."** A useful HTTP rule of thumb, but it's about the verb, not your business logic. A POST that creates a user with a unique email is idempotent in effect (second one fails). A "set status to SHIPPED" PUT is idempotent. Look at what the operation **does**, not what verb it uses.
- **"Idempotency keys are only for payments."** No. Any external operation with side effects benefits from one — sending email, calling a webhook, creating a user. Anywhere you can't roll back, push the dedup down to the external service.

## See also

- [[Delivery Semantics]]
- [[Producer]]
- [[Broker]]
- [[Consumer]]

## Index

[[Messaging Fundamentals]]
