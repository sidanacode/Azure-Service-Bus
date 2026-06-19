---
tags:
  - amqp
  - transport
---

# Link

> **A Link is a one-way pipe inside a [[Session]] that carries messages between the producer/consumer and a single queue or topic.** Every Link has a fixed direction (sender or receiver), a fixed address (target or source), its own credit pool for flow control, and a Session-scoped tag called a **handle** that routes frames to the right pipe. Sessions stamp and track frames; Links direct them.

## Definition

A **Link** is a unidirectional channel established between a client and the broker, scoped inside a Session, identified by a small integer called a **handle**. A Link binds three things at attach-time:

1. A **direction** — sender (client → broker) or receiver (broker → client).
2. An **address** — the queue or topic this Link sends to (`target`) or reads from (`source`).
3. A **credit pool** — the receiver-controlled count of how many messages the sender may currently have in flight.

Every TRANSFER frame on a Link carries the Link's handle, so the receiver can route it to the right pipe inside the right Session.

## Problem it solves

A Session by itself is **direction-blind**. It stamps every frame with a channel number and tracks delivery-ids in a notebook, but it doesn't know:

- which direction this delivery is moving
- which queue or topic this delivery is going to / coming from
- how to apply *per-flow* backpressure — a single Session might be carrying multiple unrelated streams of messages

Without a layer below Session, three things break:

1. **No routing.** The broker reads a TRANSFER frame and knows the Session, but has no idea whether to push the message into `orders-queue` or `inventory-queue`. The Session alone cannot tell it.
2. **No per-stream pacing.** A consumer reading slowly from one queue would have to slow the *whole Session* — every other queue feeding into the same Session would be dragged down with it.
3. **No clean lifecycle for one stream.** Closing a single queue's flow would mean tearing down the entire Session and every other queue with it.

Links fix all three by giving each *stream-in-one-direction-to-one-address* its own first-class object.

## Why previous solution was insufficient

A naive design might say: "Just put the queue name in every TRANSFER frame's body — the broker can route from that." It half-works, and is rejected for three concrete reasons:

| What the per-frame address gives you | What it doesn't |
|---|---|
| Broker can route a single message ✅ | No place to maintain a per-stream credit pool ❌ |
| | No place to scope errors to one stream ❌ |
| | Wastes bytes on every frame instead of negotiating once at attach-time ❌ |

The address is *stable for the lifetime of the pipe* — if you're sending order events, every message goes to `orders-queue`. Repeating that in every frame is wasteful. The clean design is: agree on the address **once**, when the pipe opens, then tag every frame with a small handle that points to that pipe.

That's what ATTACH does. Once a Link is attached, the broker knows: *"any TRANSFER on (channel=3, handle=0) goes to `orders-queue`."* The address is not in the frame — it's in the broker's memory of what handle 0 means on this Session.

## Responsibilities

A Link owns four things:

- **Direction.** Fixed at attach time. A sender Link can never carry messages back; a receiver Link can never push messages out. Bidirectional flow requires two Links.
- **Address.** A sender Link has a `target` (the queue/topic it sends to). A receiver Link has a `source` (the queue/topic it reads from). Set in the ATTACH frame, never changes during the Link's life.
- **Credit pool.** Per-Link, receiver-controlled. The sender may not transmit a message it doesn't have a credit for. Refills happen via FLOW frames from the receiver.
- **Lifecycle state.** Attached, detaching, detached, errored. Independent of every other Link on the same Session.

What a Link does *not* own:

- It doesn't track delivery state — that's the Session's notebook (delivery-ids, settlement).
- It doesn't see TCP, packets, frames, channels — those live above and below.
- It doesn't manage durability, persistence, or what the broker does with the message after it arrives — Links end at the broker's queue boundary.

## The handle — Link's per-Session tag

Just as a [[Session]] needs a **channel number** to disambiguate "which Session is this frame for?" on a shared TCP socket, Links need a **handle** to disambiguate "which Link is this frame for?" inside a shared Session.

The handle is just a small integer (0, 1, 2, …) that the producer's library picks when it opens the Link. ATTACH says: *"open a new Link on this Session, give it handle X, target = `orders-queue`, direction = sender."* From then on, every TRANSFER, FLOW, and DETACH frame for this Link carries `handle = X`.

Handles are **Session-scoped**, not global. Just like channels are Connection-scoped. Same pattern, one layer down:

```
TCP socket          ──  bytes
 └── Connection     ──  identified by the socket
      └── Session   ──  identified by channel  (Connection-scoped)
           └── Link ──  identified by handle    (Session-scoped)
```

This means handle `0` on Session 1 and handle `0` on Session 2 are completely different Links — and that's fine, because frames also carry the channel number to disambiguate the Session.

The full routing address for any TRANSFER frame is:

```
(channel, handle, delivery-id)
   │        │         │
   │        │         └──  which delivery in the Session's notebook
   │        └──────────────  which Link inside the Session  (the pipe)
   └───────────────────────  which Session inside the Connection
```

Three identifiers, three layers of routing. Every frame on the wire carries all three.

## Two Links can share a destination

A subtle but important property: **AMQP does not require Link targets to be unique.** Two Links on the same Session can both point to `orders-queue`. The Session doesn't check; it just sees handle 0 and handle 1 as two different pipes.

Why would anyone do this?

1. **Throughput / parallelism.** Each Link has its own credit pool. Two Links to the same queue means two independent credit windows, which means twice as many messages can be in flight to that queue at the same time. This is real backpressure parallelism, not just thread parallelism.
2. **Lane separation.** You might run a "VIP" Link and a "normal" Link to the same queue, with different credit policies, different retry logic, different metrics — same destination, different operational treatment. Closing one drops VIP traffic without affecting normal traffic.
3. **Independent lifecycle.** One Link can fail or be detached without affecting the other.

The handle is about pipes, not about destinations. Two pipes can share a destination — what matters is that frames carry the handle, so each pipe gets its own credit, its own state, its own lifecycle.

## Why Links are one-way

A natural question: why not let one Link carry both directions? It seems like it would be simpler — fewer objects to manage.

The answer is **flow control**. Credit is per-direction by nature: "I, the receiver, grant you, the sender, 100 credits to send to me." A bidirectional pipe would need two credit pools on one Link — one per direction — and now the credit-update logic has to specify *which direction* every FLOW frame is for. The clean per-Link model collapses.

Splitting into two Links keeps each pipe with **one credit pool, one direction, one job**. Symmetric and simple. If you need bidirectional flow, you open two Links — one sender, one receiver — and that's exactly how request/reply patterns work in AMQP (one Link sends commands, another receives replies, often on the same Session).

## Credits — flow control at the Link layer

The most important thing happening on a Link is **credit-based flow control**. This is how AMQP achieves backpressure without timeouts, sleeps, or rate guesses.

### The mechanism

A **credit** is a single unit of permission: *"you may send me one more message."*

Two simple rules:

1. **Receiver grants credits** by sending a FLOW frame: *"You have N credits."*
2. **Sender spends credits** — every TRANSFER decrements the sender's credit count by 1. **At zero, the sender must stop.** No exceptions.

When the receiver has digested some messages and has room for more, it sends another FLOW frame topping the pool back up. The sender resumes.

### Why this beats rate-based pacing

Three classes of failure that "1000 messages per second" can't handle:

| Problem | Why credits fix it |
|---|---|
| Receiver slows down mid-stream (disk fills up, GC pause) | Receiver simply stops granting more credits — no broken promise |
| Sender is already slow, capacity wasted | Receiver grants a large pool; sender uses it as fast or slow as it wants |
| Burst arrives in 10ms, crushes the receiver | Pool size *is* the burst limit, controlled by the receiver |

The receiver is **always in control**. The sender cannot send a message it doesn't have a credit for. **Backpressure is built into the protocol** — no application code is needed for it.

### Backpressure propagates up the chain for free

Because no-credits = no-send, the moment a broker stops granting credits:
- Producer's send-buffer fills
- Producer's API calls block
- Whatever code calls the producer (HTTP handler, scheduled job, background worker) starts blocking
- Backpressure has propagated all the way up the application stack — without anyone writing pacing code

This is why Service Bus producers don't need explicit rate limiting in the application: AMQP gives it for free at the transport layer.

### When the sender is stuck at zero

If the sender has exhausted its credits and no FLOW frame arrives, the sender **must wait** — it cannot send anyway, that would violate the protocol. The recovery escalation in real producer libraries:

1. Wait, with an application-level timeout (e.g. 60s).
2. If timeout fires — log it, increment a "stalled link" metric.
3. Try DETACH + re-ATTACH on the same Link.
4. If still stuck — close the Connection.
5. Reconnect to a different broker / partition (Service Bus uses geo-replication for this).
6. If everything is broken — surface the error to the application.

Disconnect is step 4+, never step 1. **Always try the cheap recovery first.**

A stuck producer is the receiver's fault, by definition: in credit-based flow control, the receiver is the *only* party that can unblock the sender. That's the whole point of the asymmetry.

## Frame types involved across a Link's lifetime

Recall: a [[Frames|frame]] is the smallest standalone unit AMQP writes on the wire. Every frame has the same fixed 8-byte header:

```
┌──────────────┬──────┬──────┬───────────────────────┐
│ size (4B)    │ DOFF │ TYPE │  CHANNEL (2B)         │
├──────────────┴──────┴──────┴───────────────────────┤
│              FRAME BODY  (the performative)         │
└─────────────────────────────────────────────────────┘
```

| Header field | What it tells the receiver |
|---|---|
| **size** | Length prefix — where this frame ends in the byte stream |
| **DOFF** | Data offset — where the body starts (lets the header grow in future versions) |
| **TYPE** | 0 = AMQP frame, 1 = SASL (pre-auth) frame |
| **CHANNEL** | Which [[Session]] this frame routes to (the multiplexing tag) |

The body holds a **performative** — the verb. A Link's life is a sequence of these performatives:

| Performative | Layer | Direction | When it's sent |
|---|---|---|---|
| `ATTACH` | Link | both ends | Open a Link — names the handle, direction, target/source |
| `FLOW` | Link / Session | receiver → sender | Grant or update credits, share window state |
| `TRANSFER` | Link | sender → receiver | Carry a message (or part of one if it spans many frames) |
| `DISPOSITION` | Session | receiver → sender | Settle a delivery — accepted / rejected / released / modified |
| `DETACH` | Link | either end | Close this Link cleanly (the other Links stay open) |

Every TRANSFER, FLOW, and DETACH frame for a given Link carries the same `handle` value in its body. The CHANNEL field in the frame *header* says which Session; the handle field in the frame *body* says which Link inside that Session. Two layers of multiplexing on every frame.

### A typical Link's life on the wire

```
Producer                                        Broker
   │                                               │
   │── ATTACH(handle=0, role=sender,               │
   │          target='orders-queue')             ─►│
   │◄─                ATTACH(role=receiver, ...)   │
   │                                               │
   │◄─                FLOW(handle=0, credit=100)   │   broker grants 100 credits
   │                                               │
   │── TRANSFER(handle=0, delivery-id=0, msg)    ─►│   credit=99
   │── TRANSFER(handle=0, delivery-id=1, msg)    ─►│   credit=98
   │   ...                                         │
   │── TRANSFER(handle=0, delivery-id=99, msg)   ─►│   credit=0  →  producer stops
   │                                               │
   │◄─        DISPOSITION(role=receiver,           │   broker confirms delivery #0 saved
   │              first=0, state=accepted)         │
   │   ...                                         │
   │◄─                FLOW(handle=0, credit=100)   │   credits refilled, producer resumes
   │                                               │
   │── DETACH(handle=0)                          ─►│   close the Link cleanly
   │◄─                              DETACH(handle=0)│
```

Each of those arrows is a frame on the wire. Each frame is length-prefixed (so the receiver can find boundaries), channel-tagged (so the receiver routes it to the right Session), and most carry a handle in the body (so the receiver routes it to the right Link inside that Session).

## Real-world example

A Service Bus producer process opens **one Connection** to the broker. Inside it, it opens **one Session**. Inside that Session, it opens **two Links**:

```
Connection
 └── Session  (channel=1)
      ├── Link, handle=0  →  SENDER  →  orders-queue
      └── Link, handle=1  →  SENDER  →  inventory-queue
```

The application code calls `producer.send(orders_message)` and `producer.send(inventory_message)`. The library:

1. Picks the right Link based on the destination.
2. Allocates a fresh delivery-id from the Session's counter.
3. Builds a TRANSFER frame: `channel=1, handle=0 (or 1), delivery-id=N, body=message`.
4. Writes the frame's bytes to the TCP socket.

On the broker side:

1. TCP delivers bytes.
2. AMQP library parses frame boundaries via the size prefix.
3. Reads `channel=1` → routes the frame to Session #1.
4. Reads `handle=0` (or `handle=1`) → routes it to the right Link.
5. Link knows its target — `orders-queue` or `inventory-queue` — writes the message to that queue.

If the orders queue's disk fills up, the broker just stops sending FLOW frames for handle 0. Link 0's credits run out, Link 0's sends pause. Link 1 keeps flowing into `inventory-queue`. **Surgical pacing per Link** — one stream slows without affecting the other.

## Mental model

> **A Link is a one-way pipe inside a meeting room.** The Connection is the building (front door, security). The Session is a meeting room inside the building (notebook of who said what). The Link is one specific pipe inside the room — a chute that carries paper in one direction only, connected to one specific outbox or inbox somewhere in the building. Every piece of paper sent through the chute has the chute's handle stamped on it, so even if the room has multiple chutes, the receiving end knows which chute each paper came from. And the chute itself has a flow valve — the receiver controls exactly how much paper it's willing to accept at any moment.

### Airport analogy (continued)

Picking up the airport analogy from [[Session]]:

| Real world | AMQP |
|---|---|
| Sky / runway | TCP |
| Airport (the building) | Connection |
| Flight (one journey) | Session |
| Gate number on the flight | Channel number |
| Boarding pass on every passenger | Channel id stamped on every frame |
| Flight manifest | Session's notebook of deliveries |
| Passenger | Delivery |
| Suitcase pieces | Frames of one delivery |
| **Boarding queue / luggage carousel** | **Link** |
| **Queue/carousel ID stamped on every bag tag** | **Handle on every frame** |
| **Capacity of the carousel ("can hold 100 bags")** | **Credit pool** |
| **"Last bag" tag on the carousel** | **`more=false` on the final TRANSFER frame** |

A flight (Session) has multiple boarding queues and luggage carousels (Links) — each one connected to a specific gate or baggage claim (target/source). Each carousel has a capacity (credits) and only handles bags moving in one direction. Bags from different carousels can ride the same flight; the bag tag tells the airport workers which carousel each bag came from.

## Interview answer

A Link is a one-way pipe inside a Session that carries messages between a client and a single queue or topic on the broker. Every Link has a fixed direction (sender or receiver), a fixed address (target for a sender, source for a receiver), and its own credit-based flow control pool. Links are identified inside their Session by a small integer called a handle, and every TRANSFER, FLOW, and DETACH frame for that Link carries the handle in its body. This gives AMQP three layers of routing on every frame: the channel number identifies the Session, the handle identifies the Link, and the delivery-id identifies the specific message inside that Session's notebook. Credits are per-Link and receiver-controlled: the sender may not transmit a message it doesn't have a credit for, so backpressure is built into the protocol — when the broker is overwhelmed, it just stops granting credits and the producer naturally pauses, with the pressure propagating up the application stack for free. Two Links on the same Session can point at the same queue: this is how AMQP achieves throughput parallelism inside a single Session without opening more Connections.

## Common misconceptions

- **"A Link is a TCP connection."** No. TCP gives you one byte stream per socket. A Connection rides on a TCP socket; many Sessions ride on a Connection; many Links ride inside each Session. Links are at least three layers above TCP.
- **"A Link is the same as a Session."** No. A Session is a stateful conversation context with a notebook of deliveries. A Link is a one-way pipe inside that Session, attached to a specific queue. One Session can hold many Links.
- **"A Link is bidirectional — a sender Link can also receive replies."** No. Every Link has a fixed direction set at ATTACH time. To send commands and receive replies, you open two Links (typically on the same Session): one sender, one receiver.
- **"The handle and the channel are the same thing."** No. Channel identifies the Session inside a Connection; handle identifies the Link inside a Session. Every frame carries the channel in its header and (for Link-scoped performatives) the handle in its body.
- **"Two Links on the same Session must point to different queues."** No. AMQP does not require unique targets. Two Links to the same queue gives you twice the credit pool — used for throughput parallelism and lane separation.
- **"Credits are per-Connection."** No. Credits are per-Link. Each Link has its own pool, refilled independently by FLOW frames. (Sessions have their own broader window, but the fine-grained flow control is at the Link layer.)
- **"Credits are time-based — like a 1000/sec rate limit."** No. A credit is one unit of permission to send one message. The receiver decides when to grant more, based on its own capacity, not a clock.
- **"If the sender runs out of credits, it should send anyway and let the broker drop overflow."** No. Sending without credit is a protocol violation. The sender must wait for FLOW or escalate (timeout → re-attach → reconnect).
- **"Closing a Link closes its Session."** No. DETACH closes only the Link. Other Links on the same Session, and the Session itself, stay open. END closes the Session; CLOSE closes the Connection.

## See also

- [[Session]] — the conversation context that contains Links
- [[Connection]] — the outer layer that contains Sessions
- [[Frames]] — the channel-tagged, handle-bearing units on the wire
- [[Messaging Semantics]] — settlement, the thing the receiver does after a TRANSFER
- [[Delivery Semantics]] — at-least-once / idempotency, the safety story credits enable

## Index

[[AMQP Transport Layer]]
