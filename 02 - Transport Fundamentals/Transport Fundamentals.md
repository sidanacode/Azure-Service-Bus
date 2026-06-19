---
tags: [moc, section]
---

# Transport Fundamentals

> Hub for section 2. The wire underneath everything.

## What this section covers

Before AMQP makes sense, we need to be solid on the floor it stands on — TCP. This section answers two things:

1. What does TCP actually deliver?
2. What does TCP **not** deliver, and why does that matter for AMQP?

## Bridge from Section 1

[[Messaging Fundamentals|Section 1]] talked about producer, broker, consumer as if they were just *talking* — no detail on **how the bytes actually move**. That handwave is what Section 2 fills in.

```mermaid
flowchart LR
    A[Section 1<br/>Producer → Broker → Consumer]:::prev
    B[Section 2<br/>How do the bytes get there?]:::cur

    A -->|wire was a black box| B

    classDef prev fill:#f0f0f0,stroke:#888,color:#222
    classDef cur fill:#e8f1ff,stroke:#3b6fbd,color:#0a1f44
```

## Section flow

TCP gives you a **byte pipe**: ordered, lossless, no duplicates. But it gives you *only* bytes — no message boundaries, no auth, no acks at the message level.

```mermaid
flowchart TB
    APP[Application data<br/>'I am a message']:::app
    TCP{{TCP<br/>handshake → byte stream → teardown}}:::tcp
    NET[Unreliable network<br/>loss, reorder, duplication]:::net
    OUT[Receiver<br/>gets bytes, in order, no gaps]:::ok

    APP -->|write| TCP
    TCP -->|chops into packets| NET
    NET -->|TCP retries / reorders| TCP
    TCP -->|reassembles| OUT

    classDef app fill:#e8f1ff,stroke:#3b6fbd
    classDef tcp fill:#d6f5e0,stroke:#2f7a4f
    classDef net fill:#fde2e2,stroke:#a83232
    classDef ok fill:#fff3d6,stroke:#b07f1f
```

**What TCP gives you:** ordered, no-loss, no-duplicate **bytes**.
**What TCP does *not* give you:** where one message ends and the next begins, who you're talking to (auth), or whether the *application* on the other end actually got it.

That gap is exactly why [[AMQP Fundamentals|Section 3]] exists.

## Notes (in order)

- [[TCP]] — reliable byte transport; what it gives, what it doesn't
- [[Reliable Byte Transport]] — what "reliable" really means (the four failure modes, head-of-line blocking, the speed tradeoff)
- [[TCP Connections]] — handshake, state, teardown

## Where this fits

Section **2 of 11**. Section 1 was messaging concepts. Section 3 is AMQP — which exists to fill the gaps TCP leaves open.

[[Index]]
