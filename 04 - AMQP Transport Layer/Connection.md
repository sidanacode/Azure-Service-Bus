---
tags: [amqp, transport]
---

# Connection

> **An AMQP Connection is an agreement between two AMQP-speaking programs about how to use a TCP pipe.** It is not a wire and not a socket — it is the contract that turns a dumb byte pipe into something AMQP-aware.

## Definition

An **AMQP Connection** is a logical, protocol-level agreement between two AMQP endpoints (typically a client and a broker), running on top of exactly one [[TCP Connections|TCP connection]]. It binds 1:1 to that TCP socket and adds everything TCP cannot provide: protocol version, authentication, heartbeats, max-frame-size, and a cap on how many sub-conversations (Sessions) can live inside it.

## Problem it solves

[[TCP]] gives you a reliable byte pipe — and only that. Two programs that want to do real work need answers to questions TCP can't help with:

- "Are you actually a broker, or some random server on this port?"
- "Are we both speaking AMQP 1.0?"
- "Here are my credentials — am I allowed in?"
- "If we both go silent, how do we know the other side is still alive?"
- "How big can a single chunk of data be?"

If every application invented its own answers to these, every messaging system would speak a different dialect. The Connection layer is AMQP's way of pinning these down once, at the front door, before any real work begins.

## Why previous solution was insufficient

Two earlier ideas don't work:

- **TCP alone.** TCP doesn't know what's being carried. It can't authenticate, version-negotiate, or detect a half-dead peer in any reasonable time (default keepalive is 2 hours).
- **Per-message authentication / version checks.** If you tried to authenticate every message, you would burn round trips on every send and tangle protocol negotiation with application logic. Pipe-wide concerns belong in a pipe-wide layer.

## Responsibilities

Everything the Connection layer owns is *true for the whole pipe* — not per-conversation:

- **1:1 binding to a TCP socket.** Exactly one Connection per TCP connection, exactly one TCP connection per Connection.
- **Protocol version negotiation.** First bytes on the wire after TCP handshake are the AMQP protocol header (`AMQP\x00\x01\x00\x00` for AMQP 1.0).
- **Authentication (SASL).** Done once. All Sessions inside this Connection inherit the authenticated identity.
- **Max frame size.** Every byte going through this socket obeys this limit. Negotiated in the OPEN frame.
- **Heartbeats.** A periodic empty frame in each direction. If silence exceeds the heartbeat interval, the peer is presumed dead. This solves the half-open connection problem TCP can't solve quickly.
- **Channel cap.** How many Sessions can be open inside this Connection at once.
- **Container-id.** A name for this endpoint, useful for reconnect/recovery.

What the Connection does *not* own:

- It does not hold messages. Queues hold messages.
- It does not track delivery state. [[Session]] does that.
- It does not distinguish send vs. receive flow. [[Link]] does that.

## Real-world example

A producer process starts up and wants to talk to a broker. Two handshakes happen, stacked:

```
Time   │  TCP layer (OS)              │  AMQP layer (library)
───────┼──────────────────────────────┼──────────────────────────────
 t0    │  open socket                 │  (nothing yet)
 t1    │  SYN  ──►                    │
 t2    │       ◄── SYN-ACK            │
 t3    │  ACK ──►                     │
 t4    │  ✅ TCP pipe open             │  (still nothing)
 t5    │                              │  Producer: "AMQP\x00\x01\x00\x00" ──►
 t6    │                              │  Broker:   ◄── "AMQP\x00\x01\x00\x00"
 t7    │                              │  SASL exchange (creds checked)
 t8    │                              │  Producer: OPEN frame ──►   (max-frame-size, heartbeat, channel-max)
 t9    │                              │  Broker:   ◄── OPEN frame
 t10   │                              │  ✅ AMQP Connection open
```

Between t4 and t10, real bytes are flowing on the TCP pipe — but those bytes are negotiation, not messages. The Connection only *exists* once both sides have exchanged OPEN frames and locked in the contract.

After t10, both sides hold roughly the same record in memory:

| Item | Example |
|---|---|
| Protocol version | AMQP 1.0 |
| Authenticated identity | `producer-app-7` |
| Max frame size | 65536 bytes |
| Channel max (max Sessions) | 256 |
| Heartbeat interval | 30 seconds |
| Container-id | `producer-host-pid-uuid` |

## Mental model

> **The Connection is the front door to the broker.** You knock once (TCP), show ID once (auth), agree on a language once (version), and check in periodically to prove you're still there (heartbeat). Everything you do inside the building rides on this one front-door agreement.

If the front door breaks — TCP dies, heartbeat lapses, peer crashes — everything inside collapses with it. But you don't show ID at every doorway inside; only at the front.

Or: TCP connection = the phone line; AMQP Connection = "we picked up and agreed to talk."

## The half-open problem and why heartbeats had to exist

If a peer's machine loses power, the TCP socket on the other side stays *open from the OS's point of view* — there is no FIN, no RST, no signal at all. The dead peer's OS isn't around to send one. The surviving side stares at a socket that looks fine, with a Connection contract in memory that says "the peer is connected and ready."

TCP keepalive exists but defaults to ~2 hours, which is useless for messaging. So AMQP added its own heartbeat at the Connection layer:

```
every <heartbeat-interval> seconds:
   if I have nothing to send: send an empty heartbeat frame
   if I haven't received anything from peer: assume peer is dead, tear down
```

Heartbeats live at the Connection layer because liveness applies to the **whole pipe**. If the pipe is dead, every Session and Link inside it is dead too — there is no point detecting it once per conversation.

## Interview answer

An AMQP Connection is a logical, protocol-level agreement between two AMQP endpoints riding on top of exactly one TCP connection. TCP only delivers bytes; the Connection layer adds everything pipe-wide that AMQP needs but TCP doesn't provide — protocol version negotiation, SASL authentication, max-frame-size, heartbeats for liveness, and a cap on how many Sessions can live inside. It opens via two stacked handshakes: TCP first, then an AMQP protocol header exchange and an OPEN frame exchange. Auth and version are settled once at this layer because they're true for the whole pipe; per-conversation concerns like sequence numbers, flow control windows, and send/receive direction live in the Session and Link layers above.

## Common misconceptions

- **"AMQP Connection = TCP connection."** No. They are two different things at two different layers. TCP gives you a byte pipe; AMQP Connection is the agreement that runs on top of it. They are 1:1 — never one without the other — but they are not the same.
- **"You should open one TCP connection per conversation."** No. Opening TCP connections is expensive (round trips, kernel state, broker state, NAT entries). One Connection, many Sessions inside it, is the design.
- **"TCP keepalive is enough — heartbeats are redundant."** No. TCP keepalive defaults to ~2 hours and is OS-level. AMQP heartbeats run at protocol level on a tens-of-seconds cadence so a dead peer is detected fast.
- **"Authentication should be per-message for security."** No. Auth is a pipe-wide concern. The Connection authenticates once; everything riding on this Connection inherits that identity. Per-message auth would burn round trips and confuse protocol layering.
- **"Closing a Session closes the Connection."** No. Sessions are independent units inside a Connection. You can open and close many Sessions over the lifetime of one Connection.

## See also

- [[TCP]] — the byte transport underneath
- [[TCP Connections]] — the OS-level handshake the Connection rides on
- [[Frames]] — the chunks AMQP writes to the socket once the Connection is open
- [[Session]] — the conversation contexts that live inside a Connection
- [[AMQP vs TCP]] — the layering relationship
- [[Why AMQP Exists]] — what TCP could not provide

## Index

[[AMQP Transport Layer]]
