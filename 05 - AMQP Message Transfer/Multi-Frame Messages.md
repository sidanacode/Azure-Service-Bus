---
tags:
  - amqp
  - transport
---

# Multi-Frame Messages

> **One message can be many frames.** When a message is bigger than `max-frame-size`, AMQP splits it into multiple TRANSFER frames that all share the same `delivery-id`, with a `more=true` flag on every frame except the last. The receiver buffers the parts, and only when `more=false` arrives does the broker treat the delivery as real. If the Connection breaks mid-message, the partial buffer is silently discarded — the consumer never sees a fragment.

## Definition

A **multi-frame message** is a single delivery that AMQP transmits across two or more TRANSFER frames. Every frame in the set carries the same `delivery-id` and the same `handle`; every frame except the last sets `more=true`; the closing frame sets `more=false`. The receiver concatenates the bodies, in order, and treats the result as one logical message — one entry in the [[Session]]'s notebook, one DISPOSITION, one unit of atomicity.

## Problem it solves

[[Connection]] negotiates a `max-frame-size` (e.g., 64 KB) at handshake time. That is a **per-frame buffer-size promise**: each side commits that no single frame it sends will exceed that size, so the receiver can pre-allocate one fixed buffer and never get caught off-guard.

But real messages can easily exceed that limit:

- Service Bus standard tier allows messages up to **256 KB**
- Service Bus premium tier allows up to **100 MB**
- Both are larger than typical `max-frame-size` defaults

If the protocol couldn't split messages across frames, you'd be stuck with one of three bad options:

1. Set `max-frame-size = max message size` → buffers in megabytes per Link, ruinous memory
2. Reject all messages bigger than the frame size → forces every consumer to design around an arbitrary protocol limit
3. Open a fresh Connection per large message → the same anti-pattern [[Handshake Choreography]] just argued against

The `more` flag is the clean answer: keep `max-frame-size` small (predictable buffers), and let messages span as many frames as they need.

## Why previous solution was insufficient

The naive approach — *"just give each chunk its own delivery-id and treat them as separate messages"* — destroys atomicity. From the broker's view:

```
Producer:  TRANSFER  delivery-id=0  body=[bytes 0..65535]
           TRANSFER  delivery-id=1  body=[bytes 65536..131071]
           TRANSFER  delivery-id=2  body=[bytes 131072..196607]
           TRANSFER  delivery-id=3  body=[bytes 196608..262143]
```

Now the broker sees **four separate deliveries**, four notebook entries, four DISPOSITION verdicts coming back. Every higher-level guarantee breaks:

| Thing that breaks | Why |
|---|---|
| **Settlement** | What if delivery #2 is `accepted` and #3 is `rejected`? Half a message accepted? |
| **Atomicity** | If the producer crashes after #2, the broker has half a message stored. Garbage. |
| **Routing** | If #1 arrives but #2 is delayed, a consumer might pull #1 and start parsing a fragment. |
| **Idempotency** | Duplicate detection works on a single id. One logical message split across four ids has no canonical id for dedup. |

The naive solution destroys "a message" as the unit of meaning — and "a message" is exactly what every higher layer (settlement, dedup, queues, consumers) is built on. The fix has to keep one delivery-id per logical message.

## The mechanism — `delivery-id` + `more` flag

AMQP solves this with two simple fields on every TRANSFER frame:

1. **The same `delivery-id` for every frame in the set.** All four frames carry `delivery-id=42`.
2. **A boolean `more` field:**
   - `more=true` → "I have more frames coming for this delivery."
   - `more=false` → "this is the last frame for this delivery."

Wire pattern for a 256 KB message split into 4 frames of 64 KB each:

```
TRANSFER  delivery-id=42  more=true   body=[bytes 0..65535]
TRANSFER  delivery-id=42  more=true   body=[bytes 65536..131071]
TRANSFER  delivery-id=42  more=true   body=[bytes 131072..196607]
TRANSFER  delivery-id=42  more=false  body=[bytes 196608..262143]   ← last frame
```

The broker's behaviour:

1. Receives frame 1 (`more=true`) → opens an in-memory buffer for delivery 42, writes bytes 0..65535
2. Receives frame 2 (`more=true`) → appends bytes 65536..131071 to delivery 42's buffer
3. Receives frame 3 (`more=true`) → appends more
4. Receives frame 4 (`more=false`) → **delivery 42 is now complete**. Only now does the broker reassemble, persist to disk, and produce a DISPOSITION.

**One entry in the notebook. One delivery-id. One settlement. One DISPOSITION.** The 4 TRANSFER frames are an implementation detail invisible to anyone above the transport layer.

## Responsibilities

What the multi-frame mechanism owns:

- **Splitting** at the producer's library — divide the message body into chunks no bigger than `max-frame-size`.
- **Tagging** every chunk with the same `delivery-id` and the right `more` flag.
- **Reassembly** at the receiver — buffer chunks, concatenate in order, hand the complete message to the next layer only when `more=false` arrives.
- **Discard-on-failure** — if the Connection drops while a delivery's buffer is still partial, throw the buffer away.

What it does *not* own:

- **Cross-frame error recovery** — that's TCP's job. AMQP can rely on TCP's ordered, lossless delivery; it never sees "frame 2 arrived but frame 3 was corrupted." If TCP can't deliver the frames in order, the Connection breaks and the partial buffer is discarded.
- **Per-frame credit** — credits are message-counted, not frame-counted (see below).
- **Per-frame settlement** — the whole delivery is settled as one unit, never partially.

## Credits are message-counted, not frame-counted

This is the second important consequence of the `more` flag.

A naive flow-control design might count frames instead of messages. That breaks multi-frame transfers: imagine a producer with 1 credit, mid-way through sending a 4-frame message — broker says "you've used your credit, stop." The producer is now stuck with 3 frames already on the wire and can't finish. The receiver has a half-buffered delivery it can never close. Deadlock.

AMQP solves this by counting at the *delivery* level, not the frame level:

> **Credits are message-counted. One credit lets the producer send one full delivery — however many frames it takes. Once the producer starts sending frames for a delivery, it is allowed to finish through the closing `more=false`, even if its credit count drops to 0 mid-message.**

In practice:
- Credit check happens at the *start* of a delivery (before frame 1).
- Frames 2..N for the same delivery are guaranteed to be allowed.
- The next delivery's frame 1 is gated by the next credit grant.

This keeps multi-frame transfers atomic at the flow-control layer, the same way `more=false` keeps them atomic at the persistence layer.

## What happens when the Connection breaks mid-message

This is the failure mode that makes the design feel solid. Scenario:

```
Producer sends frame 1 (more=true)  ✅
Producer sends frame 2 (more=true)  ✅
Producer sends frame 3 (more=true)  ✅
Producer crashes before sending frame 4 ❌
```

Broker has 192 KB buffered for delivery 42, waiting for `more=false`. What it does:

1. The Connection drops (heartbeat timeout, or TCP RST after the producer process dies).
2. Broker sees Connection gone → discards the partial buffer for delivery 42.
3. Delivery 42 is **never persisted** to the queue file on disk.
4. **A consumer of `orders-queue` sees nothing** — neither a partial blob, nor an error message. From the consumer's perspective, delivery 42 simply never existed.

The consumer never had to think about partial messages because the broker *can't* deliver one. The protocol guarantees: every message a consumer sees is complete, or the consumer sees nothing at all.

After reconnect, the producer's library re-sends the message with a fresh delivery-id. If the broker had already persisted the message via some earlier successful attempt, this creates a duplicate — which is exactly why [[Idempotency]] is the floor of the safety story.

> **The retry produces a complete duplicate, never a half-attempt.** That's only possible because the `more=false` rule discards partial deliveries silently.

## Real-world example

A Service Bus producer in Python sends a 256 KB JSON document containing an order with embedded line-item details:

```python
import json
big_message = ServiceBusMessage(json.dumps({
    "order_id": "abc-123",
    "customer": {...},
    "line_items": [...],   # 250 KB of nested data
}))
await sender.send_messages(big_message)
```

The Service Bus SDK transparently:

1. Serialises the message.
2. Notices the body is 256 KB but `max-frame-size` was negotiated at 64 KB during OPEN.
3. Splits the body into 4 chunks (≤ 64 KB each), wraps each in a TRANSFER frame:
   ```
   TRANSFER delivery-id=N  more=true   body=[chunk 1]
   TRANSFER delivery-id=N  more=true   body=[chunk 2]
   TRANSFER delivery-id=N  more=true   body=[chunk 3]
   TRANSFER delivery-id=N  more=false  body=[chunk 4]
   ```
4. Writes all 4 frames to the TCP socket in order.
5. Waits for the broker's DISPOSITION on delivery `N`.

The application sees one `send_messages` call, one return value. The 4-frame split is invisible.

If the producer's process dies after frame 3, the Service Bus SDK's retry logic (on the *next* run, or on the SDK's automatic reconnect) re-sends the whole message with a fresh delivery-id. The broker's dedup index — keyed on `MessageId` — catches the duplicate if dedup is enabled and the retry happens within the dedup window.

## Mental model

> **The `more` flag turns N frames into 1 atomic letter.** Each frame is one envelope in a stack of envelopes; they all have the same letter ID written on them, and the *last* envelope has a "no more after this" sticker on the corner. The post office holds back delivery until that closing envelope shows up. If the sender's truck crashes halfway, the post office throws away the partial stack — nothing leaks to the addressee.

### Airport analogy (continued)

Picking up the analogy from [[Session]] and [[Link]]:

| Real world | AMQP |
|---|---|
| Passenger | Delivery |
| Suitcase pieces | Frames of one delivery |
| **Bag tag with passenger ID** | **`delivery-id` repeated on every frame** |
| **"Last bag" sticker** | **`more=false` on the closing frame** |
| Carousel doesn't release bags until all are tagged complete | Broker doesn't persist the message until `more=false` arrives |

A passenger can have multiple bags. They all ride the same flight (Session), come down the same carousel (Link), and the airport workers release the passenger's stuff only when the "last bag" sticker has been seen. If the flight is cancelled mid-air, the partially-delivered bags get re-routed — the passenger never gets a half-set.

## Interview answer

A multi-frame message is one logical AMQP delivery split across two or more TRANSFER frames. All frames carry the same `delivery-id` and `handle`; every frame except the last sets `more=true`. The receiver buffers the chunks and only treats the delivery as real when `more=false` arrives — at which point it persists to disk and emits one DISPOSITION. This keeps the message as a single atomic unit even though the wire transports it in pieces. `max-frame-size` (negotiated at Connection time) is a buffering contract — "no single frame from me will exceed X" — not a message-size limit. Service Bus messages routinely exceed `max-frame-size`; the SDK splits and reassembles transparently. If the Connection breaks mid-message, the partial buffer is silently discarded, so the consumer never sees a fragment. Credits are message-counted, not frame-counted, so a producer that has started sending frames for a delivery is always allowed to finish through `more=false`. This combination — atomic delivery-id, `more` flag, message-counted credits — is what lets at-least-once retries produce *complete duplicates* rather than fragments, which is what makes [[Idempotency]] a clean defense.

## Common misconceptions

- **"`max-frame-size` is the maximum message size."** No. It's a buffer-size promise per frame. Message size is bounded by broker policy (Service Bus standard: 256 KB; premium: 100 MB), not by `max-frame-size`.
- **"A multi-frame message gets multiple DISPOSITIONs, one per frame."** No. There's exactly one DISPOSITION per delivery, regardless of how many frames it took.
- **"Each frame in a multi-frame transfer has its own delivery-id."** No. All frames share one delivery-id. That's what marks them as parts of the same delivery.
- **"If frame 2 is corrupted, the receiver asks for a retransmit."** No. AMQP doesn't have a per-frame retransmit. TCP guarantees ordered, lossless delivery — if TCP can't deliver, the Connection breaks and the whole partial delivery is discarded.
- **"The consumer sees the message as it streams in."** No. Consumers only see messages after `more=false` triggers persistence. There's no streaming-style partial delivery to consumers.
- **"Producer crashes mid-message → consumer gets a fragment."** No. The broker discards partial buffers on Connection loss. Consumer sees nothing.
- **"Credits get used per frame, so a multi-frame message uses N credits."** No. Credits are message-counted. A multi-frame delivery consumes one credit and is guaranteed to be allowed to complete.
- **"`more=true` is a separate frame type."** No. `more` is just a boolean field on the regular TRANSFER frame.

## See also

- [[Frames]] — the 8-byte frame header `delivery-id` and `more` ride on top of
- [[Connection]] — where `max-frame-size` is negotiated
- [[Handshake Choreography]] — the OPEN/BEGIN/ATTACH preamble that decides `max-frame-size` before any TRANSFER fires
- [[Settlement Modes]] — settlement is per-delivery, never per-frame; multi-frame is invisible to settlement
- [[Idempotency]] — what makes the at-least-once retry safe when partial messages get re-sent as full duplicates

## Index

[[AMQP Message Transfer]]
