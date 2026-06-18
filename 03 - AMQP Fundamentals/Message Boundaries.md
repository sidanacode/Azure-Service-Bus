---
tags: [amqp, fundamentals]
---

# Message Boundaries

> **TCP delivers a continuous stream of bytes. AMQP needs discrete messages. Message boundaries are the rule that turns one into the other.** Length prefix is how AMQP draws the line.

## Definition

A **message boundary** is the rule a protocol uses to mark where one message ends and the next begins on a byte stream. The three choices are **delimiter**, **length prefix**, and **fixed size**. AMQP uses length prefix.

## Problem it solves

A producer sends three messages:

```python
sender.send_message(order_1)
sender.send_message(order_2)
sender.send_message(order_3)
```

On the wire, TCP doesn't carry three things. It carries one continuous stream of bytes:

```
AAABBBCCC
```

The broker reads from its socket and sees those 9 bytes. Three messages went in. The broker sees one blob.

> **Where does message 1 end? Where does message 2 begin?**

TCP cannot answer that — TCP doesn't even know what a "message" is. The protocol on top of TCP has to define the rule. That rule is the message boundary.

This is also called the **framing problem**. AMQP solves it. HTTP solves it differently. Every protocol on top of TCP has to solve it somehow.

## Why previous solution was insufficient

[[TCP]] gives ordered, lossless byte delivery — that's its whole job. It does not group bytes into messages because it doesn't know what a message is. To TCP, your three orders and someone else's video stream and a database query all look the same: a stream of bytes.

If TCP tried to add message awareness, it would also have to learn web pages, SQL, video formats, and every other shape of data anyone might send. TCP stays small and generic on purpose. Message boundaries are an *application* concern, layered on top.

## Responsibilities

The boundary rule is responsible for letting the receiver answer one question, deterministically and cheaply: *"how many bytes belong to this message?"*

It must work for:

- **Any payload type** the protocol carries (text, binary, encrypted)
- **Any size** within the protocol's limits (200 bytes or 200 MB)
- **Any volume** without slowing the receiver down

The three schemes give three different answers.

### Scheme 1 — Delimiter

Pick a byte (or sequence) that means *"message ends here."*

```
{"orderId":1}|{"orderId":2}|{"orderId":3}|
```

The receiver scans bytes until it sees `|`, takes everything before as one message, repeats.

**Used by:** HTTP/1.1 headers (`\r\n`), SMTP, Redis (RESP).

**Why it can fail:** if the message body itself contains the delimiter, parsing breaks. Either pick a delimiter unlikely to appear (works for short text), or *escape* the delimiter when it appears in data (slow, bug-prone).

For binary payloads, **every byte value is valid data** — there is no safe delimiter to pick. Delimiters die on binary.

### Scheme 2 — Length prefix

Before each message, write a fixed-size number saying how big the message is.

```
┌────┬───────────────┬────┬───────────────┬────┬───────────────┐
│ 14 │ {"orderId":1} │ 14 │ {"orderId":2} │ 14 │ {"orderId":3} │
└────┴───────────────┴────┴───────────────┴────┴───────────────┘
```

The receiver:

1. Reads 4 bytes → that's the size, say 14.
2. Reads exactly 14 more bytes → that's the message.
3. Repeats.

**Used by:** AMQP, HTTP/2, gRPC, MQTT, Kafka.

**Why it wins for binary:** the receiver never inspects body bytes for boundary purposes — it just counts. Binary, text, encrypted gibberish — the body is opaque. No escaping, no scanning.

**Cost:** ~4 bytes overhead per message. Negligible at any reasonable message size.

### Scheme 3 — Fixed size

Every message is exactly N bytes. No marker needed — just count.

```
┌────────────────┬────────────────┬────────────────┐
│   16 bytes     │   16 bytes     │   16 bytes     │
└────────────────┴────────────────┴────────────────┘
```

**Used by:** sensor protocols, hardware register reads, GPS coordinate streams, old telegraph systems.

**Why it rarely fits:** real-world messages vary in size. An order can be 200 bytes or 8 KB. Padding small messages to fit the largest is wasteful; splitting large messages requires inventing a sub-framing scheme on top — at which point you've reinvented length prefix, badly.

## Why AMQP picked length prefix

AMQP messages can be:

- **Binary or text** — bodies are arbitrary bytes
- **Variable size** — from 200 bytes to many MB

Run those constraints through the schemes:

| Scheme | Survives binary? | Survives variable size? |
|---|---|---|
| Delimiter | ❌ collisions | ✅ |
| Length prefix | ✅ | ✅ |
| Fixed size | ✅ | ❌ wasteful |

Only length prefix survives both. AMQP didn't pick it because it's "best in general" — it picked it because the other two get ruled out automatically.

## Real-world example

### HTTP shows both schemes at once

When a browser uploads an image, HTTP uses **delimiters for headers** and **length prefix for the body**:

```
POST /upload HTTP/1.1\r\n          ← line 1, ends with \r\n (delimiter)
Host: api.example.com\r\n          ← line 2
Content-Type: image/jpeg\r\n       ← line 3
Content-Length: 8472\r\n           ← line 4 — the length prefix!
\r\n                               ← BLANK line = "headers done"
[8472 bytes of JPEG data]          ← body, exactly that many bytes
```

The server's logic:

1. Read bytes line-by-line, looking for `\r\n` between lines.
2. Hit `\r\n\r\n` (blank line) → headers are done.
3. Among the headers it just read, find `Content-Length: 8472`.
4. Read exactly 8472 more bytes → that's the body.

Each scheme is used where it fits best — delimiter for predictable text, length prefix for arbitrary bytes.

If the client promises 8472 bytes but only sends 8000, the server **waits or times out**. Length prefix is a promise the receiver holds the sender to.

### AMQP just uses length prefix everywhere

```
┌────────┬────────────────┬──────────────────────┐
│  size  │     header     │        body          │
│ 4 bytes│  routing info  │   your payload       │
└────────┴────────────────┴──────────────────────┘
```

The broker reads 4 bytes, knows the total frame size, reads exactly that many more, decodes it. No scanning, no escaping, no special blank-line rule.

## Protocol comparison

The same problem — *"draw boundaries on a byte stream"* — solved different ways for different shapes of data:

| Protocol | Built for | Picks | Why |
|---|---|---|---|
| **HTTP/1.1** | Web request/response | Hybrid: `\r\n` for headers, length prefix for body | Headers are short text; body can be anything |
| **HTTP/2** | Multiplexed web (many streams on one connection) | Pure length prefix | Multiplexing kills delimiters — see below |
| **AMQP** | Async messaging between services | Pure length prefix | Binary-native, variable-size payloads |
| **gRPC** | Service-to-service RPC | Length prefix on top of HTTP/2 | Inherits HTTP/2's framing |
| **Redis (RESP)** | Cache commands | Hybrid: `\r\n` with length hints | Designed for telnet-debugging *and* binary values |
| **MQTT** | IoT pub-sub | Pure length prefix | Tiny devices need minimal overhead |
| **SMTP** | Email | Pure delimiters (`\r\n`, `.\r\n` ends body) | Predates binary thinking; email was text-only |

Two clusters:

- **Text-friendly + human-debuggable** → delimiters. You can `telnet` in and type commands.
- **Binary-safe + high-throughput** → length prefix. Faster to parse, no escaping, works for any payload.

### Why HTTP/2 abandoned delimiters

HTTP/1.1 sends one request at a time per connection. HTTP/2 sends **many requests at once**, with their bytes interleaved on the same wire:

```
─[req-1-part-A]─[req-2-part-A]─[req-1-part-B]─[req-3-part-A]─
```

Delimiters fundamentally can't handle this — there's no way to tell which `\r\n` belongs to which request when streams are mixed. Length prefix survives because each chunk says *"I am N bytes and I belong to stream #3"*. The receiver routes chunks to the right buffer.

This is the same reason AMQP picked length prefix from day one — AMQP also multiplexes many sessions over one TCP connection.

### Why SMTP survives with delimiters

Email was originally plain text — letters typed by humans. Almost zero chance of `\r\n.\r\n` (the body-end marker) appearing inside a real email. No collision, no escaping, no problem.

For binary attachments today, SMTP cheats: it **base64-encodes** the binary into safe text first (`A-Z`, `a-z`, `0-9`, `+`, `/`). The result is ~33% larger, but it's now guaranteed text — so the delimiter scheme still works. This is why email attachments are bigger than the original file.

When you hear *"AMQP is binary-native"*, this is what it avoids. AMQP puts raw bytes in the body — no base64, no escaping. Length prefix doesn't care.

## Mental model

**Three ways to mark where sentences end.**

- **Delimiter** is the **fullstop**. Most of the time it works — but if your sentence happens to contain a "." (like a URL or a number), the reader gets confused, and you have to escape it.
- **Length prefix** is **labeled boxes**. Each box has *"contains 14 items"* written on the outside. The reader counts 14, takes them out, moves on. Doesn't matter what's inside.
- **Fixed size** is **identical envelopes**. Every envelope is exactly the same size. You don't need labels — just count envelopes. But everything you send has to fit the same envelope.

AMQP runs a postal system that handles any shape of cargo. Labeled boxes are the only choice that scales.

## Interview answer

Message boundaries are the rule that turns TCP's continuous byte stream into discrete messages. Three schemes exist: delimiter (a special byte ends each message), length prefix (a size field precedes each message), and fixed size (every message is the same length). AMQP uses length prefix because its payloads are binary and variable-size — delimiters die on binary collisions, fixed size wastes space on variable data, only length prefix survives both. HTTP/1.1 uses a hybrid (delimiters for text headers, length prefix for the body), but HTTP/2 dropped delimiters entirely because they can't survive multiplexed streams. The general pattern: text-friendly protocols pick delimiters for human readability; binary-safe high-throughput protocols pick length prefix.

## Common misconceptions

- **"TCP gives you messages."** No. TCP gives you bytes. Three `send()` calls turn into one byte stream. The boundary rule on top of TCP is what reconstructs messages.
- **"Length prefix is just a performance optimization."** No. It is a correctness mechanism. Without an unambiguous size, the receiver has to scan and escape, which is both slower *and* more bug-prone.
- **"Delimiters are obsolete."** No. They are perfect for short, text-only protocols where humans need to debug by typing commands. SMTP, HTTP headers, Redis still use them and aren't going away.
- **"AMQP frames are messages."** Almost — but not quite. One AMQP message can be split across multiple frames if it's large. Frames are the *transport unit*; messages are the *application unit*. Covered in the AMQP transport section.
- **"You could swap length prefix for delimiters in AMQP."** Not without breaking binary support. Every byte value is valid in an AMQP body — pick any delimiter and some payload will eventually contain it, leading to silent corruption.

## Design tradeoff

**Why length prefix:** it's the only scheme that handles arbitrary bytes at any size. The 4-byte overhead per frame is invisible at typical message sizes. Parsing is just *read-size-then-read-N-bytes* — fast, predictable, no scanning.

**The advantage:** AMQP can carry anything in a body — JSON, binary blobs, encrypted data, compressed data — with no encoding step. The framing layer never inspects body bytes for boundary purposes.

**The cost:**

1. **Not human-readable on the wire.** You can't `telnet` to an AMQP broker and type commands. Debugging requires a protocol-aware tool.
2. **Receiver must trust the size.** A malicious or buggy sender that lies about the size can stall the connection (waiting for bytes that never come) or confuse the parser. Real implementations cap maximum frame size to defend against this.
3. **Streaming is harder.** With delimiters, you can start processing as soon as you see the end marker. With length prefix, you have to know the size up front — which means buffering large messages in memory before sending. AMQP softens this with multi-frame messages, but the constraint is real.

## See also

- [[TCP]] — what AMQP runs on top of
- [[Reliable Byte Transport]] — TCP's actual promise
- [[Why AMQP Exists]] — the safety gap above TCP
- [[AMQP vs TCP]] — the layering relationship
- [[Messaging Semantics]] — how AMQP closes the safety gap (settlement)

## Index

[[AMQP Fundamentals]]
