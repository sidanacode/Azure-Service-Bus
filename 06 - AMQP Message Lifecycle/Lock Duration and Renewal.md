---
tags:
  - amqp
  - servicebus
  - lifecycle
---

# Lock Duration and Renewal

> **`LockDuration` is set on the queue (default 30s, max 5min); the consumer doesn't get to pick.** When a handler legitimately needs more time, the consumer sends `renew_message_lock` calls — and these are not "extensions" of a budget, they're **heartbeats the broker observes in real time**. The moment renewals stop arriving, the broker reclaims the message. This shape — bounded base + active renewal — is why a frozen or sync-blocked consumer can't hold a message hostage, and it's the source of ~80% of production "duplicate processing" bugs.

## Definition

**`LockDuration`** is a per-queue configuration setting (set by the queue owner, not the consumer) that defines the initial lock window the broker grants when it hands a message to a consumer. Default: 30 seconds. Maximum: 5 minutes. Service Bus refuses values higher than 5 minutes.

**Lock renewal** is the act of sending an explicit message to the broker — `renew_message_lock(msg)` in the SDK — that resets the lock timer to `now + LockDuration`. As long as renewals keep arriving, the lock keeps extending. The moment they stop (consumer crashed, deadlocked, sync-blocked, network died), the timer runs out and the broker reclaims the message.

Together they implement the same pattern as TCP keepalives, OAuth refresh tokens, and Redis distributed lock TTL+renew: **bounded base + active heartbeat beats long-lived static trust every time.**

## Problem it solves

[[Lock as Server-Side Timer]] established that locks are bounded — the broker reclaims a message if the consumer goes silent. But what if a handler legitimately takes longer than the lock window? Consider:

- A handler that calls a third-party payment API which takes 45 seconds during peak load.
- A handler that runs a complex DB transaction that occasionally blocks for 60 seconds.
- A handler that processes a large file (extract → transform → write) over 90 seconds.

Two failed approaches:

| Approach | What breaks |
|---|---|
| **Set a very long `LockDuration` (e.g. 1 hour)** | A consumer that crashes silently holds messages for an hour. A rolling deploy that crashes 50 pods means hundreds of messages are invisible for an hour. The whole "broker reclaims on silence" safety net is gutted. |
| **Let consumers ask for a custom duration per message** | A buggy consumer asks for 24h then deadlocks → broker has no defense because it gave the budget to the entity it was supposed to defend against. |

The right design: **keep the base short, but let the consumer prove it's still alive**. That's renewal.

## Why previous solution was insufficient

Imagine an alternative design where the consumer pre-declares "I need 5 minutes for this one":

```
Consumer ──► "give me 5 minutes on this message"  ──► Broker
                                                       │
                                                  trusts the request,
                                                  starts a 5-min timer
```

What happens if the consumer's process freezes 30 seconds in? The broker keeps the timer running for the full 5 minutes — because that's what the consumer asked for. Five minutes of stalled queue. If the consumer asked for 30 minutes, that's 30 minutes of stalled queue. The consumer's request and the consumer's reliability are both *self-reported* — there's no independent verification.

Renewal flips this. The consumer doesn't *promise* it will be alive in 5 minutes; it has to *prove* it every few seconds. A frozen consumer can't send renewals — its liveness is encoded into the protocol itself.

> **Per-message duration is a promise the broker can't verify. Renewal is a signal the broker observes in real time.**

This is why the same pattern appears everywhere in distributed systems:

| System | Bounded base | Active renewal |
|---|---|---|
| TCP | Keepalive interval | Keepalive packets |
| OAuth | Access token expiry | Refresh token exchange |
| Raft | Election timeout | Leader heartbeats |
| Redis distributed lock | TTL on the key | Periodic `EXPIRE` extensions |
| Service Bus lock | `LockDuration` | `renew_message_lock` |

Same shape, same reason: **active proofs of liveness are unforgeable; static promises aren't.**

## Responsibilities

`LockDuration` and renewal own:

- **Initial budget** — `LockDuration` sets the *baseline* time before silence triggers reclamation.
- **Liveness proof channel** — `renew_message_lock` is the wire signal that says "I'm still working on this."
- **Reset behavior** — every renewal resets the timer to `now + LockDuration` (it doesn't *add* `LockDuration` to whatever was left).
- **Hard ceiling enforcement** — the broker refuses `LockDuration` > 5 minutes regardless of who asks.

What they do *not* own:

- **Retry counting** — `delivery_count` is incremented when the lock expires *or* the consumer abandons; renewal alone doesn't change it.
- **Dead-lettering** — `MaxDeliveryCount` is its own broker-side counter; renewal interacts with it indirectly (preventing accidental abandons that increment the count).
- **What the handler should actually do during the renewal window** — that's application logic.

## Why the queue owner picks `LockDuration`

The queue owner is the team running the system. The consumer is whoever's reading from the queue (could be many teams, many process versions). The principle:

> **The side that owns the resource sets the limit.**

If the consumer set `LockDuration`, you'd have:
- Consumer team A: "I need 30s, my handler is fast."
- Consumer team B: "I need 24h, my handler is slow and I don't want to renew."
- Buggy consumer: "I need infinity, also my handler deadlocks half the time."

The queue owner can't reason about its own queue's behavior. Stuck messages, hidden poison messages, broken auto-DLQ — all consumer-controlled.

Same shape as:

| System | Owner sets the limit | Consumer can't override |
|---|---|---|
| OS sockets | OS sets timeouts | App can request, OS can refuse |
| JWT | Issuer sets `exp` | Verifier can shorten but not extend |
| Postgres | DB sets `lock_timeout` | Client can request, DB enforces |
| Service Bus queue | Queue owner sets `LockDuration` | Consumer renews, can't directly extend |

The 5-minute hard ceiling is the same principle one layer up — Microsoft (the broker owner) capping what the queue owner can configure. Longer base = longer windows where silence isn't punished. If you need >5min, use renewal; that way silence still kills the lock fast.

## Renewal as a heartbeat — the mechanic

Manual renewal:

```python
async with receiver:
    msg = await receiver.receive_message()
    asyncio.create_task(renew_loop(receiver, msg))   # background task
    await long_running_work(msg)                     # may take minutes
    await receiver.complete_message(msg)
```

Where `renew_loop` is roughly:

```python
async def renew_loop(receiver, msg):
    while True:
        await asyncio.sleep(15)                    # half of LockDuration
        try:
            await receiver.renew_message_lock(msg)
        except Exception:
            return                                 # message likely already lost
```

Modern Python SDK auto-renewal:

```python
async with receiver:
    receiver.max_auto_lock_renewer_duration = 600   # 10 min total renewal budget
    msg = await receiver.receive_message()
    await long_running_work(msg)
    await receiver.complete_message(msg)
```

Under the hood the SDK runs the same renewal loop on a background task — you're just declaring an upper bound (`max_auto_lock_renewer_duration`) that prevents an infinite-loop handler from holding a message forever.

> **Auto-renewal isn't magic. The renewal task runs in the same event loop as your handler. Block the loop, starve the renewal.**

## The handler — the function you actually write

Some terminology that gets fuzzy:

> **Handler** = the function you write that processes one message. Everything between "SDK pulled it from broker" and "DISPOSITION sent" is your handler. The SDK runs the boilerplate (pull, dispose, renew); you provide the business logic.

```python
# This whole function is "the handler".
async def handle(msg):
    order = json.loads(msg.body)
    await charge_card(order)        # IO
    await db.write(order)           # IO
    # SDK calls complete_message after the handler returns.
```

The handler runs *inside* the SDK's event loop, not on its own thread. That's the source of the next anchor.

## Sync-blocking handlers — the killer mechanic

Modern consumers run on a single-threaded asyncio event loop juggling many cooperative tasks:

- The pull pipeline (asking the broker for the next batch)
- The auto-renewer task (sending `renew_message_lock` periodically)
- The Connection heartbeat task (keeping the AMQP Connection alive)
- **Your handler** (running for the current message)

`async` tasks yield control at every `await`. Sync calls do not — they hold the thread until they finish:

| Sync call | Holds the loop for | Effect |
|---|---|---|
| `requests.post(url)` | Up to network timeout (often 30-60s) | Renewal can't fire → lock expires |
| `time.sleep(10)` | 10 seconds | Same |
| `pyodbc.execute("SELECT ...")` | Until query returns | Same |
| Heavy `pandas` transform on a big DataFrame | Until CPU work finishes | Same |

While the loop is hogged:

```
T=0    Handler starts. Sync requests.post() begins.
T=15   Renewal *should* have fired. It can't — loop is blocked.
T=30   LockDuration expires. Broker reclaims the message.
T=30   Pod B receives the message. Starts processing.
T=45   requests.post() finally returns to original handler.
T=45   Original handler calls complete_message(msg)
       → MessageLockLostException (token is stale)
T=60   Pod B finishes, calls complete_message → success.
       Customer charged twice. Both side effects happened.
```

> **Sync-blocking handlers are the #1 cause of "different instances processing the same message" inconsistency in production.** Most Service Bus duplicate-processing bugs are not from rare crashes; they are from sync handlers silently losing locks every single message under load.

This is why payment, inventory, and email-sending handlers absolutely require idempotency — even a "perfectly written" handler will lose locks under enough sync-IO pressure.

### Common production fixes

| Problem | Fix |
|---|---|
| `requests` blocking the loop | `httpx.AsyncClient` |
| `psycopg2` blocking the loop | `asyncpg` |
| `pyodbc` blocking the loop | `aioodbc` (or push to a thread) |
| CPU-bound work (pandas, ML inference) | `await asyncio.to_thread(fn, ...)` — runs the call in a thread pool while the loop stays free |

## The smoking-gun debugging pattern

You're paged: customers are being charged twice. You pull DLQ.

You see a pile of messages with:

```
DeadLetterReason          = MaxDeliveryCountExceeded
DeliveryCount             = 10
MaxDeliveryCount          = 10           ← exactly equal
MessageLockLostException  in app logs    (or worse: nothing in app logs)
```

Notice the **uniform** `delivery_count = MaxDeliveryCount`. A real consumer-crash story would give you mixed numbers (1, 3, 7, 11) — different messages crash at different attempt counts depending on load and timing. **Uniform values mean deterministic failure on every attempt.**

Almost always, the cause is:

1. Sync handler starves auto-renewal.
2. Lock expires every single time.
3. `MessageLockLostException` is raised on `complete_message`.
4. A generic `except` swallows it — handler appears to succeed in app logs.
5. Broker re-queues the message → another pod picks it up → same sync handler → same lock loss.
6. Repeats `MaxDeliveryCount` times → auto-DLQ.

In the meantime, every successful side effect (the parts that ran *before* `complete_message` tried to settle) happened. Customer was charged 10 times.

**Fix order:**

1. **Async-ify the handler.** This is the actual root cause.
2. **Make `MessageLockLostException` a loud log + alert**, not a silent swallow.
3. **Verify auto-renew is configured** (`max_auto_lock_renewer_duration` set).
4. **Only then** consider increasing `LockDuration`. Bumping it from 30s to 60s without fixing the async problem just delays the same failure.

## Real-world example

A team at startup launches a Service Bus-backed payment service. Handler:

```python
async def handle(msg):
    order = json.loads(msg.body)
    response = requests.post(STRIPE_URL, json=order)   # sync — 4-8s under load
    db.execute("INSERT INTO charges ...", order.id)    # sync — pyodbc
    await receiver.complete_message(msg)
```

Works fine in load test (1 msg at a time). Goes to production with `prefetch_count=20`. Within an hour:

- Customers report duplicate charges.
- DLQ fills up with `MaxDeliveryCountExceeded`.
- App logs show "payment processed successfully" for every charge.

Mechanism: 20 messages prefetched. The first message's handler blocks the loop for 6 seconds calling Stripe. Auto-renewer can't fire on any of the other 19 messages either — the loop is single-threaded. By the time message #1 returns, messages #2-#20 have already lost their locks. Broker has redelivered them. Pod B is now processing them. Pod A's `complete_message` calls all raise `LockLostException`, get swallowed, and Pod A moves on like everything succeeded.

Fix:

```python
async with httpx.AsyncClient() as client:
    async def handle(msg):
        order = json.loads(msg.body)
        response = await client.post(STRIPE_URL, json=order)   # async
        await db.execute("INSERT INTO charges ...", order.id)  # asyncpg
        await receiver.complete_message(msg)
```

Loop now yields during Stripe call. Auto-renewer fires every 15s on every prefetched message. Lock losses stop. Duplicate charges stop. (Idempotency on `charges.id` was already a layer of defense — but the right fix is to stop creating the duplicates in the first place.)

## Mental model

> **The lock isn't a deadline you race against. It's a heartbeat the broker listens for.**
>
> Default `LockDuration` is just *how long the broker waits between heartbeats before assuming the consumer is dead.* Each `renew_message_lock` is a heartbeat. Each DISPOSITION is a final "I'm done." Stop heartbeats → broker reclaims. The consumer is constantly *justifying* its hold; silence is failure.
>
> Same shape as a treadmill safety key: as long as you keep the cord clipped to your shirt (active connection), the treadmill keeps running. Step off without unclipping (silent failure), and the treadmill stops. The treadmill doesn't trust you to remember to press stop — it trusts the cord.

## Interview answer

`LockDuration` is a per-queue setting on Service Bus, default 30 seconds, maximum 5 minutes — set by the queue owner, not the consumer, because the side that owns the resource sets the limit. When a handler legitimately needs more time, the consumer sends `renew_message_lock`, which resets the timer to `now + LockDuration`. Renewal isn't a budget extension — it's a heartbeat the broker observes in real time. A frozen consumer can't send renewals, so liveness is encoded in the protocol itself. Modern SDKs offer `max_auto_lock_renewer_duration` for automatic renewal, but the renewal task runs in the same event loop as the handler — so a sync-blocking call (`requests.post`, `time.sleep`, `pyodbc.execute`, CPU-heavy pandas) silently starves the renewer and causes lock loss every message. Sync handlers under load are the single most common source of "different instances processing the same message" bugs in production. The smoking-gun signature is a pile of DLQ messages with `DeadLetterReason=MaxDeliveryCountExceeded` and uniform `delivery_count` exactly equal to `MaxDeliveryCount`, while app logs show "success" for every message. Fix is async-ify the handler and stop swallowing `MessageLockLostException`.

## Common misconceptions

- **"`LockDuration` should be set as long as my slowest handler."** No. Long base means long windows where silence isn't punished. Use renewal instead — it gives you long handlers without weakening the safety net.
- **"Renewal extends the lock by `LockDuration` more time."** No. Renewal *resets* the timer to `now + LockDuration`. The remaining time isn't added.
- **"Auto-renewal makes my handler safe regardless of what it does."** No. Auto-renewal runs in the same event loop. Sync-blocking calls starve it.
- **"Renewing means the consumer controls how long it keeps the lock."** Effectively yes — but only as long as it keeps proving liveness. The moment it goes silent, the broker reclaims.
- **"5-minute max is arbitrary."** It's principled: 5 minutes is the longest base lock that still keeps "silence is failure" punishing. Anything longer should use renewal.
- **"`MessageLockLostException` means the broker is broken."** No. It means either you were too slow, or your TCP died, or your handler starved the renewer. The broker did exactly what it was supposed to do.
- **"If I see `MessageLockLostException`, I should retry the handler immediately."** No. The message is no longer yours — another consumer has it (or will have it shortly). Idempotency on side effects is the recovery, not retry.
- **"Async handlers are immune to lock loss."** Not immune — TCP can still die, the broker can still tear down a Connection. But async handlers don't *cause* lock loss from inside the consumer the way sync handlers do.

## See also

- [[Lock as Server-Side Timer]] — the timer this duration applies to
- [[Disposition States]] — what `Complete` / `Abandon` actually send to terminate the lock
- [[Idempotency]] — the recovery path when lock loss happens anyway
- [[Connection]] — the AMQP heartbeats that protect the *Connection*, parallel to lock renewal protecting the *message*

## Index

[[AMQP Message Lifecycle]]
