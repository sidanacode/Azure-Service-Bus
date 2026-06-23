---
tags:
  - amqp
  - message-structure
---

# Body

> **Body is the message's actual payload** — the bytes carrying the order JSON, the protobuf event, the encrypted blob. Everything else in the message structure is metadata *about* the body. AMQP defines three body shapes (`data`, `amqp-value`, `amqp-sequence`); Service Bus uses `data` exclusively. The broker treats the body as opaque — it never decodes it, never filters on it, never reads inside it.

## Definition

**Body** is the 6th of an AMQP message's six sections — the section carrying the application payload. It is the only section that's logically required (the others are all optional). AMQP defines three possible shapes for the body:

| Shape | Type | Description |
|---|---|---|
| **`data`** | One or more binary chunks | Opaque bytes; the application interprets the format |
| **`amqp-value`** | A single typed AMQP value | Strongly typed; sender and receiver share the AMQP type system |
| **`amqp-sequence`** | A list of typed AMQP values | Streaming structured records |

A body uses **exactly one** of these three shapes — they're mutually exclusive. Service Bus uses `data`.

## Problem it solves

A message system is useless without a place for the actual user-facing data. Body is that place. The interesting design decision is **how strongly typed should the body be?**

Different messaging systems answer this differently:

- *"The body is bytes; the application decides what they mean."* — Most systems (Kafka, RabbitMQ classic, SQS).
- *"The body is a typed structure the protocol can introspect."* — Typed RPC frameworks (some IBM MQ scenarios).
- *"The body is a sequence of typed records."* — Streaming-oriented systems.

AMQP didn't pick one. It defined three shapes and lets the application choose at message-construction time.

## Why previous solution was insufficient

If AMQP had forced "body is bytes":

- Strongly-typed messaging systems (where every field has a known type and the protocol can read it without an app-level codec) couldn't be transported over AMQP without losing their type information.

If AMQP had forced "body is a typed value":

- Every byte-oriented producer (which is almost all of them — JSON, protobuf, encrypted blobs) would have to wrap their payload in a string or binary AMQP value, adding overhead for no benefit.

The three-shape design is the compromise: **opt into typing if your system needs it; fall back to opaque bytes if it doesn't.** Real-world distribution is roughly 99% `data`, ~1% `amqp-value`, near-zero `amqp-sequence`.

## Responsibilities

Body owns:

- **The application payload** — whatever bytes the producer wants to deliver to the consumer.
- **The shape choice** — `data` vs `amqp-value` vs `amqp-sequence` is set per message.

What Body does NOT own:

- **Metadata** — that's [[Header]], [[Properties]], [[Application Properties]], [[Message Annotations]].
- **Filtering** — Service Bus filters can't read body content. Anything that should drive routing must be in [[Application Properties]] or Properties.
- **Format declaration** — the MIME type goes in `Properties.content-type`. The body itself is just bytes (or a typed value).

## The three body shapes

### `data` — opaque bytes (Service Bus default)

```
body = data { 0x7B, 0x22, 0x6F, 0x72, 0x64, ... }   // raw bytes of {"order":...}
```

The body is one or more byte chunks. AMQP doesn't interpret them. The consumer reads `Properties.content-type` to know the format (JSON, protobuf, etc.) and decodes accordingly.

A `data` body **can have multiple chunks** — each is its own `data` section. They concatenate logically into one payload. In practice the SDK gives you one chunk; multiple chunks are a wire-level optimisation rarely surfaced to apps.

This is what Service Bus's `body=b'...'` becomes on the wire.

### `amqp-value` — a single typed value

```
body = amqp-value { string: "hello world" }
body = amqp-value { map: { "name" → "Alice", "age" → 30 } }
body = amqp-value { uuid: "abc-123-..." }
```

The body is one strongly-typed AMQP value. Sender and receiver both use the AMQP type system to encode/decode. Useful when there's a strict schema both sides agree on at protocol level.

Service Bus rarely uses this. Some Azure services (Event Grid over AMQP, certain IoT scenarios) might. If you see `BodyType.AmqpValue` in SDK code, that's this shape.

### `amqp-sequence` — a list of typed values

```
body = amqp-sequence [
    { int: 100 },
    { int: 200 },
    { int: 300 }
]
```

The body is an ordered list of typed values. Designed for record streams. Almost never used in Service Bus.

## Service Bus body in practice

When you write:

```python
ServiceBusMessage(body=b'{"order_id": 9931}')
```

The SDK builds the body as:

```
body = data { <bytes of '{"order_id": 9931}'> }
```

Other SDK conveniences:

```python
ServiceBusMessage(body="hello")
# → SDK encodes UTF-8 → data { 0x68, 0x65, 0x6C, 0x6C, 0x6F }

ServiceBusMessage(body={"order_id": 9931})
# → SDK serialises as JSON → data { <json bytes> }

ServiceBusMessage(body=protobuf_msg.SerializeToString())
# → already bytes → data { <protobuf bytes> }
```

The SDK handles the encoding. The wire format is always `data` for these.

The receiving SDK exposes `msg.body` (sometimes a generator yielding bytes) — you decode based on `msg.content_type`.

## Message size

Service Bus enforces a **total message size cap** that includes all 6 sections, not just the body:

| Tier | Limit |
|---|---|
| Standard | 256 KB total |
| Premium | 100 MB total (with large message support enabled) |

Headers, Properties, Annotations, App Properties all count against the limit. The body is usually the dominant piece, but a message with 50 KB of Application Properties leaves only ~200 KB for the body on standard tier.

When you're approaching the limit, the right answer is the **claim-check pattern**: store the large payload in Azure Blob Storage and send only a reference (URL + checksum) in the message. Covered in Section 10.

## Real-world example

A producer sends an order event:

```python
import json

order = {"order_id": 9931, "customer": "alice", "total": 4999, "items": [...]}

ServiceBusMessage(
    body=json.dumps(order).encode("utf-8"),
    content_type="application/json",        # → Properties.content-type
    message_id=f"order-{order['order_id']}", # → Properties.message-id
)
```

Wire layout:

```
properties {
    message-id   = "order-9931"
    content-type = "application/json"
}
body = data { <json bytes> }
```

The broker:
- Reads Properties to dedup and route — does **not** decode the body
- Persists the message (body bytes opaque on disk)
- Delivers to the consumer

The consumer:

```python
for msg in receiver.receive_messages():
    raw = bytes(msg.body)                    # raw bytes from Body section
    if msg.content_type == "application/json":
        order = json.loads(raw.decode("utf-8"))
```

The consumer uses `content_type` (a Properties field) to decide how to decode the body. This is the standard pattern: **metadata in Properties, payload in Body, decoder picks based on metadata.**

## Mental model

> **The Body is the letter inside the envelope.**
>
> [[Header]], [[Properties]], [[Application Properties]], [[Message Annotations]] — all the stamps, addresses, sticky notes, customs marks — are on the *outside* of the envelope. The post office reads them to deliver.
>
> The Body is the **letter itself, sealed inside.** The post office never opens the envelope. Only the recipient reads the letter.
>
> AMQP gave you three "letter formats": raw paper (bytes / `data`), a typed form (`amqp-value`), or a stack of forms (`amqp-sequence`). Service Bus uses raw paper. You write whatever you want on it; the post office doesn't read or care.

## Interview answer

Body is the AMQP message section carrying the application payload. AMQP defines three body shapes — `data` (opaque bytes), `amqp-value` (a single typed value), and `amqp-sequence` (a list of typed values) — but in practice almost everyone uses `data`. Service Bus uses `data` exclusively: when you pass `body=b'...'` to the SDK, it wraps the bytes in a `data` section. The broker treats the body as opaque — it never decodes, never filters on body content, never inspects what's inside. Anything that needs to drive routing or filtering must live in [[Properties]] or [[Application Properties]] where the broker can read it. Service Bus enforces a total message size cap (256 KB standard tier, 100 MB premium with large message support) that includes all sections — when payloads grow, the standard answer is the claim-check pattern: store the large data in blob storage, put a reference in the message.

## Common misconceptions

- **"Service Bus filters can read the body."** No. Filters can read [[Header]] (priority), [[Properties]] (subject, message-id, etc.), and [[Application Properties]] only. The body is invisible to filters.
- **"Setting `content-type` parses the body for me."** No. `content-type` is purely informational — a hint for the consumer about how to decode. The broker and SDK do not auto-parse based on it.
- **"The body has to be JSON / a specific format."** No. The body is opaque bytes. JSON, protobuf, MessagePack, encrypted binary, gzipped anything, plain text — all valid. The format is your application's choice; declare it via `content-type` so the consumer knows how to decode.
- **"Big payloads should be compressed harder."** Sometimes, but the right answer is usually the claim-check pattern. Compression hits diminishing returns; offloading to blob storage is unbounded.
- **"`body` and `application_properties` are interchangeable."** No. Body is the payload (opaque, broker can't read). Application Properties is metadata (visible to the broker for filters). Putting business attributes in the body kills filterability.
- **"Setting body to a Python dict sends it as `amqp-value`."** No. The Service Bus SDK serialises the dict as JSON and sends it as `data`. To actually send `amqp-value`, you'd construct the message with `body_type=AmqpMessageBodyType.VALUE` explicitly — and even then most receivers expect `data`.

## See also

- [[Properties]] — `content-type` and `content-encoding` declare the body's format
- [[Application Properties]] — where filterable business metadata lives (the body alternative)
- [[Multi-Frame Messages]] — large bodies span multiple TRANSFER frames; the `more` flag glues them
- [[AMQP Message Structure]] — the 6-section overview

## Index

[[AMQP Message Structure]]
