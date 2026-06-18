---
tags: [amqp, fundamentals]
---

# AMQP vs HTTP

> **HTTP is built for "ask and answer." AMQP is built for "send and let someone else handle it later."** Same TCP underneath, completely different shape of work.

## Definition

HTTP and AMQP are both application protocols layered on TCP. They are not competing implementations of the same idea — they're built for **different shapes of work**:

- **HTTP** = synchronous request/response. Both sides must be alive at once.
- **AMQP** = asynchronous fire-and-forget. The receiver can be offline; a broker holds the message until they're ready.

Choosing between them is not a performance decision. It's a shape decision.

## Problem it solves

A fair, sharp question after learning AMQP: *"HTTP also runs on TCP, also adds framing, also has metadata in headers — why didn't whoever invented AMQP just use HTTP?"*

There must be **something HTTP can't do that messaging needs**. This note pins down what.

The answer is not *"HTTP is older / slower / less features."* HTTP works wonderfully for what it was built for. The answer is that **HTTP was built for the wrong shape of work** — and trying to do messaging over HTTP forces you to reinvent AMQP, badly.

## Why previous solution was insufficient

HTTP works fine for low-volume, one-to-one, ask-and-wait jobs. It breaks the moment any of these are true:

- The receiver might be offline when the sender wants to send
- The same message needs to reach multiple receivers
- The receiver wants to be pushed events instead of polling for them
- The receiver can't keep up and needs to push back on the sender
- The sender needs to know the receiver took *responsibility* (not just received bytes)

Each of these has an HTTP workaround (long polling, multipart, retry-with-backoff). All of them are reinventing parts of AMQP on top of HTTP, and every team builds them differently. So someone built it properly, once. That's AMQP.

## Responsibilities

To pin down the difference, walk one event through both protocols.

A web server processes an order. Downstream services (payments, shipping, email, analytics) need to know.

### Path 1 — HTTP

```python
requests.post("payments-service/charge", body)
requests.post("shipping-service/schedule", body)
requests.post("email-service/send", body)
requests.post("analytics-service/track", body)
```

What this requires:

- All four services running **right now**
- The web server knowing every URL
- The web server handling retries, timeouts, and partial failures itself
- The web server's code changing every time a new service wants to react

### Path 2 — AMQP

```python
sender.send_message(ServiceBusMessage(subject="OrderPlaced", body=...))
```

What this requires:

- The broker running
- That's it

The broker fans the message out to every subscribed consumer — even ones added tomorrow — and holds the message safely if any consumer is offline. The web server's code never changes.

## The five concrete differences

Each one falls out of the shape difference (sync vs. async). Together they're the whole reason AMQP exists as a separate protocol.

### 1. Sender and receiver alive at the same time?

| HTTP | AMQP |
|---|---|
| Both required, simultaneously | Only sender + broker required |
| Receiver offline → request fails | Receiver offline → broker holds the message |

This is **temporal decoupling**. The producer and consumer don't share a moment in time.

### 2. How many receivers per message?

| HTTP | AMQP |
|---|---|
| One — bound by URL | Zero, one, or many — broker fans out |
| Adding receivers means changing the sender | Adding receivers means subscribing to the broker |

This is **fan-out**. New consumers can be added without touching the producer.

> If you tried to fix this with HTTP by writing a small service that takes one POST and re-sends to all subscribers, you would be **rebuilding a broker**.

### 3. Push or poll?

| HTTP | AMQP |
|---|---|
| Poll — receiver keeps asking *"any new?"* | Push — broker delivers when there's something |
| Wasteful at scale (mostly empty answers) | Near-zero idle traffic |

HTTP can't push by design. The workarounds — long polling, WebSockets, Server-Sent Events — all *abandon* HTTP's shape to get push. AMQP is push from the start.

### 4. Backpressure (slow consumers)

| HTTP | AMQP |
|---|---|
| No native flow control | **Credits** — receiver tells sender how many it can take |
| Burst of requests overwhelms slow consumer | Broker stops pushing when credits run out |

A consumer processing 10 messages/sec, hit with 1000 messages/sec:

- **HTTP** — bombarded. Times out, drops requests, OOMs.
- **AMQP** — issues 100 credits, broker pushes 100, consumer processes them, credits more. Producer never overwhelms.

### 5. Settlement vs. `200 OK`

| HTTP | AMQP |
|---|---|
| Single `200 OK` — mixes "received bytes" and "processed work" | Two settlements — separate transport and application acks |
| One outcome (success / failure) | Four outcomes (Accepted / Rejected / Released / Modified) |

`200 OK` cannot express *"I have your message safely, but haven't processed it yet"* — exactly the producer→broker handoff messaging needs. To get two-stage acks over HTTP, every team invents its own scheme:

```
GET    /orders/next       → fetch
DELETE /orders/123        → "I'm done"
POST   /orders/123/retry  → "I failed, give it back"
```

That's [[Messaging Semantics|settlement]] reinvented, badly, with no shared standard.

## Mental model

**HTTP is a phone call. AMQP is voicemail.**

- A phone call needs both people to pick up at the same time. Conversation is fast and direct, but if they're not there, it fails.
- Voicemail lets you leave a message and move on. They listen later — could be a minute, could be tomorrow. You don't even know who exactly will listen; the message just sits in the inbox until played.

Both are valid ways to communicate. They're shaped for different problems. You don't replace your phone with voicemail or vice versa — you use whichever fits.

## Real-world example

**HTTP shape** — checking your bank balance:

```
You ──► "What's my balance?" ──► Bank
You ◄── "$1,234.56"          ◄── Bank
```

You wait for the answer. The bank must be online. The interaction is one-to-one. `200 OK` is enough — you have the answer or you don't. ✅ Right tool.

**AMQP shape** — placing an order:

```
You ──► "OrderPlaced{id:123}" ──► broker
                                    ├──► [Payments]    (charges card)
                                    ├──► [Shipping]    (schedules delivery)
                                    ├──► [Inventory]   (reduces stock)
                                    ├──► [Email]       (sends confirmation)
                                    └──► [Analytics]   (updates dashboards)
```

You move on the second the broker has the order. You don't know how many services react. Some might be down right now and pick it up later. Each does its own work at its own pace. ✅ Right tool.

Doing the second pattern with HTTP would force the web server to know every downstream service, retry each one independently, handle partial failures, slow down when any of them is overloaded — *that's reinventing the broker.*

## Where each shows up

| Shape | Examples |
|---|---|
| HTTP-shaped (synchronous, ask + wait) | Web pages, REST APIs, GraphQL queries, RPC, "give me the user", "log me in" |
| AMQP-shaped (async, fire + forget) | Service Bus, RabbitMQ, ActiveMQ, Kafka (its own protocol but same shape), order events, payment processing, audit logs |

Some hybrids exist — AWS SQS and webhooks do messaging over HTTP. They work for low-volume cases but inherit HTTP's shape limits (polling, no push, weak settlement).

## Interview answer

HTTP and AMQP are both application protocols on TCP, but they're built for opposite shapes of work. HTTP is synchronous request/response — both sender and receiver must be alive at the same moment, the message goes to exactly one URL-bound receiver, and the receiver acknowledges with a single `200 OK`. AMQP is asynchronous fire-and-forget — the producer publishes to a broker that holds the message durably until consumers are ready, the same message can fan out to many consumers, the receiver pushes back on overload via credits, and acknowledgement is a two-stage settlement with four possible outcomes. You can do messaging over HTTP for small workloads, but anything serious — push delivery, two-stage acks, fan-out, backpressure — forces you to reinvent AMQP on top of HTTP. So someone built it properly. The choice between them isn't about modernness or speed; it's about which shape fits the work.

## Common misconceptions

- **"AMQP is just slow HTTP."** No. They're not the same shape. HTTP is sync, AMQP is async. AMQP isn't trying to do what HTTP does at all. Speed isn't the differentiator — shape is.
- **"Use HTTP for queries, AMQP for commands."** Closer, but not the rule. Use **HTTP for ask-and-wait**, **AMQP for fire-and-forget plus durability**. Some commands fit each side depending on whether the sender needs to wait.
- **"WebSocket replaces AMQP."** No. WebSocket gives you a persistent two-way channel — a *transport*. It has no settlement, no flow control, no broker, no durability, no fan-out. AMQP would be built on top of a WebSocket-like pipe, not replaced by it.
- **"AMQP is for high volume; HTTP is for low volume."** Volume is correlated, not causal. The reason AMQP scales is its *shape* — push, async, credits, fan-out. A low-volume async event fits AMQP perfectly even if you only publish ten times a day.
- **"HTTP/2 with multiplexing replaces AMQP."** No. HTTP/2 fixes HTTP/1.1's connection-per-request problem, but the shape is still request/response. The receiver still has to be alive when you ask. Multiplexing speeds HTTP up; it doesn't make HTTP async.

## Design tradeoff

**Why two protocols, not one:** the cost of adding messaging features (settlement, credits, fan-out, durability) to HTTP would distort HTTP into something HTTP shouldn't be — and existing HTTP clients/servers wouldn't speak the new dialect. Building a separate protocol designed for async messaging from day one keeps each protocol simple.

**The advantage:** developers pick the shape that fits the work. A REST API stays clean and synchronous. A messaging system stays clean and asynchronous. No layer is doing a job it wasn't built for.

**The cost:**

1. **Two protocols to learn and operate.** Your team needs HTTP literacy *and* AMQP literacy.
2. **Two infrastructures to monitor.** REST endpoints have their own health checks; brokers have queue depth, dead-letter alarms, settlement lag.
3. **More moving parts.** A pure HTTP architecture has fewer pieces. Adding a broker introduces something else that can break and that you must run, scale, and patch.

For most non-trivial systems, that overhead is worth it — the alternative is squeezing async work through a sync protocol and paying for it in tight coupling, polling traffic, and reinvented settlement schemes.

## See also

- [[Why AMQP Exists]] — the safety gap above TCP
- [[AMQP vs TCP]] — the layering relationship
- [[Message Boundaries]] — framing schemes (length prefix vs. delimiters)
- [[Messaging Semantics]] — settlement, the deep difference
- [[Broker]] — the piece that makes async possible

## Index

[[AMQP Fundamentals]]
