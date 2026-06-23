---
tags:
  - amqp
  - message-structure
---

# Properties

> **Properties is the addressing section.** A closed list of ~13 fields, defined by the AMQP spec, that every endpoint (sender + consumer) reads to figure out *what is this message, where is it going, and how do I reply to it?* Same closed-list discipline as [[Header]] — but Header is for the broker's hot-path delivery decisions, Properties is for endpoint addressing. The key fields are `message-id`, `correlation-id`, `subject`, `reply-to`, and `group-id` (Service Bus `SessionId`).

## Definition

**Properties** is the 4th of an AMQP message's six sections. It contains a closed list of ~13 standardised fields covering identity (`message-id`, `user-id`), addressing (`to`, `subject`, `reply-to`), correlation (`correlation-id`), body interpretation (`content-type`, `content-encoding`), lifecycle (`creation-time`, `absolute-expiry-time`), and grouping (`group-id`, `group-sequence`, `reply-to-group-id`).

Like [[Header]], the field list is closed — vendors can't add new ones. Custom application metadata goes in [[Application Properties]] instead.

## Problem it solves

Once Header has answered "is this message alive, durable, prioritised?", endpoints need standardised slots to answer:

1. **What's this message's ID?** — for dedup, tracing, and replying back to it later.
2. **Who sent it?** — for auth and audit (`user-id`, broker-stamped).
3. **What kind of message is it?** — for filtering and routing (`subject`).
4. **Where should a reply go?** — for request/reply patterns (`reply-to`).
5. **Which earlier message is this a reply TO?** — to wire up the response (`correlation-id`).
6. **What format is the body?** — JSON, XML, gzipped JSON (`content-type`, `content-encoding`).
7. **Which logical group does it belong to?** — for FIFO ordering (`group-id` = Service Bus `SessionId`).

These are universal across messaging — every system needs them. So AMQP gave them fixed slots that every client/broker on Earth understands the same way.

## Why previous solution was insufficient

If these fields lived in [[Application Properties]] (the open custom dictionary), each app would invent its own field name:

- App 1: `request_id` / `respond_to` / `in_reply_to`
- App 2: `msg_uuid` / `reply_queue` / `parent_msg`
- App 3: `id` / `callback` / `ref`

Three problems fall out:

1. **No cross-team interop.** Two services from different teams can't talk request/reply without first agreeing on field names.
2. **No tooling.** Service Bus Explorer, AMQP packet inspectors, distributed tracing — none of these can auto-build views without standard names.
3. **No SDK helpers.** Service Bus's request/reply patterns (`reply_to`, `reply_to_session_id`) only work because the underlying AMQP fields are standardised. If they were custom, every SDK would need per-app config.

The fix is the same principle behind [[Header]]: **closed core for interoperability, open extensions for innovation.** Properties is the closed core for endpoint addressing; [[Application Properties]] is the open extension.

## Responsibilities

Properties owns:

- **Message identity** — a globally unique ID the producer stamps and the broker can dedup against.
- **Endpoint addressing** — where the message is going (`to`), what kind it is (`subject`), where a reply should go (`reply-to`).
- **Correlation** — the back-reference (`correlation-id`) that wires replies to their original requests.
- **Body interpretation** — how the consumer should decode the bytes (`content-type`, `content-encoding`).
- **FIFO grouping** — `group-id` is Service Bus's `SessionId`; all messages with the same value go to the same consumer in order.

What Properties does NOT own:

- **Hot-path delivery decisions** — TTL, durability, priority live in [[Header]].
- **Custom app metadata** — `customer_tier`, `region`, `priority_lane` go in [[Application Properties]].
- **Vendor-specific extensions** — Service Bus's `ScheduledEnqueueTime`, `PartitionKey`, `LockedUntil` live in [[Message Annotations]] under `x-opt-`.

## The 13 fields

```
properties {
    // Identity
    message-id            : *           // unique ID of THIS message
    user-id               : binary      // who sent it (auth identity, broker-stamped)

    // Addressing
    to                    : string      // destination address
    subject               : string      // short label / message type
    reply-to              : string      // where to send the reply

    // Correlation
    correlation-id        : *           // ID of the message THIS is replying to

    // Body interpretation
    content-type          : symbol      // MIME type of the body
    content-encoding      : symbol      // gzip, deflate, etc.

    // Lifecycle
    absolute-expiry-time  : timestamp   // hard deadline (vs Header.ttl which is relative)
    creation-time         : timestamp   // when producer built it

    // Grouping (used by Service Bus Sessions)
    group-id              : string      // logical group / Service Bus SessionId
    group-sequence        : uint        // position within the group
    reply-to-group-id     : string      // group on the reply queue
}
```

### `message-id` — the message's unique ID

Producer stamps it. Used for:

- **Broker dedup** — Service Bus drops duplicates within the dedup window when `MessageId` matches a recent one.
- **Tracing** — logs and dashboards refer to the message by this ID.
- **Reply correlation** — the responder will copy this into its `correlation-id` when replying.

Mental model: **the certified-mail tracking number printed on the envelope.**

### `correlation-id` — back-reference to an earlier message

When B replies to A's question, B copies A's `message-id` into the reply's `correlation-id`. A reads it to know which pending request this is the answer to.

The pairing:
- `message-id` = *"I am ___"*
- `correlation-id` = *"I'm responding to ___"*

Mental model: **"Re: your letter dated June 23rd"** written on a reply letter.

### `to` — destination address

The intended target queue/topic. SDK usually fills this automatically based on the sender's target. Used by auto-forwarding and federated-broker scenarios that need to know where the message was originally aimed.

Mental model: **the printed delivery address on the envelope.**

### `subject` — short message-type label

Like an email subject line. *"OrderPlaced"*, *"PriceQuery"*, *"PriceResult"*.

**Why it's important:** subscriptions in topics can filter on `subject`. The broker reads `subject` (a few bytes) instead of decoding the body to decide which subscriptions get the message.

Mental model: **the *RE: monthly invoice* line of an email.**

### `reply-to` — where the answer should go

When A asks B a question, A puts the message on a queue B reads from. But where does B put the **answer**? B doesn't know A's address — A might be 1 of 50 services using B.

So A tells B: *"after you answer, send the reply to this queue."* That queue's name is `reply-to`.

```python
# A asking B
ServiceBusMessage(
    body=b'{"sku": "SKU-99"}',
    message_id="req-7741",
    reply_to="price-replies-for-svc-A",   # ← B, send the answer to this queue
)
```

`reply-to` is just the **name of a queue** as a string. A is listening on that queue. Different services use different `reply-to` queues; B doesn't care, B just reads the field and addresses its reply accordingly.

Mental model: **the return address on a postcard.**

### `user-id` — who sent it (auth identity)

Broker-stamped, not producer-stamped. The broker knows who connected with which credentials, so its stamp on this field is trustworthy. Prevents spoofing — without `user-id`, anyone could put any name in a custom field and lie.

Mental model: **a notarised signature** the clerk (broker) witnessed.

### `content-type` — body format

Standard MIME types: `application/json`, `application/xml`, `text/plain`, `application/octet-stream`, etc. Tells the consumer how to parse the body.

### `content-encoding` — body compression

Used together with `content-type`: *"this is JSON, gzipped"*. Empty if not compressed.

### `creation-time` — when the producer built it

Timestamp, filled in by SDK automatically. Useful for debugging — *"this message was created 4 hours ago, why is it still in the queue?"*

### `absolute-expiry-time` — hard deadline

A specific UTC timestamp at which the message expires. Different from Header's `ttl`:

- **`ttl`** (Header) = *"live for 60 seconds from now"* — relative duration
- **`absolute-expiry-time`** (Properties) = *"die at exactly 3:00:00 PM on June 23rd"* — absolute timestamp

Service Bus mostly uses `ttl`. The absolute version exists for cases where multiple producers across time zones agree on a single deadline.

### `group-id` — Service Bus `SessionId`

All messages with the same `group-id` belong to the same logical group and go to the **same consumer in order**. This is how Service Bus does FIFO — within a session, order is preserved; across sessions, order doesn't matter.

```python
ServiceBusMessage(group_id="order-9931", body=b'step 1: charge card')
ServiceBusMessage(group_id="order-9931", body=b'step 2: ship item')
# Both messages go to the same consumer, in order
```

Covered in detail in Section 10.

### `group-sequence` — position within the group

`0`, `1`, `2`, ... lets the consumer detect gaps (*"I got 0 and 2, where's 1?"*).

### `reply-to-group-id` — group on the reply queue

Pairs with `reply-to`. When A's reply queue is also session-based, A can say *"send the answer to queue X (`reply-to`) AND tag it with session Y (`reply-to-group-id`)."*

## Service Bus name mapping

Most fields you'll touch in the SDK map directly to AMQP Properties:

| AMQP field | Service Bus SDK name |
|---|---|
| `message-id` | `MessageId` |
| `correlation-id` | `CorrelationId` |
| `subject` | `Subject` (was `Label` in older SDKs) |
| `reply-to` | `ReplyTo` |
| `to` | `To` |
| `content-type` | `ContentType` |
| `group-id` | `SessionId` |
| `reply-to-group-id` | `ReplyToSessionId` |

These are SDK-level parameters with their own named slots in `ServiceBusMessage(...)`. **Application Properties** is the dict — anything inside `application_properties={}` is custom metadata you defined.

## Real-world example: full request/reply

Service A asks Service B for the price of `SKU-99`:

```python
# A → B (the question)
request = ServiceBusMessage(
    body=b'{"sku": "SKU-99"}',

    # Properties (standard AMQP fields)
    message_id="req-7741",
    subject="GetPrice",
    reply_to="price-replies-for-svc-A",
    content_type="application/json",

    # Application Properties (A's custom metadata)
    application_properties={
        "tenant_id": "tenant-44",
        "trace_id": "abc-123",
    },
)
sender_to_B.send_messages(request)
```

What's on the wire:

```
properties {
    message-id     = "req-7741"
    subject        = "GetPrice"
    reply-to       = "price-replies-for-svc-A"
    content-type   = "application/json"
    creation-time  = 2026-06-23T10:33:21Z      // SDK auto-fills
    user-id        = "svc-a@company.com"        // broker stamps
}
application-properties {
    tenant_id  = "tenant-44"
    trace_id   = "abc-123"
}
```

B receives, computes the price, replies:

```python
# B → A (the answer)
reply = ServiceBusMessage(
    body=b'{"sku": "SKU-99", "price": 4999}',

    message_id="resp-9912",          # B's NEW id for this reply
    correlation_id="req-7741",       # ← echoes A's message-id
    subject="PriceResult",
    content_type="application/json",
)
sender_to_replies_A.send_messages(reply)
```

A pulls the reply off `price-replies-for-svc-A`:

```python
for reply in receiver_a.receive_messages():
    answer_to = reply.correlation_id   # "req-7741"
    pending_request = self.in_flight[answer_to]
    pending_request.set_result(reply.body)
```

The whole pattern works because three fields are **standardised**:

- A used `message-id` to stamp the request
- A used `reply-to` to tell B where to send the answer
- B copied A's `message-id` into `correlation-id` on the reply

If any of these were custom-named, generic SDK helpers and tracing tools would break. With Properties, it just works.

## Mental model

> **Properties is the printed fields on an envelope.**
>
> - `message-id` = certified-mail tracking number printed on the envelope
> - `to` = printed delivery address
> - `reply-to` = return address
> - `subject` = "RE:" line through the address window
> - `correlation-id` = "Re: your letter dated June 23rd" written on the inside front
> - `content-type` = language declaration ("written in English / JSON")
> - `user-id` = notarised stamp from the post office
> - `group-id` = "this is letter 3 of a 5-letter series sent to the same address"
>
> The post office (broker) reads the address slots to deliver. The recipient reads everything. Both can rely on every envelope from every sender having these slots **in the same place** — that's standardisation.
>
> [[Application Properties]] are the **sticky notes** you put on the envelope. Both sides can write whatever they want; the post office doesn't read sticky notes. Properties is the printed envelope itself — fixed slots, every post office understands them.

## Interview answer

Properties is the AMQP message section that carries standardised endpoint-addressing fields — about 13 of them, fixed by the spec. The key ones are `message-id` (the message's unique ID, used for dedup and tracing), `correlation-id` (the back-reference that wires replies to their original requests), `subject` (a short label used for filtering in topics), `reply-to` (the queue name where the answer should be sent), and `group-id` (which is Service Bus's `SessionId` — the FIFO ordering key). The list is closed for the same reason Header is — every AMQP-compliant client and broker must understand these fields with the same meaning, so that request/reply patterns, dedup, and SDK helpers work generically across vendors. Custom application metadata goes in [[Application Properties]] instead. The four fields you'll touch most often in Service Bus code are `MessageId`, `CorrelationId`, `Subject`, and `SessionId`.

## Common misconceptions

- **"Properties is where I put my custom message metadata."** No. Properties is closed — about 13 standard fields. Custom data goes in [[Application Properties]].
- **"`reply-to` is a special return channel."** No. It's just a string field carrying the name of a regular queue. The producer of the reply reads that string and sends the reply to the queue named there. There's no magic.
- **"`message-id` and `correlation-id` are the same thing."** No. `message-id` is the ID of THIS message; `correlation-id` is the ID of an EARLIER message that this one is responding to. A reply has both — its own new `message-id` and the original request's id in `correlation-id`.
- **"`user-id` is set by the producer."** No. The broker stamps it based on the authenticated identity of the connection. That's what makes it trustworthy for audit.
- **"`subject` is the same as `message-id`."** No. `subject` is a short label describing the *kind* of message (`"OrderPlaced"`); `message-id` is a unique identifier for this *specific instance* (`"order-9931"`). Many messages share a subject; each has a unique message-id.
- **"`absolute-expiry-time` and Header's `ttl` are the same."** Related but different. `ttl` is a relative duration ("60 seconds from now"); `absolute-expiry-time` is a specific timestamp ("die at 3:00 PM"). Service Bus mostly uses `ttl`.
- **"`group-id` is just Service Bus `SessionId` rebranded."** Other way round. `group-id` is the AMQP standard field; Service Bus's `SessionId` is the SDK name for it. The wire field has been called `group-id` since AMQP 1.0 in 2011.
- **"If a Service Bus SDK parameter has its own slot, it's in Properties."** Mostly true — the SDK's named parameters (`message_id`, `subject`, `reply_to`, `session_id`) map to Properties. Anything inside `application_properties={...}` is the custom dict.

## See also

- [[Header]] — the closed-list section that comes before; broker hot-path decisions vs Properties' endpoint addressing
- [[Application Properties]] — the open counterpart to Properties; where custom app metadata goes
- [[Message Annotations]] — vendor-specific extensions under `x-opt-` (where Service Bus puts `ScheduledEnqueueTime`, `PartitionKey`, etc.)
- [[AMQP Message Structure]] — the 6-section overview Properties is part of
- [[Settlement Modes]] — `message-id` is the dedup key Service Bus uses to deliver exactly-once via broker-side dedup
- [[Disposition States]] — `correlation-id` ties replies to requests in the request/reply pattern that uses dispositions to confirm delivery

## Index

[[AMQP Message Structure]]
