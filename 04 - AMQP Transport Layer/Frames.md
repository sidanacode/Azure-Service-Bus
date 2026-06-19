---
tags: [amqp, transport]
---

# Frames

> **A frame is the smallest standalone unit AMQP puts on the wire.** Every byte AMQP sends is part of some frame. Each frame is length-prefixed and channel-tagged, which is what makes multiplexing work.

## Definition

A **frame** is a length-prefixed chunk of bytes the AMQP library produces, hands to TCP, and parses on the other side. Every frame carries a fixed-shape header that tells the receiver how big the frame is, what type it is, and **which Session (channel) it belongs to**. The body holds the actual content — an OPEN, BEGIN, ATTACH, TRANSFER, DISPOSITION, FLOW, and so on.

Frames are an AMQP-layer concept. TCP does not know about them — TCP just sees bytes.

## Problem it solves

[[TCP]] gives you an unbounded stream of bytes with no boundaries. To do anything useful AMQP needs:

1. A way to know where one chunk ends and the next begins ([[Message Boundaries]] — solved by length prefix).
2. A way to multiplex many [[Session|Sessions]] over one [[Connection]] without giving each its own TCP socket.

Frames solve both at once: the size field is the length prefix, and the channel field is the multiplexing tag.

## Why previous solution was insufficient

Without frames, every chunk written to the socket would be opaque to the receiver. You'd be back to the [[Message Boundaries|three framing problems from Section 3]]: delimiters collide on binary data, fixed-size doesn't fit variable payloads, and there's no way to interleave multiple conversations.

You'd also have nowhere to put the channel number. If you can't tag a chunk with "this belongs to Session 2," you can't share a TCP connection across multiple Sessions, and you're forced into one TCP connection per conversation — exactly the cost AMQP is trying to avoid.

## Anatomy of a frame

```
┌────────────────────────────────────────────────────┐
│            FRAME HEADER  (8 bytes)                 │
├──────────────┬──────┬──────┬───────────────────────┤
│ size (4B)    │ DOFF │ TYPE │  CHANNEL (2B)         │
├──────────────┴──────┴──────┴───────────────────────┤
│                                                    │
│            FRAME BODY  (variable size)             │
│            (the performative + payload)            │
│                                                    │
└────────────────────────────────────────────────────┘
```


| Field | Size | What it is | Why it's there |
|---|---|---|---|
| **size** | 4 bytes | Total length of THIS frame in bytes | Length prefix — receiver knows where this frame ends and the next begins |
| **DOFF** | 1 byte | "Data offset" — where the body starts | Lets headers grow in future protocol versions without breaking parsers |
| **TYPE** | 1 byte | 0 = AMQP frame, 1 = SASL frame | Distinguishes pre-auth (SASL) frames from real AMQP frames |
| **CHANNEL** | 2 bytes | The Session number | The multiplexing tag — receiver routes the frame to a Session by this number |
| **body** | variable | The performative (OPEN, BEGIN, TRANSFER, …) and any payload | What the frame is actually carrying |

The **CHANNEL** field is the whole multiplexing trick. That 2-byte number is what makes the byte smoothie un-mixable on the receiving side.

## Responsibilities

- **Carry length** so the receiver can find frame boundaries on a raw TCP byte stream.
- **Carry channel number** so the receiver can route the frame to the right [[Session]].
- **Carry a performative** in the body that says what this frame *means* — open Connection, send a message, settle a delivery, update credits, tear down.

What frames do *not* do:

- They don't track delivery state — that's the Session's job.
- They don't know about messages as units — a single message can be many frames.
- They don't know about TCP packets — that's a different layer of chopping.

## Frame ≠ TCP packet

This is the most common confusion. They look similar (chopping bytes, identifiers, reassembly) but live on different layers:

| Layer | Cuts into | Why |
|---|---|---|
| **AMQP** | **Frames** | Multiplexing — many Sessions over one pipe, each frame tagged with its channel |
| **TCP** | **Packets** | Reliable network transport — handles loss, reordering, congestion |

When the producer's library produces an 800-byte frame, it writes those 800 bytes to the TCP socket. TCP might send all 800 in one packet, or split them across two, or combine them with bytes from another frame into a single packet. **AMQP doesn't care.** It trusts TCP to deliver bytes in order; it parses frames out of the byte stream on the other side using the size prefix.

```
Application  →  message
                    ↓
AMQP library →  cuts into frames, tags each with channel number
                    ↓
TCP socket   →  ignores frame boundaries, chops into packets
                    ↓
Network      →  bytes
```

And in reverse on the receiver:

```
Network      →  bytes
                    ↓
TCP socket   →  reassembles packets back into byte stream
                    ↓
AMQP library →  parses frame boundaries via size prefix,
                routes each frame to its Session by channel number
                    ↓
Application  →  message
```

Two reassembly steps. TCP reassembles packets into a byte stream. AMQP reassembles frames out of that byte stream.

## Real-world example

A producer has three Sessions open: orders (channel 1), audit (channel 2), replies (channel 3). It wants to send one short message on each, then settle one earlier delivery. Four frames get produced:

```
Frame 1:  [size=412][DOFF][type=0][channel=1] TRANSFER + order data
Frame 2:  [size=180][DOFF][type=0][channel=2] TRANSFER + audit data
Frame 3:  [size=64 ][DOFF][type=0][channel=3] TRANSFER + reply data
Frame 4:  [size=32 ][DOFF][type=0][channel=1] DISPOSITION (settle delivery #4719)
```

The library writes ~688 bytes to the TCP socket in that order. TCP carries the bytes. On the broker side:

```
Step 1 — read 4 bytes:   size = 412
         read 408 more:  full frame, channel=1 → route to Session "orders"
Step 2 — read 4 bytes:   size = 180
         read 176 more:  full frame, channel=2 → route to Session "audit"
Step 3 — read 4 bytes:   size = 64
         read 60 more:   full frame, channel=3 → route to Session "replies"
Step 4 — read 4 bytes:   size = 32
         read 28 more:   full frame, channel=1 → route to Session "orders"
```

Multiplexing in action. Each frame is delivered to the right Session entirely on the basis of its channel number.

## Big messages → multi-frame transfers

Recall that the [[Connection]] handshake negotiated a `max-frame-size` (e.g. 65,536 bytes). If a message is bigger than that, the library splits it across **multiple TRANSFER frames**, all on the same channel, all marked as parts of one larger transfer (`more=true` on every part except the last). The receiver reassembles them back into the original message before handing it to the application.

```
5 MB message, max-frame-size = 64 KB  →  ~80 TRANSFER frames on the same channel
```

`max-frame-size` is a **buffering contract**, not a message size limit. It tells the peer "promise me no single frame will be bigger than X, so I can pre-allocate a buffer of size X and never get surprised." Message size itself is bounded only by what the broker accepts.

This is the [[Message Boundaries|length-prefix framing]] promise from Section 3, finally cashing in.

## The body — performatives

The body of a frame says what the frame *means*. AMQP defines a small fixed set of frame body types, called **performatives**. Each performative is a verb:

| Performative | Meaning |
|---|---|
| `OPEN` | Negotiate Connection settings (the Connection handshake) |
| `BEGIN` | Open a Session |
| `ATTACH` | Open a Link inside a Session |
| `TRANSFER` | Carry a message (or part of one) |
| `DISPOSITION` | Settlement — accept / reject / release / modify a delivery |
| `FLOW` | Update flow control credits or window |
| `DETACH` / `END` / `CLOSE` | Tear down a Link / Session / Connection |

Every meaningful thing AMQP does is a frame whose body is one of these performatives. The header is always the same shape; the body is what varies.

## Mental model

> **A frame is an envelope.** Outside the envelope (the header) is the addressing — how big it is, what type, **which conversation it belongs to (channel)**. Inside the envelope (the body) is the actual content.
>
> TCP is the postal service that delivers envelopes in order. AMQP is the office that opens each envelope, looks at the channel number, and drops it into the right Session's inbox.

## Interview answer

A frame is the smallest standalone unit AMQP writes to the wire. Every frame has a fixed 8-byte header — size, data-offset, type, and channel — followed by a variable-size body that carries one of AMQP's performatives like OPEN, BEGIN, ATTACH, TRANSFER, DISPOSITION, FLOW. The size field is the length prefix that lets the receiver find frame boundaries on a raw TCP stream; the channel field is the multiplexing tag that routes the frame to the right Session. AMQP frames are not TCP packets — TCP chops the byte stream into packets independently for reliable transport, while AMQP chops it into frames for multiplexing. A single message can span many TRANSFER frames if it exceeds max-frame-size, all on the same channel and marked as parts of one delivery. This is what lets one AMQP Connection carry many independent conversations over a single TCP socket.

## Common misconceptions

- **"A frame is a TCP packet."** No. They are different units at different layers. AMQP doesn't know about TCP packets; TCP doesn't know about AMQP frames. TCP can split or combine frames into packets however it wants.
- **"One message = one frame."** No. A message larger than `max-frame-size` is sent as multiple TRANSFER frames on the same channel, reassembled by the receiver.
- **"`max-frame-size` caps message size."** No. It's a buffer-size promise — "no single frame from me will exceed X." Message size is bounded by broker policy, not max-frame-size.
- **"The channel number is the message ID."** No. The channel is the *Session* tag. Messages have their own delivery-id inside the [[Session]].
- **"Frames are encrypted on the wire."** No. Frames are AMQP-level constructs. Encryption happens via TLS on the TCP layer, transparently — frames are produced/parsed in plaintext by the AMQP library.

## See also

- [[Message Boundaries]] — the length-prefix framing idea frames apply
- [[Connection]] — where `max-frame-size` is negotiated
- [[Session]] — what a channel number routes to
- [[Why AMQP Exists]] — the gap frames help close

## Index

[[AMQP Transport Layer]]
