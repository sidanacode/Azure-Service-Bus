---
tags: [fundamentals, role]
---

# Broker

> The **broker** is a long-running program with disks attached. It accepts messages from producers, saves them, holds them safely, and hands them to consumers when ready.

## Interview-ready answer

A broker is a long-running server program with attached storage that sits between producers and consumers. It accepts messages, writes them to disk for durability, and holds them until a consumer is ready to process them. The broker tracks every message through two acknowledgements — one from the producer when it accepts the message, and one from the consumer when it confirms processing — and only deletes a message after the second ack. This design decouples producers and consumers in time, identity, and load, and removes the need for each service to build its own retry, timeout, and fallback logic.

## Why does it exist?

A producer wants to send a message. A consumer wants to read it.

The easy answer is direct connection:

```
[Producer]  ──── bytes ────►  [Consumer]
```

But this breaks the moment something goes wrong:

- Consumer is down for maintenance → message is lost
- Consumer is slow → producer waits
- Producer doesn't know the consumer's address → can't send
- Two consumers want the same message → producer has to send twice
- Consumer crashes mid-read → was the message delivered or not?

Direct connection forces both sides to be alive at the same time. It also forces the producer to know too much about the consumer.

We need something in the middle. A safe holding place.

That place is the **broker**.

## What is a broker?

A broker is just **a program running on a server**.

It has a process ID. It runs on a real computer in a real datacenter.

Two things make it special:

1. **It has disks attached.** Messages saved to disk survive restarts.
2. **It accepts many connections at once.** Producers and consumers all connect to it.

Azure Service Bus, RabbitMQ, and Kafka are all brokers. Same idea: a long-running program with disks and a network port.

## What does it do?

Three jobs.

### Job 1 — Accept and store messages

A producer pushes bytes. The broker:

1. Receives the bytes
2. Writes them to disk
3. Sends back *"got it"*

The "writes to disk" part is critical. If the broker just held the message in memory and crashed, the message would be gone.

This is called **durability**.

### Job 2 — Hold messages until a consumer is ready

The message sits on disk. It waits.

The producer is gone. It's done its job.

The consumer might not be running yet. That's fine.

When a consumer finally connects and asks for messages, the broker reads from disk and sends them.

### Job 3 — Track who got what

The broker remembers:

- Which messages have been delivered
- Which ones have been confirmed processed
- Which ones are still waiting

This is so it can delete messages when they're done — and re-send them if a consumer crashed.

## The two acknowledgements

This is the most important idea. Lock it in.

A message has **two ack moments** in its life:

```
[Producer]                [Broker]                  [Consumer]
    │                        │                          │
    │  ──── message ────►    │                          │
    │                        │  (writes to disk)        │
    │  ◄──── "got it" ────   │                          │
    │     ↑                  │                          │
    │   ACK #1               │                          │
    │                        │  ──── message ────►      │
    │                        │                          │  (does work)
    │                        │  ◄──── "done" ────       │
    │                        │     ↑                    │
    │                        │   ACK #2                 │
    │                        │  (deletes message)       │
```

- **ACK #1** — Producer → Broker. *"I have your message saved. You can forget it."*
- **ACK #2** — Consumer → Broker. *"I have processed this message. You can delete it."*

The broker holds the message **between** these two acks.

Almost everything later — peek-lock, complete, abandon, dead letter, settlement — is just **different rules for ACK #2**.

## Why brokers beat direct calls

The protocol doesn't matter. HTTP, gRPC, raw TCP, REST, GraphQL — they all have the same problem when used for direct service-to-service calls.

If service A calls service B directly, **service A** has to manage:

- Retries when B is down
- Timeouts when B is slow
- Fallback logic when B fails
- Tracking what's been sent
- Coordinating delivery

Every service that talks to another service has to build all this **itself**. Multiply this across dozens of services and it becomes a nightmare to maintain.

A broker absorbs all of it into **one place**. The services just send messages and forget. The broker handles reliability for everyone.

| Without broker | With broker |
|----------------|-------------|
| Both sides must be alive | Either side can be down |
| Producer knows consumer's address | Producer only knows broker |
| Lost on crash | Survives crash (durability) |
| One-to-one only | One-to-many easily |
| Each service builds its own retries | Broker handles it once |

## A real example

A user clicks "Place Order".

The web server sends `OrderPlaced` to the broker.
The broker writes it to disk and replies *"got it"*.
The web server tells the user "Order placed!" and moves on.

Five minutes later, the payments service starts up.
It connects to the broker and asks: *"any messages for me?"*
The broker reads `OrderPlaced` from disk and sends it.
Payments processes the message and replies *"done"*.
The broker deletes the message.

The web server and the payments service **never spoke directly**. They never even had to be running at the same time.

## Mental model

**A package locker at an apartment building.**

The delivery driver (producer) drops a package in the locker and gets a receipt — they leave immediately, not waiting for you. The locker holds the package safely on its shelves (durability) for as long as needed. When you (consumer) come home, you open the locker with your code, take your package, and the locker logs that it was picked up. The driver and the resident never had to be there at the same time. The locker absorbs all the timing problems — and one locker serves the whole building, so each driver doesn't need to coordinate with each resident individually.

## Common misconceptions

- **"The broker pushes messages to consumers."** No. Consumers **pull**. The broker waits to be asked. This lets each consumer control its own pace.
- **"The broker deletes a message after the producer's ack."** No. It deletes only after the **consumer's** ack (ACK #2). Between the two acks, the message is held safely.
- **"A broker is just a database."** No. A database stores rows you query at will. A broker is a *queue* with delivery semantics, locks, redelivery, and dead-lettering. Different shape, different job.
- **"Without a broker, services can just call each other."** They can — until something fails. The broker absorbs retries, timeouts, and outages once on everyone's behalf, instead of every service rebuilding that machinery.

## See also

- [[The Raw Substrate]]
- [[Producer]]
- [[Messages]]
- [[Consumer]]

## Index

[[Messaging Fundamentals]]
