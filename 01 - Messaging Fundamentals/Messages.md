---
tags: [fundamentals]
---

# Messages

> **A message is a self-contained unit of information exchanged between systems, carrying data and metadata.**

## Interview-ready answer

A message is a self-contained unit of information exchanged between systems, carrying both data and metadata. The body holds the actual content, usually JSON, while the headers hold facts about the message such as its ID, timestamp, and type. Each message is independent — it carries everything the consumer needs to act on it without depending on any other message. This independence is what allows messaging systems to scale.

## Why we need messages

Two programs can only push bytes through a wire.

But bytes alone are useless. `01001000 01101001` is just numbers.

The receiver can't do anything with raw bytes unless it knows:

- Where do these bytes start and end?
- What do they mean?
- What format are they in?

So we need a **package** of bytes that both sides agree on.

That package is a message.

## What is a message?

A message is a chunk of bytes with structure.

Think of it like a small file. A file you never save — you just send it over the wire.

It has two parts:

```
┌─────────────────────────────┐
│   Headers (metadata)        │  ← info ABOUT the message
├─────────────────────────────┤
│                             │
│   Body (payload)            │  ← the actual content
│                             │
└─────────────────────────────┘
```

- **Body** — the real content. Usually JSON.
- **Headers** — info about the message. ID, timestamp, type, etc.

A message is **headers + body**, packed as bytes.

## A real message

```python
ServiceBusMessage(
    body='{"orderId": 123, "amount": 99}',
    content_type="application/json",
    message_id="msg-abc-001",
    subject="OrderPlaced"
)
```

| Part | Meaning | Example |
|------|---------|---------|
| `body` | the content | `{"orderId": 123}` |
| `content_type` | format of the body | `application/json` |
| `message_id` | unique ID | `msg-abc-001` |
| `subject` | type of message | `OrderPlaced` |

The SDK turns this object into bytes and pushes them over TCP.
The broker reads the bytes back into a message.

## Body vs Headers

> The **body** says *what happened*.
>
> The **headers** say *facts about it*.

A message is **not**:
- A function call
- A database row
- A permanent thing — it lives only until consumed

A message **is**:
- A small bundle of bytes
- Body + headers
- Travels [[Producer]] → broker → consumer

## Messages are independent

Each message stands on its own.

It carries everything the consumer needs. The consumer reads one message without needing any other.

This is why messaging systems can scale.

If messages depended on each other, you'd need ordering and coordination. Independent messages keep things simple.

> Service Bus does support ordered messages when you need them. That's called **Sessions**, covered later. But the default is: each message stands alone.

## Mental model

**A sealed envelope.**

The **body** is the letter inside — the actual content. The **headers** are everything written on the outside — sender, recipient, return address, "fragile" sticker, postage stamp. The envelope travels as one sealed unit. The post office reads the outside to route it; the recipient opens it to act on the inside. A message is exactly the same shape: routing and metadata on the outside, payload sealed in.

## Common misconceptions

- **"The body is the message."** No. The message is **headers + body** as one unit. Strip the headers and the broker can't route it; strip the body and there's nothing to act on.
- **"Messages are permanent."** No. A message lives only until consumed. After the consumer's ack, the broker deletes it. Unlike a database row, it isn't a long-term record.
- **"Messages are like function calls."** No. A function call is synchronous — caller waits for return value. A message is fire-and-forget — the producer doesn't wait for processing or a return.

## See also

- [[The Raw Substrate]]
- [[Producer]]
- [[Broker]]

## Index

[[Messaging Fundamentals]]
