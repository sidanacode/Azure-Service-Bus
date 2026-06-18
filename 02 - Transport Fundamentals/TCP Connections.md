---
tags: [transport, fundamentals]
---

# TCP Connections

> A **TCP connection** is an agreed-upon, live conversation between two programs — established by a three-way handshake, identified by a 4-tuple `(source IP, source port, dest IP, dest port)`, and held as bookkeeping in RAM on both sides.

## Interview-ready answer

A TCP connection is the shared state two programs hold so that reliable byte transport has somewhere to attach. Before any bytes flow, both sides perform a three-way handshake — SYN, SYN-ACK, ACK — to confirm they're alive, willing, and agreed on starting sequence numbers. From that moment on, the connection is a 4-tuple of source IP, source port, destination IP, destination port, plus state in each side's OS memory. Connections are not free: the handshake costs a full round-trip before any data moves, and idle connections still consume memory. This is why messaging clients always pool and reuse connections instead of opening one per message — and why protocols like AMQP multiplex many logical conversations over a single TCP connection. Connections die in RAM when a process crashes, and the other side doesn't notice immediately, which is why heartbeats exist at higher layers.

## Why does it exist?

[[Reliable Byte Transport]] gives us strong guarantees — but those guarantees need somewhere to live. Sequence numbers, ack state, retransmit timers — they all need a stable record on each side that says *"I'm in a conversation with that program, and here's where we are in it."*

Without that record, "just start sending bytes" fails three ways:

1. **No one's listening.** The receiver's process may not exist on that port. Bytes get dropped silently.
2. **Wrong program answers.** Some other program (SSH, a database) is bound to that port and gets garbage it can't handle.
3. **No mutual agreement.** Bytes from the same sender can't be tied together as one conversation — they look like random unrelated packets.

A connection is the agreement that fixes all three before any real bytes flow.

## What is it really?

A connection is **state**, not a wire. There is no physical line between the two computers. Each side just holds a record in OS memory:

- The 4-tuple identifying the peer
- The current sequence number (what byte am I up to?)
- The ack window (what has the peer confirmed?)
- Retransmit buffers (bytes I've sent but not seen acked yet)

The "connection" is that shared bookkeeping. Crash the process, lose the bookkeeping, the connection is gone — even though no wire was ever cut.

## The three-way handshake

```
Client (web server)            Server (broker)
       │                              │
       │  ─── SYN ────►                │   "Hi, I want to talk.
       │     (seq=100)                 │    My starting seq is 100."
       │                              │
       │  ◄─── SYN-ACK ───             │   "Got it. I'm willing.
       │   (seq=500, ack=101)          │    My starting seq is 500.
       │                              │    I expect your next byte at 101."
       │                              │
       │  ─── ACK ────►                │   "Confirmed. Expecting
       │     (ack=501)                 │    your byte at 501."
       │                              │
       │  ===== connection open =====  │
```

Three messages. After the third, both sides have:

- Confirmed the other is alive
- Confirmed the other is willing
- Agreed on starting sequence numbers (so [[Reliable Byte Transport]] can do its job)
- Recorded the connection in OS memory

Only **then** do real bytes flow. The handshake costs a full round-trip — on a 100ms link, that's 100ms of pure waiting before your first byte of payload.

## Why connections are expensive

Two costs, both real:

1. **Setup latency.** A round-trip before any data. Cheap on a LAN, painful across regions.
2. **Memory.** Both sides hold connection state in RAM. Idle connections still cost memory. At 1000 connections/sec, the OS tables fill up faster than they drain.

This is why every messaging client — Service Bus SDK, Kafka client, RabbitMQ client — opens **one** connection at startup and reuses it for every message. The handshake is paid once. Every send after that is cheap.

It's also the foundation of AMQP's design: open one TCP connection, then multiplex thousands of logical conversations (sessions, links) on top of it. The TCP cost is amortized across the entire process lifetime.

## Connections die silently

When the broker process crashes, **the web server doesn't immediately know.**

The connection was just bookkeeping in RAM. The broker's RAM is gone, but the web server's half is still there, happily believing the conversation is alive. The web server only finds out when:

- It tries to send bytes and no ack ever comes back, or
- A keepalive timer expires (TCP defaults are *long* — sometimes hours)

This delay is why higher layers add their own heartbeats. AMQP sends periodic heartbeat frames so dead peers are detected in seconds, not hours. Without them, you'd be sending `OrderPlaced` messages into the void for a very long time before anyone noticed.

## Clean teardown

When a connection ends normally, both sides do a four-step **FIN handshake**:

```
Client                  Server
   │                       │
   │  ─── FIN ────►         │   "I'm done sending."
   │  ◄──── ACK ───         │
   │  ◄─── FIN ───          │   "I'm done too."
   │  ─── ACK ────►         │
   │                       │
   │   ===== closed =====   │
```

Both sides free their bookkeeping. The 4-tuple is released and can be reused for a new connection.

When teardown *doesn't* happen cleanly (process crash, network partition), the connection becomes a "half-open" zombie until a timer eventually reaps it. Useful to know, not required.

## Common misconceptions

- **"A TCP connection is a wire."** No. It's state in RAM on each side. No wire is allocated, no path is reserved. Packets find their own way through the network each time.
- **"If the other side crashes, my side gets notified."** No. You find out only when a send fails or a long timer expires. Detecting dead peers fast requires heartbeats at a higher layer.
- **"Reusing a connection is just a performance trick."** It's more than that — at scale, opening a connection per message will exhaust OS resources and add round-trips that dominate latency.

## A real example

Your web server starts up:

```
1. Web server starts.
2. SDK opens ONE TCP connection to broker:port 5671.
   - SYN, SYN-ACK, ACK    (~one round-trip)
   - Connection open. State recorded in RAM on both sides.
3. User #1 places an order.
   - send_message(OrderPlaced{orderId: 1})
   - Bytes flow over the existing connection. No handshake.
4. User #2 places an order. (1 second later)
   - Same connection. No handshake.
5. User #500 places an order. (3 hours later)
   - Same connection. No handshake.
6. Web server shuts down.
   - FIN handshake. State freed on both sides.
```

The handshake cost is paid **once at startup**. Every message after that is bytes-only. If the SDK opened a fresh connection per message, the web server would spend more time handshaking than actually sending data — and the broker would run out of memory tracking thousands of short-lived connections.

## Mental model

**A phone call.**

You don't shout into your phone hoping the other person hears — you dial, they pick up, you both say "hello." That's the handshake. Once both sides confirm, words flow freely; you don't re-dial for every sentence. When you're done, you say "bye" and both hang up — the teardown.

If the other person's phone dies mid-call, you don't immediately know. You keep talking until you notice no one's answering. That's why long calls have "you still there?" check-ins — the same role TCP keepalives and AMQP heartbeats play.

## See also

- [[TCP]]
- [[Reliable Byte Transport]]
- [[Broker]]
- [[Producer]]

## Index

[[Transport Fundamentals]]
