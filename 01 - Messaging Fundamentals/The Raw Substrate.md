---
tags: [fundamentals, mental-model]
---

# The Raw Substrate

> The picture to hold in your head before any new word like "broker" or "queue" appears.

## Interview-ready answer

When two programs run as separate processes, the OS isolates them — they cannot share memory or call each other's functions. The only way they can interact is by pushing bytes through a network card. Every messaging system, including Azure Service Bus, is built on top of this raw fact: programs talk by sending bytes over a wire. All higher-level concepts — TCP, AMQP, queues, settlement — exist because raw bytes alone are not enough to coordinate real business work.

## Two programs want to talk

Imagine two programs running on two computers.

Each one is a separate process. Each has its own memory. Neither can see the other.

They are completely isolated by the operating system.

So how do they talk to each other?

There is only one way: **push bytes through a network card.**

That's it. That's the only tool they have.

## What "sending a message" really is

```
[Program A]  ──── bytes through a wire ────►  [Program B]
```

There is no magic here. Just:

- A network card on each side
- A wire between them
- Bytes flowing across

Everything in this whole syllabus — TCP, AMQP, Service Bus, queues — is built on top of this.

Why? Because raw bytes on a wire are a terrible way to run a business.

## The three programs

We will keep coming back to this picture:

```
[Program A]              [Program B]                 [Program C]
your code                Microsoft's broker          your code
                         + disks attached
       │                       │                          │
       │  ── bytes ──►         │      ◄── bytes ──        │
       │                       │                          │
```

When a future note says *"the producer sends a message to a broker"*, it really means:

> Program A opens a TCP connection to Program B and pushes bytes.
> Program B writes those bytes to a disk.
> Later, Program C asks Program B for bytes. Program B reads from disk and sends them.

**Three programs. Two wires. Bytes on the wires.**

## The big question

> How do we make pushing bytes between programs reliable enough to build a business on it?

Every concept after this answers part of that question.

## See also

- [[Producer]]
- [[Messages]]

## Index

[[Messaging Fundamentals]]
