---
tags: [fundamentals, message-type]
---

# Event

> An **event** is a message that says **"this happened"**. It is a fact, announced to anyone who cares to listen.

## Interview-ready answer

An event is a message that announces a fact — something that has already happened. The producer publishes the event without knowing or caring who listens. Events are named in the past tense, like `OrderPlaced` or `PaymentCleared`, and can be received by zero, one, or many subscribers. Unlike commands, events cannot be rejected — you can't refuse a fact — and they enable loose coupling, since new services can react to existing events without the producer changing.

## Why does it exist?

A [[Command]] says *"do this."* But that's not the only kind of intent a message can carry.

Sometimes the sender doesn't want anything done. They just want to **announce that something happened**.

> *"An order was placed."*
> *"A user signed up."*
> *"A payment cleared."*

The sender is reporting a fact. They don't care who hears it. They don't care what anyone does about it.

That kind of message is an event.

## What is an event?

An event says: **"this happened."**

It is a **fact**. Past tense. Already real.

The producer is telling the world: *"here's something that occurred. Do whatever you want with this information."*

Examples:

- `OrderPlaced`
- `UserRegistered`
- `PaymentCleared`
- `EmailSent`
- `OrderCancelled`

Notice the names. They are **past tense verbs** — *placed, registered, cleared, sent, cancelled*.

That's how you spot an event. The name describes something that **already happened**.

## Key properties

### One sender, many receivers (or zero)

An event can be heard by anyone interested. Or by no one.

The sender doesn't know and doesn't care.

`OrderPlaced` might be read by:

- The payments service
- The shipping service
- The analytics service
- The email service

All of them. None of them. The sender doesn't care.

### The sender expects nothing back

When you publish an event, you are not asking for anything. You're just announcing.

The system hasn't failed if no one reacts. The event still happened.

### Events cannot be rejected

A command can be refused. An event cannot.

You can't refuse a fact. `OrderPlaced` already happened. The receiver might choose not to act on it, but they can't say *"no, the order wasn't placed."*

### Events are immutable

Once published, an event represents a fact at a moment in time. It doesn't change.

If something changes later, that's a **new event**.

If an order gets cancelled, you don't edit the `OrderPlaced` event. You publish a new `OrderCancelled` event.

## In code

```python
ServiceBusMessage(
    body='{"orderId": 123, "amount": 99}',
    subject="OrderPlaced"
)
```

Same shape as a command. The difference is the **name** and the **intent**, not the message structure.

## Command vs Event

| | Command | Event |
|---|---------|-------|
| **Says** | *"do this"* | *"this happened"* |
| **Tense** | imperative (charge, send) | past tense (charged, sent) |
| **Receivers** | exactly one | zero or many |
| **Sender expects** | action to happen | nothing |
| **Can be rejected?** | yes | no |
| **Goes to** | usually a queue | usually a topic |

Both are messages. The difference is **intent**.

## A real example

A user clicks "Place Order". The web server saves it.

Then the web server publishes `OrderPlaced`.

```
                                  ┌──► [Payments Service]   (charges card)
[Web Server] ──► OrderPlaced ─────┼──► [Shipping Service]   (schedules delivery)
                                  ├──► [Analytics Service]  (updates dashboard)
                                  └──► [Email Service]      (sends confirmation)
```

Each service gets its own copy. They all react in their own way.

The web server doesn't know how many services are listening. It doesn't care.

Tomorrow, a fraud-detection service can be added. It subscribes to `OrderPlaced` and starts checking for suspicious orders. **The web server's code doesn't change.**

That's the power of events. **Loose coupling.**

## The shape to remember

> An event says **"this happened."**
>
> It is sent to **zero or many** listeners.
>
> The sender expects **nothing** in return.

## Mental model

**A newspaper headline on a public board.**

The newspaper announces *"Election Result: Candidate A Wins"* and posts it on a public board. Anyone can read it. Some people care, some don't. Some act on it (traders, analysts, fans); others ignore it. The newspaper doesn't know or care who reads. The headline is a fact — it can't be undone. If the result later changes, that's a *new* headline (`ResultRevised`), not an edit. Events are public-board headlines for the system.

## Common misconceptions

- **"Events are commands without a target."** No. The intent is different. A command requires action; an event is just a fact. Phrasing a command as `OrderShouldBePaid` doesn't make it an event.
- **"If no one is listening, the event is wasted."** No. The event still represents a fact. New consumers can subscribe later, even retroactively (in some systems), and act on past events.
- **"Events should be edited if the underlying data changes."** No. Events are immutable. A change is a new event — `OrderCancelled`, `OrderUpdated`. This is the whole basis of event sourcing.
- **"Events guarantee delivery."** Not by themselves. Whether an event reaches every subscriber depends on the broker's [[Delivery Semantics|delivery semantics]] — usually at-least-once with [[Idempotency|idempotent]] consumers.

## See also

- [[Messages]]
- [[Command]]
- [[Notification]]
- [[Query]]
- [[Producer]]
- [[Consumer]]

## Index

[[Messaging Fundamentals]]
