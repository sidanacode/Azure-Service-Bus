---
tags:
  - amqp
  - message-structure
---

# Message Annotations

> **Message Annotations is the broker's open dictionary** — the escape hatch where vendor-specific extensions live without polluting AMQP's standardised sections. Service Bus uses it heavily, with the `x-opt-` prefix marking every entry as a Microsoft extension. This is where `ScheduledEnqueueTime`, `PartitionKey`, `LockedUntil`, and the broker-stamped `SequenceNumber` and `EnqueuedTimeUtc` all live. If you see `x-opt-` in a packet capture or SDK source, it's Message Annotations.

## Definition

**Message Annotations** is the 3rd of an AMQP message's six sections. It's an **open dictionary** (string keys mapped to scalar values) that the broker fills with metadata it needs to track per message. Unlike [[Application Properties]] (which is the *application's* open dict), Message Annotations is owned by the **broker**.

Vendors stamp keys with the `x-opt-` prefix as a convention to mark *"this is a vendor extension, not AMQP-standard."* A `Service Bus → RabbitMQ` cross-broker forward would carry the `x-opt-` keys but the receiving broker would pass them through opaquely.

## Problem it solves

Once Header (5 closed fields) and Properties (~13 closed fields) are settled, brokers still need extra metadata that AMQP itself doesn't define:

- *"When did this message enter the queue?"* — `x-opt-enqueued-time`
- *"What's its position in the queue?"* — `x-opt-sequence-number`
- *"Until when is the current consumer's lock valid?"* — `x-opt-locked-until`
- *"Should this be held back until 3pm tomorrow?"* — `x-opt-scheduled-enqueue-time`
- *"Which partition should this go to?"* — `x-opt-partition-key`
- *"What was the queue this came from before being dead-lettered?"* — `x-opt-deadletter-source`

None of these are universal across AMQP — they're Service Bus features. AMQP can't pre-define every vendor's broker extensions, so it leaves a section open for vendors to fill.

## Why previous solution was insufficient

If Microsoft had tried to extend [[Header]] or [[Properties]] with these fields:

1. **Spec violation.** Header is a closed 5-field list; Properties is a closed ~13-field list. AMQP forbids adding more.
2. **Cross-broker confusion.** A RabbitMQ consumer reading a Service Bus message would see fields it doesn't understand inside the standardised sections — does it ignore them? Error? Crash?
3. **Future collisions.** If AMQP 2.0 ever adds a real `scheduled-enqueue-time` field to Header, Microsoft's custom version would silently collide.

The fix is the `x-opt-` prefix convention plus a section that's open by design. By prefix, every reader knows *"this is vendor-specific, not protocol-standard."* By section, every reader knows where to find it without polluting standardised slots.

Same principle as Application Properties, just on the broker side.

## Responsibilities

Message Annotations owns:

- **Broker-stamped metadata** — `x-opt-sequence-number`, `x-opt-enqueued-time`, `x-opt-locked-until` are written by the broker when it accepts/locks a message.
- **Producer-set broker hints** — `x-opt-scheduled-enqueue-time`, `x-opt-partition-key` are set by the producer but read and acted on by the broker.
- **Cross-broker pass-through** — when a Service Bus message is forwarded to another AMQP broker, `x-opt-` fields travel with it but the receiving broker treats them as opaque.

What Message Annotations does NOT own:

- **Universal AMQP fields** — `ttl`, `priority`, `message-id`, `subject` go in [[Header]] / [[Properties]].
- **Application metadata** — your `tenant_id`, `region`, `customer_tier` go in [[Application Properties]].
- **Consumer-side state** — lock tokens consumed by the SDK live on the [[Disposition States|delivery]], not in annotations.

## Service Bus's `x-opt-` field catalogue

The most common Service Bus extensions you'll see in traces:

| `x-opt-` field | Set by | What it carries |
|---|---|---|
| `x-opt-sequence-number` | Broker | Globally unique long; broker-assigned position in the queue |
| `x-opt-enqueued-time` | Broker | UTC timestamp when broker accepted the message |
| `x-opt-locked-until` | Broker | Timestamp when current consumer's lock expires (see [[Lock Duration and Renewal]]) |
| `x-opt-scheduled-enqueue-time` | Producer | "Hold this back; deliver at this UTC time" |
| `x-opt-partition-key` | Producer | Partitioned-queue routing key |
| `x-opt-via-partition-key` | Producer | Partition key for transactional auto-forward |
| `x-opt-deadletter-source` | Broker | When DLQ'd, the original queue/subscription name |
| `x-opt-enqueue-sequence-number` | Broker | Sequence within the destination after auto-forward |
| `x-opt-message-state` | Broker | Active / Deferred / Scheduled |

These all map to SDK properties:

| `x-opt-*` field | SDK property |
|---|---|
| `x-opt-sequence-number` | `msg.sequence_number` |
| `x-opt-enqueued-time` | `msg.enqueued_time_utc` |
| `x-opt-locked-until` | `msg.locked_until_utc` |
| `x-opt-scheduled-enqueue-time` | `scheduled_enqueue_time_utc` |
| `x-opt-partition-key` | `partition_key` |
| `x-opt-deadletter-source` | `msg.dead_letter_source` |

## Real-world example

A producer schedules a reminder email for 3pm tomorrow:

```python
ServiceBusMessage(
    body=b'{"to": "user@example.com", "subject": "reminder"}',
    message_id="reminder-9931",
    scheduled_enqueue_time_utc=datetime(2026, 6, 24, 15, 0),
)
```

What lands on the wire:

```
properties {
    message-id = "reminder-9931"
    creation-time = 2026-06-23T16:33Z
}
message-annotations {
    x-opt-scheduled-enqueue-time = 2026-06-24T15:00Z   ← producer-set
    x-opt-sequence-number = 4471                        ← broker-stamped on accept
    x-opt-enqueued-time = 2026-06-23T16:33:21Z          ← broker-stamped on accept
}
```

The broker reads `x-opt-scheduled-enqueue-time`, sees the future timestamp, holds the message in a hidden "scheduled" state until 3pm. Then it moves the message to the active queue. Consumers never see it before 3pm; their `receive_messages` calls don't return it.

If this same message were forwarded to RabbitMQ via federation, RabbitMQ would carry the `x-opt-scheduled-enqueue-time` annotation forward but ignore it — RabbitMQ has no concept of scheduled enqueue. That's the value of the `x-opt-` convention: it survives transit but doesn't break receivers that don't understand it.

## Mental model

> **Message Annotations is the customs stamps on a parcel.**
>
> [[Properties]] is the printed delivery address. [[Application Properties]] is sticky notes from the sender to the recipient. **Message Annotations is the customs stamps** — added by airports, ports, and border agencies as the parcel passes through.
>
> *"Entered Mumbai customs 16:33 UTC."* (`x-opt-enqueued-time`)
> *"Sequence #4471 in today's intake."* (`x-opt-sequence-number`)
> *"Hold until pickup at 15:00 tomorrow."* (`x-opt-scheduled-enqueue-time`)
> *"Reserved for picker A; release at 16:34 if not collected."* (`x-opt-locked-until`)
>
> The recipient can read them. Other airports passing the parcel can read them. But only the airport that defined the stamp truly *acts* on it.

## Interview answer

Message Annotations is the AMQP message section where broker-specific extensions live, separated from the standardised sections to keep the closed lists in [[Header]] and [[Properties]] from being polluted by vendor features. Service Bus uses Message Annotations heavily — every `x-opt-` field you see in packet captures or SDK source is a Microsoft extension that doesn't exist in plain AMQP. Some are broker-stamped on accept (`x-opt-sequence-number`, `x-opt-enqueued-time`, `x-opt-locked-until`); others are producer-set hints the broker reads (`x-opt-scheduled-enqueue-time`, `x-opt-partition-key`). The split between Message Annotations and [[Application Properties]] is by *ownership*: Message Annotations is the broker's open dict, Application Properties is the application's. The `x-opt-` prefix convention means a non-Microsoft broker can pass these fields through as opaque without breaking on unknown extensions.

## Common misconceptions

- **"Message Annotations and Application Properties are the same."** No. Message Annotations is the **broker's** open dict. Application Properties is the **application's** open dict. Different owners, different purposes.
- **"Anything I want to add custom goes in Message Annotations."** No. Your custom fields go in [[Application Properties]]. Message Annotations is for broker-stamped or broker-actionable fields. Producers don't typically write directly to Message Annotations — the SDK does it for known Service Bus features.
- **"`x-opt-` is part of the AMQP spec."** No. `x-opt-` is a Microsoft naming convention. AMQP itself only defines that Message Annotations is an open dictionary. Different vendors might use different prefixes (RabbitMQ uses `x-` for its annotations).
- **"`SequenceNumber` and `MessageId` are interchangeable."** No. `MessageId` (Properties) is producer-assigned, used for dedup. `SequenceNumber` (Message Annotations, broker-stamped) is broker-assigned and globally unique within the queue — used for ordering and as a stable cursor for receivers.
- **"`ScheduledEnqueueTime` is the same as `Header.ttl`."** Different ends of the lifecycle. `ScheduledEnqueueTime` says *"don't activate this until this time."* `ttl` says *"if not delivered by this duration, kill it."* One delays activation; the other enforces expiry.
- **"Setting `x-opt-locked-until` extends my lock."** No. `x-opt-locked-until` is broker-set state — the consumer reads it but doesn't write it. To extend the lock, the consumer calls `renew_message_lock` (which sends a control frame; the broker then updates `x-opt-locked-until` for the next read). See [[Lock Duration and Renewal]].

## See also

- [[Application Properties]] — the application's open dict counterpart
- [[Header]] — closed list for universal AMQP delivery controls
- [[Properties]] — closed list for universal AMQP addressing
- [[Lock Duration and Renewal]] — `x-opt-locked-until` is the wire field behind the lock timer
- [[Disposition States]] — `x-opt-deadletter-source` records where a DLQ'd message originated
- [[AMQP Message Structure]] — the 6-section overview

## Index

[[AMQP Message Structure]]
