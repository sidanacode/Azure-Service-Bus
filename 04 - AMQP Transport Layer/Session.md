---
tags:
  - amqp
  - transport
---

# Session

> **A Session is a logical room inside an [[Connection|AMQP Connection]] with its own room number (channel id) and its own notebook of deliveries.** Every frame on the wire carries the room number stamped on it so the receiver knows which room to deliver it to. Inside the room, both sides maintain a synchronised notebook tracking what was sent, what got settled, and what is still in flight.

## Definition

A **Session** is a stateful conversation context that lives inside an AMQP Connection. It is identified by a **channel number** that is stamped on every [[Frames|frame]] flowing through it. Sessions provide three things a flat Connection cannot:

1. A namespace for **delivery-ids** — a counter that names every logical message-send made on this Session.
2. An independent **flow control window** — pacing this conversation without affecting the others.
3. An independent **error scope** — a failure here does not poison sibling Sessions on the same Connection.

Sessions do not hold messages. Queues hold messages. Sessions hold the *bookkeeping* of deliveries flowing through them.

## Problem it solves

A real producer process is rarely doing one thing. A typical app might be:

- publishing order events
- publishing audit logs
- waiting for replies on a reply queue
- publishing notifications

Four independent logical conversations. Each has its own pace, its own state, its own pattern of acks and errors.

If everything ran on a flat Connection with no per-conversation layer:

- **Flow control would collide.** "Slow down" applied to the whole Connection would freeze all four conversations even if only one consumer was overloaded.
- **Errors would blast everything.** A protocol error on one conversation would have to be reported on the Connection itself — and a Connection-level error tears the whole front door down.
- **State would tangle.** Sequence numbers, pending settlements, in-flight deliveries from four different conversations would have to share one global counter and one global pending-set, with no clean way to recover one without disturbing the others.

The Session layer fixes this by giving each conversation its own room.

## Why previous solution was insufficient

A naive design would say: "Just put a message-id on every message — the receiver can sort by id." This sounds reasonable but only solves one problem (sorting) while leaving three others.

A message-id is a **label**. A Session is a **container with state**. Specifically:

| What an ID-on-the-message gets you | What it doesn't |
|---|---|
| Receiver can sort messages by conversation ✅ | No place to keep sequence-number space ❌ |
| | No place to keep flow-control window ❌ |
| | No place to keep in-flight delivery state ❌ |
| | No place to scope an error ❌ |

If you had only message-ids, every application would end up reinventing Session in user code:

```python
state_by_conversation = {
    "orders":  {seq: 4721, pending: [...], credits: 87, ...},
    "audit":   {seq: 102,  pending: [...], credits: 50, ...},
}
```

That dictionary *is* the Session — just rebuilt poorly, per app. AMQP formalises it as a first-class layer so both ends agree on its shape and lifecycle.

## Responsibilities

A Session owns three things, all *per conversation*:

- **A delivery-id sequence space.** Every TRANSFER on this Session gets a fresh delivery-id from this Session's counter. Channel 1 having delivery #4721 does not conflict with channel 2 having its own #4721 — they live in different number spaces.
- **A flow control window.** Both sides agree at BEGIN-time how many unsettled deliveries can be in flight. The window is delivery-counted, not byte-counted, and it is per-Session — slowing one conversation does not slow the others.
- **An error scope.** A protocol error on this Session ends *this Session*. The Connection survives. Other Sessions on the Connection survive.

A Session also acts as a parent scope for [[Link|Links]] — Links exist *inside* a Session. The handles that name Links are scoped per-Session, and ending a Session ends every Link inside it.

What a Session does *not* own:

- It does not hold messages. Messages live on queues at the broker. The Session is just the conversation through which they flow.
- It does not own auth, version, heartbeat, or max-frame-size — those are [[Connection]] responsibilities.
- It does not know whether you are sending or receiving — direction is a Link concern.

## Two notebooks, kept in sync

Both ends of a Session keep what is essentially the same notebook. At BEGIN time they exchange the starting state and the window sizes; from then on, every TRANSFER frame and DISPOSITION frame keeps the two notebooks in sync.

```
   Producer                                 Broker
   ┌──────────────────────┐                 ┌──────────────────────┐
   │  Session "orders"    │                 │  Session "orders"    │
   │  (channel = 1)       │ ←─ same id ─→   │  (channel = 1)       │
   │                      │                 │                      │
   │  next-outgoing-id:   │                 │  next-incoming-id:   │
   │     4722             │                 │     4722             │
   │                      │                 │                      │
   │  unsettled:          │                 │  unsettled:          │
   │     #4720 in-flight  │                 │     #4720 received,  │
   │     #4721 in-flight  │                 │            disposing │
   │                      │                 │     #4721 received   │
   │                      │                 │                      │
   │  outgoing-window: 13 │                 │  incoming-window:200 │
   └──────────┬───────────┘                 └──────────▲───────────┘
              │                                        │
              │   frames stamped channel=1             │
              │   carry TRANSFER / DISPOSITION / FLOW  │
              └────────────────────────────────────────┘
```

Three frame bodies do all the keeping-in-sync:

| Frame body | Updates the notebook with |
|---|---|
| **TRANSFER** | "I sent delivery #N. Add it to unsettled." |
| **DISPOSITION** | "I have a verdict on delivery #N — accepted / rejected / released / modified. Update its state." |
| **FLOW** | "Here are my latest window sizes / next-incoming-id." |

## Channel ids are Connection-scoped

A channel number is **not globally unique**. It is meaningful only inside the one Connection it lives in.

The full address of any Session, from the broker's view, is:

```
(Connection identity, channel number)
```

Not just `channel number`. The Connection identity disambiguates.

Concretely:

```
                              ┌──────────────────────┐
   Producer A ───TCP1────────►│      Broker          │
   Connection 1               │                      │
     ├── Session ch=1         │  Sees Conn1.ch=1     │
     ├── Session ch=2         │  Sees Conn1.ch=2     │
     └── Session ch=3         │  Sees Conn1.ch=3     │
                              │                      │
   Producer B ───TCP2────────►│                      │
   Connection 2               │                      │
     ├── Session ch=1         │  Sees Conn2.ch=1     │
     └── Session ch=2         │  Sees Conn2.ch=2     │
                              └──────────────────────┘
```

Producer A's channel 1 and Producer B's channel 1 are **completely different Sessions**. There is no collision because frames from each producer arrive on different TCP sockets, and each socket carries its own Connection. "Channel 1" is interpreted *relative to the socket the frame came in on*.

Same idea as: "apartment 3" doesn't conflict between two different buildings — the building scope makes the apartment number unique within that building.

**The producer (its library) picks the channel number** when it opens a Session — it just allocates the next free slot from a per-Connection counter, sends a `BEGIN` on it, and the broker replies with its own `BEGIN` on the same channel. When the Session ends, that channel slot is released and a future Session on the same Connection can reuse it. Channel numbers are *slots*, not identities.

The cap on how many channel slots a Connection has is the `channel-max` value negotiated in the OPEN frame at Connection time. With `channel-max=256`, a producer can have up to 256 Sessions open at once on this one Connection.

## How a Session is born and dies

A Session opens with a `BEGIN` frame on a previously-unused channel and closes with an `END` frame:

```
Producer: [channel=1]  BEGIN(next-outgoing-id=0, incoming-window=200, outgoing-window=100)  ──►
                                                                          ◄── BEGIN(...)  Broker
              ─── Session 1 is now open. Producer and broker share its notebook. ───

Producer: [channel=1]  TRANSFER(delivery-id=0)  ──►
Producer: [channel=1]  TRANSFER(delivery-id=1)  ──►
                                                                       ◄── DISPOSITION(0, accepted)  Broker
                                                                       ◄── DISPOSITION(1, accepted)  Broker

Producer: [channel=1]  END  ──►
                                                                          ◄── END  Broker

              ─── Session 1 is gone. Channel 1 is free for reuse. ───
```

Sessions are **not durable**. When a Session ends — either gracefully via END, or because the Connection underneath died — the notebook is gone on both sides. There is no "resume Session 1 from where it left off." A new Session starts with a fresh delivery-id counter at zero.

This is fine because *messages* are durable in the queue, not in the Session. The Session was just a conversation; ending it does not lose any settled or queued messages.

## What survives a crash, and what doesn't

This is the most important consequence of Sessions being non-durable. Two completely different things must be kept separate in your head:

| Thing | Where it lives | Survives a crash? |
|---|---|---|
| **Session notebook** (delivery state — sent, settled, in-flight) | Producer's RAM + broker's RAM | ❌ No — gone with the Session |
| **The message itself** (the actual order data) | Broker's queue, on disk | ✅ Yes — durable |

A producer is sending delivery #2 when it crashes. After reconnect:

- The new TCP, new Connection, new Session on channel 1 has **no relationship** to the old Session that died. Same channel number, fresh notebook.
- The new Session's delivery counter starts at #0, not #3.
- Delivery #2 might already have been written to the broker's queue (durable, safe), or might not have arrived (lost). The producer cannot tell the difference from where it sits.

This is exactly the scenario [[Delivery Semantics|at-least-once]] and [[Idempotency]] were designed for:

```
Producer crash with delivery #2 in flight
       │
       ▼
Producer can't know if broker got it
       │
       ▼
Safe move: re-send the message after reconnect
       │
       ▼
If broker had already saved it → duplicate
       │
       ▼
Idempotent consumer absorbs the duplicate → no double-charge
```

The chain is load-bearing:

1. Sessions are not durable → bookkeeping is lost on crash.
2. → Producer cannot be sure if in-flight deliveries got through.
3. → Producer must re-send unconfirmed deliveries (at-least-once).
4. → Consumers must be idempotent to handle the duplicates this creates.

**Sessions being ephemeral isn't a flaw — it's the reason the rest of the safety story exists.** Persisting Session state to disk on every BEGIN/END/window-update would devastate broker performance for almost no benefit, because the *producer's* notebook crashed with the producer anyway. Durability lives where it belongs: in the queue.

## Settlement is not the same as durability

A subtle but important distinction. Two things people conflate:

| Step | What it means | Triggered by |
|---|---|---|
| **Durable** | The message has been written to the broker's disk and would survive a broker restart | Broker writes the queue file |
| **Settled** | The producer has been *told* the broker accepted the delivery, and both notebooks now agree it is done | Broker sends a DISPOSITION frame back |

Disk-write makes the *message* durable. The DISPOSITION frame makes the *delivery* settled. Settlement is the **acknowledgement of** the durable write — it requires a frame on the wire to communicate it. The producer cannot infer settlement from anything else; it has to be told via DISPOSITION.

So the full life of a successful delivery is:

```
1. Producer sends TRANSFER frames on channel 1, delivery-id=4721
2. Broker reassembles, writes message to queue on disk    ← durable
3. Broker sends DISPOSITION(4721, accepted) back          ← settled
4. Producer's notebook updates: #4721 settled, drop from unsettled
```

If the producer crashes between steps 2 and 3, the *message is durable but not settled* — and the producer has no way to know. That is why idempotent retry is the safe pattern.

## Real-world example

A producer process opens one Connection to the broker and runs three concurrent jobs. The library opens three Sessions — one per job — on channels 1, 2, 3:

```
Connection
   ├── Session "orders"   (channel 1)   high volume, fire-and-forget
   ├── Session "audit"    (channel 2)   low volume, must not lose
   └── Session "replies"  (channel 3)   sporadic
```

The broker's disk for the orders queue fills up. Broker stops dispositioning orders. Session 1's outgoing window fills up; producer pauses sends on channel 1.

Sessions 2 and 3 are unaffected. Audit logs keep flowing. Replies keep coming in. The producer's `audit-write` thread does not even know there is a problem on `orders`.

This is what per-Session flow control buys you: surgical pacing.

## Mental model

> **A Session is a room inside the broker's building.** The Connection is the front door of the building (auth, version, heartbeat all happen there). Once inside, you can open multiple rooms — each room has a number stamped on its door (the channel id). Every piece of paper passed through a room (a frame) has the room number written on it so it gets routed correctly. Inside each room, both you and the broker keep a synchronised notebook of every delivery: what was sent, what was settled, what is still pending.
>
> End the meeting in one room and that notebook is wiped — but the building stays open and other rooms keep working.

Or in two-column form:

| Connection | Session |
|---|---|
| Front door of the building | A meeting room inside it |
| Auth, version, heartbeat, max-frame-size | Sequence numbers, windows, in-flight deliveries |
| One per TCP socket | Many per Connection |
| Dies → everything dies | Dies → just this conversation dies |

### Airport analogy

The whole transport stack also maps cleanly onto airport operations, which can be easier to hold in one picture:

| Real world | AMQP |
|---|---|
| Sky / runway (raw infrastructure) | TCP connection — bytes carrier |
| Airport (the building, with security) | AMQP Connection — auth, version, heartbeat |
| Flight (one specific journey) | Session — one conversation |
| Gate number on the flight | Channel number on the Session |
| Boarding pass on every passenger | Channel id stamped on every frame |
| Flight manifest (who boarded, who deplaned) | Session's notebook of deliveries |
| Passenger (one logical traveler) | Delivery (one logical message-send) |
| Suitcase pieces (one passenger's three bags) | Frames of one delivery |
| "Last bag" tag on the carousel | `more=false` on the final TRANSFER frame |

A passenger (delivery) can have multiple bags (frames). All bags travel on the same flight (Session). The "last bag" tag tells the receiver the passenger's luggage is complete.

## Interview answer

A Session is a stateful conversation context that lives inside an AMQP Connection, identified by a channel number that is stamped on every frame flowing through it. Sessions exist because real applications run many independent conversations against the same broker — orders, audit logs, replies, notifications — and each needs its own delivery-id space, its own flow-control window, and its own error scope. Both ends maintain a synchronised notebook of in-flight deliveries, kept in sync by TRANSFER, DISPOSITION, and FLOW frames. Sessions are not durable: when a Session ends, the notebook is gone, but messages themselves are durable in queues so nothing is lost. Per-Session flow control is surgical — slowing one conversation does not slow the others — which is the key advantage over a flat Connection with message-ids.

## Common misconceptions

- **"A Session holds messages."** No. Messages live in queues on the broker. The Session is the conversation context through which messages flow. Ending a Session does not delete messages.
- **"Frames have a sequence number; the Session tracks frame sequences."** No. Frames carry a channel number (which Session) and, for TRANSFER frames, a delivery-id (which logical send). Frames themselves are anonymous and disposable. The Session tracks deliveries, not frames.
- **"The channel number identifies a frame."** No. The channel number identifies the *Session* that frame belongs to. Many frames share the same channel number.
- **"One Connection means one conversation."** No. One Connection can hold many Sessions, and Sessions can be opened and closed throughout the Connection's lifetime. The channel-max in the Connection's OPEN frame caps how many can be open at once.
- **"Sessions can be resumed after a crash."** No. Sessions are not durable. After a crash and reconnect, you open a fresh Session. Delivery durability comes from the broker's queue, not the Session.
- **"Sequence ids are global across the Connection."** No. Delivery-ids are scoped per-Session. Channel 1 and channel 2 both having a delivery #5 is normal — they refer to different deliveries in different conversations.
- **"Channel numbers are globally unique across the broker."** No. Channel numbers are scoped to one Connection. Producer A's channel 1 and Producer B's channel 1 are different Sessions — there is no collision because frames arrive on different TCP sockets, and the broker disambiguates by `(Connection, channel)`.
- **"Settled and durable mean the same thing."** No. *Durable* = message is written to the broker's disk. *Settled* = producer has been told via a DISPOSITION frame that the broker accepted the delivery. A message can be durable but not yet settled (the disk-write happened, but the DISPOSITION hasn't been delivered or processed yet).

## See also

- [[Connection]] — the outer layer, where Sessions live
- [[Frames]] — the channel-tagged units that carry Session state on the wire
- [[Link]] — the one-way pipes inside a Session where messages actually flow
- [[Messaging Semantics]] — settlement and disposition, the things Sessions track
- [[Delivery Semantics]] — at-least-once / at-most-once, supported by the Session's delivery-id space

## Index

[[AMQP Transport Layer]]
