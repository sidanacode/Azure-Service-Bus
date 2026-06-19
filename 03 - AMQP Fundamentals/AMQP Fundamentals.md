---
tags: [moc, section]
---

# AMQP Fundamentals

> Hub for section 3. Why we need a protocol on top of TCP, and what AMQP adds.

## What this section covers

[[TCP]] gave us reliable bytes. [[TCP Connections]] gave us a stable conversation. But bytes alone aren't messages. This section is the bridge from "bytes on a wire" to "messages with structure."

## Bridge from Section 2

Section 2 left us with reliable bytes. Three problems remain.

```mermaid
flowchart LR
    TCP[Section 2<br/>Reliable byte stream]:::prev
    Q1[Where does one<br/>message end?]:::gap
    Q2[Who are you?<br/>What version?]:::gap
    Q3[Did the app<br/>actually get it?]:::gap
    AMQP[Section 3<br/>AMQP fills the gaps]:::cur

    TCP --> Q1 --> AMQP
    TCP --> Q2 --> AMQP
    TCP --> Q3 --> AMQP

    classDef prev fill:#f0f0f0,stroke:#888,color:#222
    classDef gap fill:#fde2e2,stroke:#a83232,color:#5a0e0e
    classDef cur fill:#e8f1ff,stroke:#3b6fbd,color:#0a1f44
```

## Section flow

AMQP is a **message-oriented protocol on top of TCP**. It adds three things TCP doesn't give: message boundaries, identity/version, and message-level acknowledgement (settlement).

```mermaid
flowchart TB
    TCP[(TCP byte pipe<br/>ordered, lossless)]:::tcp

    subgraph AMQP[AMQP layer]
        direction TB
        MB[Message Boundaries<br/>length-prefix framing]:::box
        SEM[Messaging Semantics<br/>settlement = receiver's verdict]:::box
        WHY[Why AMQP exists<br/>fills 3 gaps TCP leaves open]:::box
    end

    APP[Application<br/>sees discrete MESSAGES,<br/>not bytes]:::app

    TCP --> AMQP --> APP

    classDef tcp fill:#d6f5e0,stroke:#2f7a4f
    classDef box fill:#e8f1ff,stroke:#3b6fbd
    classDef app fill:#fff3d6,stroke:#b07f1f
```

**The three framing schemes** AMQP could have picked, and why it picked length-prefix:

```mermaid
flowchart LR
    DATA[Bytes on wire]:::data
    DATA --> D[Delimiter<br/>'\n' between msgs]:::bad
    DATA --> L[Length prefix<br/>'4 bytes = N, then N bytes']:::good
    DATA --> F[Fixed size<br/>every msg is 1KB']:::bad

    D -. dies on binary collisions .-> X1[❌]:::reject
    F -. wastes space, varies poorly .-> X2[❌]:::reject
    L -. binary-safe + variable size .-> Y[✅ AMQP's choice]:::accept

    classDef data fill:#f5f5f5,stroke:#666
    classDef bad fill:#fde2e2,stroke:#a83232
    classDef good fill:#d6f5e0,stroke:#2f7a4f
    classDef reject fill:#fff,stroke:#a83232,color:#a83232
    classDef accept fill:#fff,stroke:#2f7a4f,color:#2f7a4f
```

**HTTP vs AMQP — same TCP, different *shape* of work.** This is a shape difference, not a speed difference:

```mermaid
flowchart LR
    subgraph HTTP[HTTP — synchronous]
        direction LR
        HC[Client] <-->|both alive at once| HS[Server]
    end

    subgraph AMQPSHAPE[AMQP — asynchronous]
        direction LR
        AP[Producer] -->|fire and forget| AB[(Broker)]
        AB -->|push or poll| ACO[Consumer]
    end

    classDef default fill:#e8f1ff,stroke:#3b6fbd
```

The five consequences fall out of the shape: temporal decoupling, fan-out, push vs poll, credits/backpressure, two-stage settlement. None of these work in pure HTTP without bolting on a queue.

## Notes (in order)

- [[Why AMQP Exists]] — the gaps TCP leaves open and how AMQP fills them
- [[AMQP vs TCP]] — message-oriented on top of byte-oriented
- [[Message Boundaries]] — where one message ends and the next begins
- [[Messaging Semantics]] — what AMQP promises about delivery
- [[AMQP vs HTTP]] — same TCP, different shape of work

## Where this fits

Section **3 of 11**. Section 2 was the wire (TCP). Section 4 is AMQP's transport layer (connections, sessions, links, frames).

[[Index]]
