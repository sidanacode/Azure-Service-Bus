---
tags:
  - amqp
  - message-structure
---

# Application Properties

> **Application Properties is your custom metadata dictionary.** AMQP defines none of the keys — they're yours to invent. The broker passes them through opaquely, with one important exception: topic subscription filters can read them. That's why business metadata that needs to influence routing (tenant ID, region, customer tier) goes here rather than in the body.

## Definition

**Application Properties** is the 5th of an AMQP message's six sections. It's an **open dictionary** — string keys mapped to scalar values (string, int, bool, timestamp, uuid, etc.) — that the producer fills with whatever custom fields the application needs.

Unlike [[Header]] and [[Properties]], the keys are not standardised. AMQP doesn't define what `customer_tier` or `tenant_id` mean — those are conventions of your application.

## Problem it solves

Once Header (broker hot-path) and Properties (universal endpoint addressing) are settled, every real application still has its own metadata to carry:

- *Which customer does this belong to?* — `tenant_id`
- *Which region's consumer should handle it?* — `region`
- *What service tier?* — `customer_tier`
- *Which experiment bucket?* — `ab_variant`
- *Which trace?* — `trace_id`, `span_id`

None of these are universal — every application invents its own. AMQP can't pre-define a slot for every possible business field, so it gives one section that's intentionally undefined: *"put whatever you want here, we won't touch it."*

## Why previous solution was insufficient

Without Application Properties, you'd have two bad choices:

1. **Stuff it in the body.** Then the broker can't see it. Topic filters become impossible — the broker would have to decode every message body to evaluate filter rules. Massive CPU cost, and binary/encrypted bodies wouldn't be readable at all.
2. **Cram it into Properties.** But Properties is a closed list of standard fields. You'd have to misuse `subject` as `tenant_id` or invent collisions with future AMQP versions. Cross-tool interop dies.

Application Properties is the clean third option: **a standardised location for non-standardised content.** AMQP defines *where* custom metadata goes (in this section, structured as a key/value dict); your app defines *what* goes in it.

## Responsibilities

Application Properties owns:

- **Custom business metadata** — tenant IDs, regions, customer tiers, anything app-specific.
- **Routing hints for topic filters** — broker reads these to evaluate subscription SQL filters.
- **Tracing/correlation extensions** — `trace_id`, `span_id`, request context that doesn't fit Properties' `correlation-id` model.
- **Feature flags / experiment buckets** — anything the consumer needs to read alongside the body.

What Application Properties does NOT own:

- **AMQP-standard fields** — `message-id`, `subject`, `reply-to` go in [[Properties]].
- **Vendor-specific broker extensions** — Service Bus's `ScheduledEnqueueTime`, `PartitionKey` go in [[Message Annotations]] under `x-opt-`.
- **Hot-path delivery decisions** — durable, ttl, priority go in [[Header]].
- **Large or binary payloads** — those go in the [[Body]]. Application Properties is for small scalar metadata.

## Service Bus filtering — the one time the broker reads it

For most messages, the broker treats Application Properties as opaque pass-through bytes. The exception: **topic subscription filters**.

A topic can have multiple subscriptions, each with its own SQL-style filter:

```
Topic: orders
  Subscription: gold-tier
    Filter: customer_tier = 'gold'
  Subscription: ap-south-region
    Filter: region = 'ap-south-1'
  Subscription: high-value
    Filter: order_total > 10000
```

When a message arrives, the broker reads `application_properties.customer_tier`, `application_properties.region`, etc. to evaluate each subscription's filter. Matching subscriptions get a copy.

This is why **business metadata that should drive routing belongs in Application Properties, not the body** — the broker can filter without decoding the payload.

## Real-world example

A multi-tenant SaaS sending invoice events:

```python
ServiceBusMessage(
    body=invoice_json,

    # Properties — AMQP standard
    message_id="invoice-evt-7741",
    subject="InvoiceGenerated",
    correlation_id="billing-run-2026-06",

    # Application Properties — app-specific custom dict
    application_properties={
        "tenant_id": "tenant-44",
        "region": "ap-south-1",
        "customer_tier": "gold",
        "invoice_amount_usd": 12_500,
        "trace_id": "abc-123",
    },
)
```

What the broker does with each piece:

| Field | Section | Broker behaviour |
|---|---|---|
| `message_id` | Properties | Reads for dedup |
| `subject` | Properties | Reads for filters that filter on `Subject` |
| `tenant_id` | Application Properties | Pass-through (no filter uses it here) |
| `region` | Application Properties | Reads if a subscription filters on region |
| `customer_tier` | Application Properties | Reads if a subscription filters on tier |
| `invoice_amount_usd` | Application Properties | Reads if a subscription filters on amount |

A subscription `Filter: customer_tier = 'gold' AND region = 'ap-south-1'` matches → gets a copy. A subscription `Filter: customer_tier = 'silver'` doesn't → skipped.

## Mental model

> **Application Properties is the sticky notes on the envelope.**
>
> [[Properties]] is the printed-on fields (To, From, RE:) the post office uses to deliver. **Application Properties is sticky notes** stuck to the envelope by the sender — *"VIP customer", "rush", "fragile contents"*. The post office doesn't read sticky notes by default.
>
> **Exception:** if the recipient set up a forwarding rule (*"only forward letters with a 'VIP' sticky"*), the post office will glance at the stickies to decide where to route. That's exactly Service Bus topic filters.
>
> The post office can't read your handwriting on the body of the letter, but it CAN read the stickies because they're in a fixed format. That's why business metadata that affects routing goes on the sticky, not in the letter.

## Interview answer

Application Properties is the open key/value section of an AMQP message — a dictionary the producer fills with whatever custom metadata the application needs (tenant IDs, regions, business categories, tracing context). Unlike [[Properties]], the keys are not standardised; the application defines them. The broker treats Application Properties as opaque pass-through except in one important case: Service Bus topic subscription filters can evaluate SQL-style expressions against Application Properties keys, which is the standard mechanism for routing messages to subscribers based on business metadata. This is why business attributes that should drive routing — `customer_tier`, `region`, `tenant_id` — belong in Application Properties rather than the message body: the broker can filter without decoding the body. Anything AMQP itself defines (`message-id`, `subject`, `reply-to`) belongs in [[Properties]] instead.

## Common misconceptions

- **"Application Properties is the same as Properties, just renamed."** No. Properties is a closed list of ~13 AMQP-standard fields. Application Properties is an open dict where the application defines all the keys. Different sections, different rules.
- **"The broker can't see Application Properties."** Mostly true (pass-through), but Service Bus topic filters CAN read them — that's the whole point of using them for routing metadata.
- **"I should put my big payload data here."** No. Application Properties is for small scalar metadata. Large content goes in the [[Body]]. Some brokers cap Application Properties size (Service Bus has a per-message size limit that includes them).
- **"Custom Service Bus fields like `PartitionKey` go here."** No. `PartitionKey`, `ScheduledEnqueueTime`, `LockedUntil` are *broker* extensions — they go in [[Message Annotations]] under `x-opt-`. Application Properties is for *application* extensions.
- **"I can put any value type."** Mostly. AMQP supports the standard scalar types (string, int, bool, uuid, timestamp, binary). Nested dicts and lists are technically allowed but poorly supported by tooling — keep values flat.
- **"Filter rules will see the body too."** No. Filters can only read Properties (`Subject`, `MessageId`, etc.), Header fields (`Priority`), and Application Properties. Never the body.

## See also

- [[Properties]] — the closed standard counterpart; AMQP-defined fields
- [[Message Annotations]] — vendor-specific broker extensions under `x-opt-` (different from app-specific Application Properties)
- [[Header]] — broker hot-path fields, also closed list
- [[AMQP Message Structure]] — the 6-section overview
- [[Body]] — where large payloads go (not Application Properties)

## Index

[[AMQP Message Structure]]
