---
tags:
  - amqp
  - transport
---

# Handshake Choreography

> **Sending the first message takes ~9 frames; sending each subsequent message takes 2.** The first 4 frames out (OPEN → BEGIN → ATTACH → TRANSFER) and 5 frames back (mirrored OPEN/BEGIN/ATTACH + initial FLOW + DISPOSITION) build a Connection, Session, and Link before a single byte of message data can flow. Once that's done, every additional message reuses the same plumbing.

## Definition

The **handshake choreography** is the deterministic frame sequence both ends exchange to bring an AMQP conversation from "TCP socket just connected" to "first message persisted in queue." Each layer (Connection, Session, Link) is opened with its own pair of frames — one declaration from the producer, one mirrored confirmation from the broker — followed by a credit grant before the first TRANSFER is allowed.

## Problem it solves

[[TCP]] gives you bytes the moment the 3-way handshake completes. AMQP needs more than bytes — it needs:

- Agreed **auth, version, and `max-frame-size`** before either side can safely emit larger frames (the [[Connection]] layer)
- Agreed **window sizes and a delivery-id space** so both sides can track in-flight deliveries (the [[Session]] layer)
- Agreed **direction, target queue, and settlement mode** before any specific message can be addressed somewhere (the [[Link]] layer)

You can't skip any of these. Every TRANSFER frame's body is interpreted *relative to* the parameters negotiated in the OPEN/BEGIN/ATTACH preamble. Without the preamble, the frame is meaningless bytes.

## Why previous solution was insufficient

A naive design would say: *"just put all the parameters in the first frame — auth, version, queue name, max-frame-size, all of it together."* That collapses for three reasons:

1. **Different layers, different lifecycles.** A Connection lasts hours. A Session might last seconds. A Link might be one of dozens opened and closed during one Session. Bundling everything into one frame would make it impossible to add or remove individual Links/Sessions without renegotiating the whole world.
2. **Different layers, different scopes.** `max-frame-size` is a Connection property (one buffer per socket). `target-queue` is a Link property (one per pipe). Mixing them into one frame would force every Link change to redo Connection-level negotiation.
3. **Mirrored confirmation only works per-layer.** The broker confirms its own version of each object. A monolithic frame would have to bundle confirmations too, which produces unreadable and brittle wire formats.

The layered handshake is the natural shape of the layered model.

## Responsibilities

The handshake owns:

- **Auth** — completed during OPEN (or before, via SASL frames if TLS+SASL is in use).
- **Version negotiation** — both ends declare their AMQP protocol version; mismatched versions abort the Connection cleanly.
- **Buffer-size agreement** — `max-frame-size` is the smaller of the two proposals.
- **Window sizing** — Session windows are negotiated in BEGIN.
- **Settlement-mode agreement** — `snd-settle-mode` and `rcv-settle-mode` are negotiated in ATTACH.
- **Initial credit grant** — the broker's first FLOW frame is what actually unblocks the producer to send TRANSFER.

What it does *not* own:

- Routing decisions (target queue routing logic is the broker's, not the handshake's).
- Durability (disk-write happens later, on TRANSFER).
- Anything per-message (payload framing, body structure — those live in [[Multi-Frame Messages]] and Section 7).

## The full sequence — first message

The "first message" path. Producer has just opened a TCP socket to the broker. The producer wants to send one message to `orders-queue`.

```
Producer                                         Broker
   │                                                │
   │── 1. OPEN (channel=0)                       ──►│
   │◄─                       OPEN (channel=0)       │   ◄ Connection ready
   │                                                │
   │── 2. BEGIN (channel=1)                      ──►│
   │◄─                       BEGIN (channel=1)      │   ◄ Session ready
   │                                                │
   │── 3. ATTACH (channel=1, handle=0,           ──►│
   │              role=sender, target='orders-queue')
   │◄─                       ATTACH (channel=1,     │   ◄ Link ready
   │                          handle=0, role=receiver)
   │                                                │
   │◄─                       FLOW (channel=1,       │   ◄ "you have N credits"
   │                          handle=0, credit=100)
   │                                                │
   │── 4. TRANSFER (channel=1, handle=0,         ──►│   ◄ message bytes finally fly
   │              delivery-id=0, body=...)          │
   │                                              ┌─┴─┐
   │                                              │disk│  ◄ broker writes to queue
   │                                              └─┬─┘
   │                                                │
   │◄─                       DISPOSITION            │   ◄ "delivery-id=0 accepted"
   │                          (first=0,             │
   │                           state=accepted)
```

**Frame count:**
- Producer → broker: 4 frames (OPEN, BEGIN, ATTACH, TRANSFER)
- Broker → producer: 5 frames (mirrored OPEN/BEGIN/ATTACH, initial FLOW, DISPOSITION)
- **Total for first message: 9 frames**

## Channel 0 is the front-desk intercom

Every frame has a `channel` field in its 8-byte header. For Connection-level frames, that field is **always 0**.

| Channel | Purpose |
|---|---|
| **0** | Connection control — only OPEN, CLOSE, and Connection-wide errors travel here |
| **1, 2, 3, …** | Sessions (created via BEGIN, capped by `channel-max` from OPEN) |

Channel 0 is **not a Session**. It's a reserved slot for "talk about the Connection itself." This means an OPEN frame still has the same fixed-shape header as every other frame (parsers don't need a special case), but its `channel=0` says *"this is not for a Session, this is Connection-level."*

You can think of channel 0 as the broker's front-desk intercom — distinct from the meeting rooms (channels 1+) but reachable through the same building.

## Mirrored frames are negotiation, not acknowledgement

This is the most important non-obvious thing in the handshake.

The broker doesn't reply to `OPEN` with a generic ack. It replies with its own `OPEN` frame containing **its** chosen parameters. Each side proposes; each side confirms with what *it* will actually use:

```
Producer:  OPEN (max-frame-size=131072, heartbeat=30s, channel-max=512, ...)
Broker:    OPEN (max-frame-size=65536,  heartbeat=60s, channel-max=256, ...)
                       ↑ broker disagreed; final value is the smaller / more restrictive one
```

The same pattern applies to BEGIN (window sizes, next-outgoing-id) and ATTACH (settlement modes, `source`/`target` verification).

A generic ack couldn't carry these — it would only say "yes, I heard you" — and now the two sides could disagree silently on `max-frame-size`. The producer would happily send a 130 KB frame and the broker, who promised it would only handle 64 KB frames, would close the Connection with a `framing-error`. Without the mirrored confirmation, every cross-vendor implementation would be a debugging nightmare.

> **Mirrored confirmation = both sides are now in sync about the *parameters* of the object, not just its existence.** This is what makes AMQP work across implementations from different vendors — the negotiation forces explicit agreement on every knob.

## A freshly-attached Link starts with 0 credits

Look at the sequence again. After ATTACH succeeds, the producer cannot immediately send a TRANSFER. The receiver must first send a `FLOW` frame granting initial credits.

> **A freshly-attached Link has credit = 0. No credits = no TRANSFER. The producer must wait for the broker's first FLOW frame before sending message #1.**

In practice the broker sends FLOW *immediately* after the ATTACH-confirm, so this round-trip is invisible to most code. But if you ever see your producer hanging right after a "Link attached" log line and *before* the first message, this is exactly the symptom: the broker never sent (or your client never processed) the initial FLOW. The kind of thing only an AMQP trace will show you.

## Setup amortises — second message is just 2 frames

Once OPEN, BEGIN, and ATTACH have completed, sending the *next* 999 messages is just:

```
Producer ──► TRANSFER (channel=1, handle=0, delivery-id=N, body=...) ──► Broker
Producer ◄── DISPOSITION (delivery-id=N, accepted)              ◄── Broker
```

**Two frames per message** after setup.

For a producer running for 1 hour and sending 10,000 messages:

| Frame type | Count over 1 hour |
|---|---|
| `OPEN` | **1** — at process startup |
| `BEGIN` | **1** (or a handful, if the app uses multiple Sessions) |
| `ATTACH` | **1 per Link** — typically 1–3 total |
| `FLOW` | **periodic** — every time the broker tops up credits |
| `TRANSFER` | **10,000** (one per message) |
| `DISPOSITION` | **10,000** (one per accepted delivery) |
| Heartbeats | **~120** (every ~30s) |

ATTACH is **0.01%** of the wire traffic over the producer's lifetime. The 4-frame setup is a one-time cost amortised across the entire process.

This is why production code **never opens a Connection per message**. The Service Bus SDK keeps one Connection alive for the life of the process and reuses it. Opening per-message would multiply your wire traffic by 4-5× for zero benefit, *and* would burn TCP slow-start every time.

## Real-world example

A Service Bus producer at startup, sending its first order event:

```python
# Application code:
async with ServiceBusClient(connection_string) as client:
    sender = client.get_queue_sender("orders-queue")
    await sender.send_messages(ServiceBusMessage("order #4721 placed"))
```

What happens on the wire:

```
[startup]
TCP handshake (SYN, SYN-ACK, ACK)             ← Section 2
TLS handshake (Client Hello, etc.)            ← if TLS

[the AMQP handshake — Section 5]
1. OPEN          ─── auth, max-frame-size = 64KB negotiated
2. BEGIN         ─── Session #1 opened on channel 1
3. ATTACH        ─── sender Link, handle=0, target=orders-queue, snd-settle-mode=unsettled
   FLOW          ◄── broker grants 100 credits

[the actual message]
4. TRANSFER      ─── delivery-id=0, body="order #4721 placed"
   DISPOSITION   ◄── delivery-id=0, state=accepted    ← message is now durable in queue

[application code's send() returns success]
```

Subsequent `sender.send_messages(...)` calls only emit TRANSFER + DISPOSITION. The first send paid the setup cost; everyone after rides for free.

## Mental model

> **The handshake is a stack of locked doors, each with its own key, each opened in order.** TCP gives you the building. OPEN unlocks the front door (Connection). BEGIN unlocks a meeting room (Session). ATTACH unlocks a chute leading to a specific outbox (Link). FLOW says "this many letters at a time, please." Only then can you drop your first letter (TRANSFER) into the chute.
>
> Each door, when unlocked, stays unlocked for as long as you're using the room — so the second letter, third letter, and ten-thousandth letter all pass through doors that are already open.

## Interview answer

The AMQP handshake is a layered choreography that brings a TCP socket up to a state where the first message can flow. The producer sends OPEN (Connection — auth, version, max-frame-size, heartbeat), then BEGIN (Session — window sizes, delivery-id space), then ATTACH (Link — direction, target, settlement modes). Each frame is mirrored back by the broker with *its* chosen parameters, and the smaller / more restrictive value wins for negotiated knobs like max-frame-size. After ATTACH, the broker grants initial credits via a FLOW frame; only then can the producer send TRANSFER. So the first message takes about 9 frames; every subsequent message is just TRANSFER + DISPOSITION (2 frames). Channel 0 is reserved for Connection-level frames; channels 1+ identify Sessions. The whole pattern means setup amortises across thousands of messages — production code keeps a Connection alive for the process lifetime, never per-message.

## Common misconceptions

- **"Channel 0 is the first Session."** No. Channel 0 is reserved for Connection-level frames (OPEN, CLOSE). Sessions start at channel 1.
- **"OPEN/BEGIN/ATTACH replies are just acks."** No. They carry the broker's chosen parameters back. They're a two-way negotiation, not an acknowledgement of receipt.
- **"After ATTACH, the producer can immediately send TRANSFER."** No. A freshly-attached Link has 0 credits. The producer must wait for the receiver's first FLOW frame.
- **"`max-frame-size` is whatever the producer asked for."** No. It's the smaller of the producer's and broker's proposals — both sides converge on the more restrictive value.
- **"Each message takes 9 frames."** No. The *first* message takes ~9 because of setup. Subsequent messages are 2 frames each (TRANSFER + DISPOSITION).
- **"You should open a fresh Connection for each message for safety."** No. This multiplies wire traffic by ~5×, burns TCP slow-start, and gains nothing. Production code keeps Connections alive for the entire process lifetime.
- **"BEGIN can use any channel number, even 0."** No. Channel 0 is reserved. BEGIN must use a free non-zero channel ≤ `channel-max`.
- **"If max-frame-size is 64 KB, my message must be ≤ 64 KB."** No. `max-frame-size` is a buffer-size contract, not a message size limit. Messages bigger than that get split across multiple TRANSFER frames — see [[Multi-Frame Messages]].

## See also

- [[Connection]] — what OPEN actually negotiates
- [[Session]] — what BEGIN actually opens
- [[Link]] — what ATTACH actually opens, including the credit pool that's empty until first FLOW
- [[Frames]] — the fixed 8-byte header that every frame in this handshake shares
- [[Multi-Frame Messages]] — the next note: what happens when one TRANSFER isn't enough
- [[Settlement Modes]] — what `snd-settle-mode` / `rcv-settle-mode` in ATTACH actually mean

## Index

[[AMQP Message Transfer]]
