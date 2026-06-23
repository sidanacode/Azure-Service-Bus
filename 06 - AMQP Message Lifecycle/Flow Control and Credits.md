---
tags:
  - amqp
  - transport
  - lifecycle
---

# Flow Control and Credits

> **The receiver controls the pace.** AMQP doesn't have a "slow down" frame — there's just `FLOW`, the 9th frame verb, that the receiver sends to grant credits. Sender at zero credits *must* stop. Same shape as TCP's receive window, one layer up. Credits don't decay; the receiver has to actively top them up. This is what makes consumer-driven backpressure automatic and ungameable.

## Definition

**Flow control** in AMQP is the mechanism that prevents a sender from overwhelming a receiver. It's implemented through:

- **`FLOW`** — the 9th frame verb (after OPEN, BEGIN, ATTACH, TRANSFER, DISPOSITION, DETACH, END, CLOSE). Sent by the receiver. Carries fields including `link-credit` ("I'm willing to receive this many more"), `delivery-count`, `available`, `drain`, and `echo`.
- **Credits** — receiver-controlled permissions to send. One credit = permission to send exactly one message. A sender at zero credits must stop. Sending without credits is a protocol violation.

The receiver decides the pace; the sender complies. There is no negotiation, no "please throttle" message — just the absence of credits, which forces the sender to wait.

## Problem it solves

[[Settlement Modes]] and [[Disposition States]] gave us the safety story — what happens to a delivery from "sent" through "settled." But safety isn't enough. A producer that ships 1M msg/sec at a consumer that can only process 100 msg/sec needs *something* to stop the producer from filling the broker (or the consumer's local buffer) until either side runs out of memory.

Three options exist:

| Option | What breaks |
|---|---|
| **(a) Sender decides the pace** | Buggy or aggressive sender drowns slow consumer. Same shape as DDoS — the side without skin in the game wins. |
| **(b) Broker meters out messages** | Works for broker→consumer, but doesn't help producer→broker. Also forces broker to have a separate flow-control mechanism per Link. |
| **(c) Receiver grants permissions; sender can only send what was granted** | Receiver knows its own capacity. Sender has no choice but to comply. The broker enforces the protocol rule. |

Option (c) is credits. The same shape as TCP's receive window, one protocol layer up — *the side that has to absorb the data is the side that controls how much can flow.*

## Why previous solution was insufficient

A naive approach: have the sender "slow down" voluntarily on a hint from the receiver. This is what some early messaging protocols tried (e.g. polling-based throttling). Two problems:

1. **The hint is advisory, not enforced.** A bug in the sender's throttle code = drowned receiver. The protocol can't tell the difference between "you didn't get my hint" and "you ignored my hint."
2. **The hint is reactive, not preventative.** By the time the receiver realizes it's overwhelmed and emits a hint, its buffers are already full. Damage is done.

Credits flip both:

- **Enforced**: zero credits → protocol forbids sending. A sender that ignores this is committing a protocol violation; the broker will close the Link.
- **Preventative**: the receiver grants only as many credits as it can absorb. The sender can never get ahead of the receiver's capacity in the first place.

> **Sending without credits is forbidden by the protocol, not just discouraged. The receiver doesn't have to defend itself — the protocol does it.**

## Responsibilities

Flow control owns:

- **Pacing** — how many messages can be in flight at once.
- **Backpressure** — automatic propagation of slowness from receiver upstream to sender, with no application code involvement.
- **Bounded queues** — local buffers (sender's outbox, receiver's prefetch buffer) can never grow unboundedly.
- **Liveness for empty pulls** — drain mode lets the receiver get a definite "nothing here" answer instead of waiting forever.

What it does *not* own:

- **Settlement** — DISPOSITION is a separate frame for a separate concern.
- **Routing** — credits are per-Link; they don't decide where messages go.
- **Persistence** — a credit lets you send a message; it doesn't say anything about whether the broker writes it to disk.
- **Cross-Link fairness** — each Link has its own credit pool. Two Links to the same queue have separate flow control.

## FLOW frame fields

```
FLOW frame (channel = N, handle = K)
├── link-credit       "I'm willing to receive this many more on Link K"
├── delivery-count    "I've processed up to delivery-id D" (for reconciliation)
├── available         (sender → receiver only) "I have this many ready to send"
├── drain             true = "send what you have OR send FLOW(credit=0) saying nothing left"
└── echo              true = "reply with a FLOW so I know you got this"
```

The two fields that matter day-to-day:

- **`link-credit`** — the valve. Receiver sets it; sender can send up to that many messages.
- **`drain`** — the "commit or admit nothing" flag. Used to implement bounded waits.

`echo` is a probe ("did this FLOW even arrive?"); rarely seen in app code. `available` is informational; not all brokers populate it.

## Credits don't decay — receiver must top up

This is the part that surprises people coming from TCP-style window-based flow control where the window slides automatically.

```
Receiver sends:    FLOW(link-credit=10)        ← grants 10 permissions
Sender sends:      TRANSFER(...)               ← credit count is now 9
Sender sends:      TRANSFER(...)               ← 8
... (8 more TRANSFERs)
Sender sends:      TRANSFER(...)               ← 0
Sender:            STOPS — no credits, can't send
                   *waits for receiver to send another FLOW*
Receiver sends:    FLOW(link-credit=10)        ← refill
Sender resumes.
```

Same shape as [[Lock Duration and Renewal]]: **active signal, not passive trust.** Silence from the receiver = sender stays starved. The credit balance is a tap valve the receiver opens; the sender pours until the valve closes itself (credit = 0); the receiver re-opens it as it drains its local buffer.

> **No credits = no send. There is no "send anyway and hope" path.** The protocol forbids it; the broker disconnects Links that violate it.

## Two refill strategies

How often should the receiver send FLOW frames to top up credits? Two patterns:

### Pattern A — refill after every DISPOSITION

```
TRANSFER(d=1) → DISPOSITION(d=1) → FLOW(link-credit=1)
TRANSFER(d=2) → DISPOSITION(d=2) → FLOW(link-credit=1)
TRANSFER(d=3) → DISPOSITION(d=3) → FLOW(link-credit=1)
...
```

Simple, predictable. Three frames per message instead of two. At high throughput this doubles the wire cost.

### Pattern B — threshold-based refill

```
Initial: FLOW(link-credit=100)         ← grant 100
TRANSFER → DISPO  (credit balance: 99)
TRANSFER → DISPO  (98)
... (50 more)
TRANSFER → DISPO  (49)                 ← crossed threshold (prefetch / 2)
                   FLOW(link-credit=51) ← refill back to 100 in one frame
TRANSFER → DISPO  (99)
... continues
```

One FLOW frame per ~50 messages instead of one per message. Much cheaper at high throughput. SDKs use Pattern B with sensible defaults.

> **A high `prefetch_count` with throughput plateau is often the FLOW refill cadence, not the handler.** The SDK is being conservative about how often it tops up. Tune by checking SDK config (`prefetch_count`, refill threshold) before assuming the handler is the bottleneck.

## Drain mode — "commit or admit nothing"

Without drain, an empty queue is invisible to the receiver. You granted 5 credits; the sender sees no messages to deliver; the sender just sits there with the 5 credits unused. Forever. There's no "queue is empty" frame.

Drain converts this into a definite answer:

```
Receiver:  FLOW(link-credit=5, drain=true)
                          │
                          ▼
              Broker MUST do one of two things:
              ┌─────────────────────┐
              │ A) Send messages    │ ── consumes credits normally
              │    (using credits)  │
              └─────────────────────┘
                       OR
              ┌──────────────────────────────────┐
              │ B) Send FLOW(link-credit=0)      │ ── "nothing here, I'm zeroing out"
              │    indicating queue is empty     │
              └──────────────────────────────────┘
```

After the drain completes, the receiver has either some messages or a definite "nothing." Without drain, you'd have to guess — wait some arbitrary timeout client-side and hope.

> **`max_wait_time` on `receive_messages` is implemented via drain mode underneath.** When you call `receive_messages(max_wait_time=10)`, the SDK sends a FLOW with `drain=true`; after 10 seconds (or sooner if messages arrive), it has either messages or a definite empty answer. Returning `[]` instead of hanging is drain mode completing.

This is one of those cases where a tiny protocol field unlocks a whole class of clean SDK APIs. Without drain, every "wait up to N seconds" client call would have to be a hack racing the broker's silence with a local timer.

## Asymmetry: lock is broker-only, credits are partly client-side

The lock state lives entirely on the broker (see [[Lock as Server-Side Timer]]). Credits are **partly** client-side:

- When the SDK *receives* a TRANSFER, it decrements its internal credit counter immediately — before your handler even runs.
- When the SDK *sends* a FLOW, the broker increments its mirror.
- Both sides include `delivery-count` in FLOW frames so they can reconcile if they drift.

This is why **a receiver with a high `prefetch_count` uses memory even when no `receive` call is active**:

```python
receiver = client.get_queue_receiver(queue_name, prefetch_count=100)
# At this point, even before any receive_message() call,
# the SDK has potentially pulled up to 100 messages into a local buffer
# because it granted 100 credits at link attach.
```

The broker filled the credits; the SDK is buffering them. Memory pressure at idle is the symptom.

## Backpressure on the wire — "valve not opening" is the signal

There's no explicit "throttle me" message in AMQP. There's just **"I'm not granting more credits."** The same shape as TCP's receive window: a stalled receiver advertises a smaller window; the sender naturally slows.

```
Receiver overwhelmed → handler getting slower → DISPOSITIONs slower
                    → SDK's threshold for refill not crossed
                    → no FLOW sent
                    → sender's credit count stays low
                    → sender's outbox fills
                    → sender's API call (e.g. send_messages) blocks
                    → upstream blocks
                    → backpressure has propagated all the way to the producer's caller
```

No application code wrote a single line of "slow down" logic. The protocol gives chained backpressure for free.

## Recognition triggers

When you see these in production code or SDK docs, you're seeing flow control in action:

| Symptom | What's happening underneath |
|---|---|
| `prefetch_count` config | Size of the credit grant on the receiver Link |
| `max_wait_time` on `receive_messages` | Implemented via drain mode |
| `receive_messages` returns `[]` instead of hanging | Drain mode completed; broker confirmed empty |
| Periodic FLOW frames in packet captures | Pattern B threshold-based refill |
| Receiver process using memory when idle | Local prefetch buffer holding pre-pulled messages |
| Sender's `send_messages` blocking under load | Outbox full because credits dried up |
| Throughput plateaus despite low handler latency | FLOW refill cadence, not handler — bump `prefetch_count` |

## Real-world example

A consumer service runs with `prefetch_count=200`. The handler does ~50ms of async work per message. Throughput plateaus at 1,000 msg/sec — half what you'd expect from 200 prefetch / 50ms.

Investigating with packet capture: FLOW frames arrive every ~100 messages, not after every DISPOSITION. The SDK is using Pattern B refill at threshold = `prefetch_count / 2`. So:

- At 200 credits, 100 messages flow.
- Buffer drops to 100 → SDK sends FLOW(credit=100) → buffer back to 200.
- Net effect: 200 messages buffered locally, 100 in flight at any time, but credits refill in chunks.

Bumping `prefetch_count` to 1000 doesn't help proportionally — handler latency matters too. But bumping to 500 with a SDK that refills at threshold 250 *does* help, because the SDK keeps the pipe full while old messages are still being handled.

The lesson: **credits + refill cadence + handler latency together determine throughput.** Tuning any one in isolation gives misleading results.

## Mental model

> **Credits are a tap valve in the kitchen, controlled by the kitchen, not the supplier.**
>
> The kitchen yells "send 10 more!" The supplier sends exactly 10. The supplier *cannot* send an 11th — there's no path through the wall for it. When the kitchen is ready for more, it yells again. If the kitchen goes quiet, the supplier waits.
>
> **Drain mode** is the kitchen yelling: "send what you have, OR yell back that you have nothing." It converts the supplier's silence into a definite answer.
>
> **Backpressure** is automatic: a slow kitchen → fewer "send more!" yells → supplier's queue fills → supplier's clients (the people upstream giving the supplier work) start waiting. No one had to write a "slow down" memo.

## Interview answer

Flow control in AMQP is implemented through credits, granted by the receiver via FLOW frames. Each credit is one permission to send one message. A sender at zero credits must stop — sending without credits is a protocol violation. Credits don't decay automatically; the receiver has to actively top them up by sending more FLOW frames, either after every DISPOSITION (Pattern A) or when the credit balance crosses a low-water threshold (Pattern B, what most SDKs use). Drain mode (`drain=true` on FLOW) forces the broker to either send messages or send a FLOW(credit=0) saying the queue is empty, which is how `receive_messages(max_wait_time=…)` is implemented underneath. The whole design is the same shape as TCP's receive window — the side that has to absorb the data controls how much can flow — and it gives backpressure to the application for free: a slow consumer means slower DISPOSITIONs means slower FLOW refills means a sender's outbox fills and its API blocks. No application code writes any "slow down" logic.

## Common misconceptions

- **"Credits decay automatically over time."** No. The receiver has to actively send FLOW. Silence from the receiver = sender stays at the current credit count.
- **"`prefetch_count` controls how many messages I process at once."** No. It controls how many the SDK pulls into its local buffer (the credit grant size). Concurrency is a separate setting.
- **"Sending without credits just gets the message queued at the broker."** No. It's a protocol violation; the broker will close the Link.
- **"Drain mode is rarely used."** No — it's used every single time you call `receive_messages` with a `max_wait_time`.
- **"If FLOW is missing from packet captures, the connection is broken."** Probably not. Pattern B refill means FLOW is rare under steady load — only when crossing the threshold.
- **"FLOW and DISPOSITION must be sent together."** No. They're decoupled wire-level. SDKs often group them, but the protocol allows independent timing.
- **"Credits work across all Links."** No. Each Link has its own credit pool. Two Links to the same queue have separate flow control.
- **"More `prefetch_count` always means more throughput."** No. Past a point, you're just buffering messages locally with no handler keeping up. Memory pressure rises, lock losses become more likely (locks are held longer), DLQ rate climbs.

## See also

- [[Link]] — the per-direction pipe credits live on
- [[Handshake Choreography]] — where the *first* FLOW frame appears (post-ATTACH initial credit grant)
- [[Lock Duration and Renewal]] — the same active-liveness pattern, applied to locks
- [[Disposition States]] — what triggers DISPOSITIONs that often pair with FLOW refills
- [[Connection]] — the layer above where heartbeats use the same active-signal-or-die pattern

## Index

[[AMQP Message Lifecycle]]
