---
tags: [fundamentals, message-type]
---

# Command

> A **command** is a message that says **"do this thing"**. It is an instruction sent to one specific receiver.

## Interview-ready answer

A command is a message that instructs another service to perform an action. It is sent to exactly one receiver, and the sender expects something to happen as a result. Commands are named with verbs like `ChargeCard`, `SendEmail`, or `CancelOrder` because they describe an action the receiver must take. Commands can succeed or be rejected — for example, a `ChargeCard` may fail if the card is declined.

## Why does it exist?

Not all messages are the same kind. They differ by **intent** — what the sender wants.

Sometimes the sender wants something to happen:

> *"Charge this card."*
> *"Send this email."*
> *"Delete this user."*

The sender is asking another service to **do something**.

That kind of message is a command.

## What is a command?

A command says: **"do this thing."**

It is an instruction. The producer is telling another service to perform an action.

Examples:

- `ChargeCard`
- `SendWelcomeEmail`
- `CancelOrder`
- `CreateUser`
- `RefundPayment`

Notice the names. They are **verbs** — *charge, send, cancel, create, refund*.

That's how you spot a command. The name tells someone to do something.

## Key properties

### One sender, one receiver

A command is meant for **one** specific service to act on.

You don't broadcast a command. You send it to the one place that knows how to handle it.

`ChargeCard` goes to the payments service. Only the payments service charges cards.

### The sender expects something to happen

When you send a command, you are saying *"this needs to be done."*

The system has failed if no one acts on it.

### Commands can be rejected

The receiver might refuse:

- *"Card was declined."*
- *"User already exists."*
- *"Order is already cancelled."*

This is normal. Commands have outcomes.

## In code

```python
ServiceBusMessage(
    body='{"cardId": "abc-123", "amount": 99.00}',
    subject="ChargeCard"
)
```

The `subject` is the command name. The receiver reads the subject and knows what to do.

## A real example

The web server accepts an order. It needs payments to charge the card.

It sends a `ChargeCard` command to the payments queue.

```
[Web Server] ──── ChargeCard ────► [Queue] ────► [Payments Service]
                                                       │
                                                       ▼
                                                 (charges card)
```

The payments service:

1. Reads the message
2. Charges the card
3. Confirms back to the broker

The web server told payments **what to do**. That's a command.

## The shape to remember

> A command says **"do this."**
>
> It is sent to **one** specific receiver.
>
> The sender wants an action to happen.

## Mental model

**A work order at a repair shop.**

A work order is addressed to one specific mechanic (one receiver), names the job that needs doing (`ReplaceBrakes`), and expects an outcome — the work either gets done or comes back rejected ("part not in stock"). Nobody else picks up the work order; it has one intended hands. Commands are work orders for services.

## Common misconceptions

- **"A command is just a message."** True technically — but the **intent** matters. A command says *"do this"* and is sent to one specific service. Calling an event a command (or vice versa) leads to broken designs.
- **"Commands always succeed."** No. They can be rejected — declined card, duplicate user, invalid input. The handler must report success or failure.
- **"Commands and HTTP POSTs are the same."** Similar shape (one sender, one receiver, action requested), but a command is **asynchronous** — the producer doesn't wait. An HTTP POST is synchronous. The shape of work is different.

## See also

- [[Messages]]
- [[Event]]
- [[Notification]]
- [[Query]]
- [[Producer]]
- [[Consumer]]

## Index

[[Messaging Fundamentals]]
