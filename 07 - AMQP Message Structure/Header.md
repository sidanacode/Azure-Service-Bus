---
tags:
  - amqp
  - message-structure
---

# Header

> **Header is the broker's must-read section.** Five fields, fixed by the AMQP spec, sitting at the top of every message. Every broker on the path reads them on the hot path to decide *accept / drop / route / persist*. Anything broker-relevant that's universal across AMQP lives here. Anything vendor-specific (Service Bus extensions) lives in [[Message Annotations]] instead.

## Definition

**Header** is the first of an AMQP message's six sections. It contains a closed list of exactly **5 fields** standardised by the AMQP 1.0 spec — `durable`, `priority`, `ttl`, `first-acquirer`, `delivery-count`. Every field is something a broker reads on every hop to make an immediate routing/persistence/expiry decision, before touching the rest of the message.

The list is *closed* — vendors cannot extend it. Microsoft cannot add a 6th field. Service Bus's own extensions (e.g. `ScheduledEnqueueTimeUtc`, `PartitionKey`) live in [[Message Annotations]] under the `x-opt-` prefix.

## Problem it solves

A broker receives a TRANSFER frame carrying a multi-section message. Before doing anything else (dedup check, filter evaluation, routing, body delivery), it has to answer 3 questions:

1. **Is this message even still alive?** → TTL — has it expired?
2. **Should I save it across a restart?** → durable — is this telemetry I can drop, or a payment I must persist?
3. **Does this jump the queue?** → priority — VIP customer or critical alert?

These decisions are **hot-path** — they happen at line rate on every hop. The broker hasn't checked message-id yet, hasn't evaluated subscription filters, hasn't even deserialised the body. It just needs the bare minimum to decide *"do I keep processing this, or drop it now?"*

Header is that bare minimum, in a section the broker can read with a single short decode.

## Why previous solution was insufficient

Imagine the alternative — one flat dictionary `{message_id, ttl, region, body, priority, ...}`:

- Broker has to scan the whole dict every hop just to find its own fields. At 1M msg/sec, that's CPU wasted on every message.
- Vendor extensions and AMQP-standard fields collide in one namespace. *"Whose `enqueued-time` wins, mine or the broker's?"*
- Mandatory hot-path fields (TTL, priority) are mixed with optional rare fields (footer, signatures). Broker decodes the body just to peek at TTL.

The flat-dict design makes both extensibility and performance impossible.

The fix — the principle behind AMQP's 6-section split — is **group fields by who owns them and how often they're read.** Header is the section for *broker-relevant, hot-path, universal* fields. That's why it sits first in the wire layout and why its schema is frozen.

## Responsibilities

Header owns:

- **Hot-path delivery decisions** — TTL expiry check, durable persistence flag, priority ordering.
- **Redelivery tracking** — `first-acquirer` and `delivery-count` survive across reconnects (broker-side state, see Section 6 anchor) so retries are visible to the consumer.
- **Interoperability across all AMQP brokers** — every AMQP 1.0 broker on Earth must understand these 5 fields with the same meaning.

What Header does *not* own:

- **Addressing** — `message-id`, `correlation-id`, `to`, `reply-to` live in [[Properties]]. Header is for "should I deliver?", Properties is for "where and to whom?".
- **Custom application metadata** — your `region`, `customer_tier`, `priority_lane` live in [[Application Properties]]. Header is closed; Application Properties is open.
- **Vendor-specific extensions** — Service Bus's `ScheduledEnqueueTime`, `PartitionKey`, `LockedUntil` etc. live in [[Message Annotations]] under `x-opt-`.

## The 5 fields

```
header {
    durable           : bool       // survive broker restart?
    priority          : ubyte      // 0..9, default 4 — higher jumps the queue
    ttl               : uint       // milliseconds; 0 = never expire
    first-acquirer    : bool       // has anyone tried to deliver this before?
    delivery-count    : uint       // how many times has this been delivered?
}
```

### `durable` — the disk-write decision

Tells the broker *"if you crash, must this message survive?"*

- `durable = true` → broker MUST persist before sending DISPOSITION
- `durable = false` → broker MAY keep it only in memory (faster, lost on crash)

**Per-message, not per-queue.** A single queue can carry both kinds — the producer decides per-message. Some brokers (RabbitMQ classic) made this a queue property; AMQP made it per-message because real workloads mix telemetry (drop-OK) and payments (must-survive) on the same queue.

**Service Bus welds this to `true`.** The product writes everything to disk; the SDK sets `durable=true` on every TRANSFER. The flag is "redundant" in the sense that Service Bus would persist anyway — but it's still on the wire because AMQP-compliance requires it.

This is a recurring pattern: **the protocol exposes a knob; the product chooses to weld it in one position.** Service Bus welds `durable=true`. Kafka with `acks=1` welds the *opposite* direction (acks before fsync — fast, but loses on crash).

The two-part guarantee for Service Bus durability:

1. `durable=true` on the wire → "I want this persisted"
2. DISPOSITION sent only AFTER fsync → "I have it persisted"

Both together mean: when the producer receives `accepted`, the message is on disk. (See the *settled ≠ durable* anchor in [[Session]] — `durable` is what binds those two steps together at the broker side.)

### `priority` — queue-jumping

Range 0–9, default 4. Higher = delivered first.

- **Service Bus standard tier**: ignores priority. FIFO only.
- **Service Bus premium tier**: respects priority.

Recognition trigger: *"I set priority but it had no effect"* almost always means standard tier.

### `ttl` — time-to-live in milliseconds

Broker drops expired messages without looking at the rest of the message. First-pass filter.

- `ttl = 0` → never expire (default)
- Service Bus exposes this as `TimeToLive`, capped at **14 days**

A message can expire on the wire (TTL while sitting in the queue waiting to be delivered) or in flight (TTL while locked by a consumer). In both cases, the broker reads `ttl` from Header to make the call without deserialising anything else.

### `first-acquirer` — fresh delivery flag

`true` = no consumer has ever taken this message before. `false` = at least one prior delivery attempt (lock expired, abandon, crash).

Useful for retry-tolerant logic — *"if first-acquirer is false, my idempotency check needs to be on."*

Service Bus SDK does **not** expose this directly. It exposes `delivery_count` instead, which is more useful, and you derive the boolean if needed (`first_acquirer == (delivery_count == 0)`).

### `delivery-count` — redelivery counter

How many times the broker has tried to deliver this message before this attempt. Incremented on every redelivery (lock expiry, abandon, consumer crash).

This is the field that drives **auto-DLQ at `MaxDeliveryCount`** — the Service Bus retry-storm safety net (default 10). When `delivery-count` crosses the threshold, the broker auto-moves the message to DLQ with reason `MaxDeliveryCountExceeded`. (See Section 5 anchor.)

**Why is `delivery-count` in Header rather than Message Annotations?** Because *redelivery tracking is universal across AMQP*. Every broker on Earth needs to behave the same way around it — every consumer, on every SDK, must be able to see "this message was delivered N times before." If Microsoft had put it in `x-opt-delivery-count`, a RabbitMQ consumer reading a Service Bus message wouldn't see it; cross-broker SDKs would each invent their own field name. Closed-list standardisation is what makes interoperability possible.

The general rule: **a field belongs in Header (or Properties) if every AMQP broker on Earth needs to behave the same way around it. It belongs in Annotations if only some brokers care.**

## Real-world example

A producer sends a payment confirmation to Service Bus:

```python
ServiceBusMessage(
    body=b'{"order_id": 9931, "paid": true}',
    time_to_live=timedelta(hours=1),    # → Header.ttl = 3_600_000 ms
    # priority not set                  → Header.priority = 4 (default)
)
```

When the SDK builds the TRANSFER, the on-wire Header looks like:

```
header {
    durable        = true       // SDK welds this on
    priority       = 4          // default
    ttl            = 3_600_000  // 1 hour in ms
    first-acquirer = true       // first attempt
    delivery-count = 0          // never delivered before
}
```

The broker, on receiving the TRANSFER:

1. Reads `header.ttl` → "still alive, accept" (without touching Properties or Body)
2. Reads `header.durable` → "must fsync before DISPOSITION"
3. Writes the message to disk
4. Sends DISPOSITION with `accepted`

If the broker power-cycles 2 seconds later, the message is still there — because step 4 only ran after step 3. The two welded behaviours (`durable=true` flag + DISPOSITION-after-fsync) together produce Service Bus's "guaranteed durability" promise.

Now the consumer abandons the message twice (network blip, retry). Broker increments `delivery-count` to 2. The consumer's SDK sees `msg.delivery_count == 2` and can decide: *"third try, log loudly."* If `delivery-count` reaches `MaxDeliveryCount` (default 10), the broker auto-DLQs the message — reading `delivery-count` from Header without ever touching Properties.

## Mental model

> **Header is the shipping label glued to the outside of a parcel.**
>
> Every customs officer (broker) on the route reads it without opening the parcel: weight (priority), expiry date (TTL), fragile sticker (durable), how many handlers it's passed through (delivery-count). The label is **standardised by international postal treaty (the AMQP spec)** so every country's customs agency understands it the same way.
>
> If you wanted to attach Country-of-Mexico-specific info, you'd use a *separate sticker* — that's [[Message Annotations]]. The shipping label itself can't be customised; that's the whole point.

## Interview answer

Header is the AMQP message section that carries broker-relevant, hot-path, universal fields — exactly 5 of them, fixed by the spec: `durable`, `priority`, `ttl`, `first-acquirer`, `delivery-count`. Every AMQP broker reads Header on every hop to make immediate accept/drop/persist/route decisions before deserialising anything else, which is why Header sits first in the wire layout and has a closed schema. The list is closed for interoperability — any AMQP-compliant client talking to any AMQP-compliant broker speaks the same Header. Vendor extensions like Service Bus's `ScheduledEnqueueTime` or `PartitionKey` cannot extend Header; they live in [[Message Annotations]] under the `x-opt-` prefix instead. Service Bus welds `durable=true` for all messages (the product always persists), respects `ttl` up to 14 days, ignores `priority` on standard tier (premium tier respects it), and uses `delivery-count` to drive auto-DLQ at `MaxDeliveryCount` — the retry-storm safety net.

## Common misconceptions

- **"Header is where I put my custom message metadata."** No. Header is closed — exactly 5 fields, frozen by the spec. Your custom metadata goes in [[Application Properties]].
- **"`durable=false` means Service Bus won't persist."** No. Service Bus welds persistence on regardless of the flag. The flag is a protocol contract; product behaviour is independent.
- **"Service Bus respects message priority."** Only on premium tier. Standard tier is FIFO; the priority field is silently ignored.
- **"`delivery-count` resets when the consumer reconnects."** No. `delivery-count` is broker-side state, stored on the message itself (Section 6 anchor — *broker-side state survives reconnects, Session-side state doesn't*). Reconnect a thousand times; the count keeps climbing.
- **"`first-acquirer` is the field I should check for retry logic."** Use `delivery-count` instead. Service Bus SDK doesn't even expose `first-acquirer` — it gives you the count, which is more useful and contains the boolean.
- **"`ttl` is the same as Service Bus message lock duration."** No. `ttl` is *how long the message is alive in the system* (in queue + in flight). Lock duration is *how long one consumer can hold a message before it's redelivered.* Different timers, different concerns. (See [[Lock Duration and Renewal]].)
- **"Microsoft could add a field to Header in a future Service Bus version."** No. Header is frozen by the AMQP 1.0 spec, an OASIS standard. Microsoft can add fields to Message Annotations or Application Properties; they cannot extend Header without breaking AMQP-compliance.

## See also

- [[AMQP Message Structure]] — the 6-section overview Header is the first of
- [[Properties]] — the addressing section that comes next, also a closed list
- [[Message Annotations]] — where vendor extensions like `x-opt-scheduled-enqueue-time` live
- [[Application Properties]] — your custom metadata, the open counterpart to Properties
- [[Settlement Modes]] — the `durable=true` + DISPOSITION-after-fsync pairing makes settled-and-durable a real guarantee
- [[Lock Duration and Renewal]] — `delivery-count` is the field that drives `MaxDeliveryCount` auto-DLQ
- [[Disposition States]] — `Abandon` increments `delivery-count`; `Complete` deletes the message before the count matters

## Index

[[AMQP Message Structure]]
