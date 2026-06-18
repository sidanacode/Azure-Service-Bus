---
tags: [transport, fundamentals]
---

# TCP

> **TCP** is a reliable byte transport. It sits between the application and the raw network, hides packet loss and reordering, and delivers a clean stream of bytes from one program to another.

## Interview-ready answer

TCP is the protocol that turns the unreliable internet into a reliable byte pipe. Without it, every application would have to handle packet loss, reordering, duplication, and corruption on its own. TCP does that work once, on behalf of everyone. The application just hands it bytes; TCP guarantees those bytes arrive at the other end, in order, intact, and without duplicates. What TCP does **not** give you is message boundaries, routing, intent, or application-level acks — those gaps are exactly why protocols like AMQP exist on top.

## Why does it exist?

The raw internet (IP) is a mess. Packets:

- Get **lost** — a router drops them under load
- Arrive **out of order** — different packets take different paths
- Get **duplicated** — a router resends one it thought was lost
- Get **corrupted** — a bit flips somewhere

If every program had to handle this itself, every program would reinvent the same machinery — sequence numbers, acks, retransmits, reordering buffers. Insanity.

TCP solves it **once**, in the operating system, for everyone.

## What does it actually do?

The application calls `socket.send("hello world")`. On the other side, the application calls `socket.recv()` and gets `"hello world"` — exactly, intact, in order.

Underneath, TCP did all this:

- Broke the data into packets
- Numbered each packet (sequence numbers)
- Watched for acks from the other side
- Retransmitted any unacked packet
- Reordered packets that arrived out of sequence
- Dropped duplicates
- Used checksums to catch corruption

The application sees none of it. **Bytes in → bytes out.**

## Reliable, byte, transport — three loaded words

- **Reliable** — bytes you sent will arrive, in order, intact
- **Byte** — TCP doesn't know or care what the bytes mean. To TCP, `"hello world"` is just 11 bytes. Not a message, not a command, not anything.
- **Transport** — it moves bytes from program A to program B

That's the whole job description.

## What TCP does NOT give you

This is the **important** list. Every gap here is a reason a higher-level protocol (like [[AMQP]]) exists.

### 1. Message boundaries (the big one)

TCP is a **stream of bytes**. There are no "messages."

If you send `"hello"` then `"world"`, the receiver might see:

- `"hello"` then `"world"` ✅
- `"helloworld"` (merged)
- `"hel"` then `"loworld"` (split weirdly)
- `"h"`, `"ello"`, `"wo"`, `"rld"` (spread across many reads)

TCP doesn't know there were two separate things. To TCP, it's all `helloworld`.

The application has to figure out where one message ends and the next begins. **This is why AMQP exists** — it adds *framing* on top of TCP so receivers can carve the stream back into discrete messages.

### 2. Routing / addressing

TCP connects two endpoints (IP:port to IP:port). It doesn't know about named queues, topics, or subscriptions. It can't deliver "to whoever consumes the `orders` queue." That's the broker's job, on top.

### 3. Intent

TCP doesn't know if the bytes are a [[Command]], [[Event]], [[Query]], or [[Notification]]. It's all just bytes.

### 4. Multiple conversations on one connection

If a producer wants to send to two queues at once, TCP would need *two* separate connections. Connections are expensive (handshake, memory). AMQP fixes this with **sessions and links** — many logical conversations sharing one TCP pipe.

### 5. Application-level acks

TCP has acks — but they only mean *"bytes arrived at the OS network buffer."* They do **not** mean *"the application processed them."*

Remember the two-ack model from [[Broker]]:

- TCP gives you **ack #1** (received by OS)
- TCP does **not** give you **ack #2** (processed by application)

That's the layer AMQP adds. (Settlement, in Section 6.)

### 6. Persistence

If the receiver crashes after TCP delivered the bytes but before the application saved them — bytes are gone. TCP has no memory. Persistence comes from the [[Broker]]'s disks.

### 7. Authentication / authorization

TCP doesn't ask *"who are you, and are you allowed to publish to this queue?"* It just moves bytes. AMQP and Service Bus add SASL, tokens, and RBAC on top.

## The layer stack

```
┌────────────────────────────────┐
│  Service Bus                   │  queues, topics, DLQ, sessions
├────────────────────────────────┤
│  AMQP                          │  framing, routing, sessions, settlement
├────────────────────────────────┤
│  TCP                           │  reliable, in-order bytes
├────────────────────────────────┤
│  IP / network                  │  unreliable packet delivery
└────────────────────────────────┘
```

Each layer **hides the mess of the layer below** and **adds something new**.

- TCP hides packet loss → gives you bytes
- AMQP hides "it's just bytes" → gives you messages with structure and routing
- Service Bus hides AMQP details → gives you queues, topics, dead letters

## Mental model

**A water pipe.**

TCP gives you a **clean pipe of bytes**. Water flows in one end, comes out the other end in the same order, no leaks, no debris, no air bubbles. The pipe doesn't know whether you're delivering drinking water, irrigation water, or industrial coolant. It doesn't know where one bucket of water ends and the next begins. It just delivers a clean continuous flow.

Every other concern — message structure, routing, application acks, intent, persistence — has to be built on top.

When AMQP comes up next, every feature it adds will map directly to one of the gaps in the "What TCP does NOT give you" list above. Keep that list in mind.

## Common misconceptions

- **"TCP delivers messages."** No. TCP delivers **bytes** in order. The concept of a "message" doesn't exist at the TCP layer. The receiver may need 1, 2, or 17 reads to assemble what the sender wrote — TCP makes no promises about that.
- **"TCP's ack means the receiving application got my data."** No. TCP's ack means the bytes reached the receiver's OS network buffer. The application may not have read them yet. For application-level confirmation, you need a higher protocol's ack (settlement in AMQP).
- **"TCP can lose data."** No. If TCP delivers bytes, they are intact, in order, and exactly once. If it can't deliver them, the connection breaks and the application is told. There is no silent loss — but there can be silent duplicates at the *message* level if the application retries after a broken connection.
- **"HTTPS replaces TCP."** No. HTTPS is HTTP-over-TLS-over-TCP. TCP is still doing its job underneath; TLS just encrypts the bytes flowing through it.

## See also

- [[Broker]]
- [[Delivery Semantics]]
- [[Idempotency]]

## Index

[[Transport Fundamentals]]
