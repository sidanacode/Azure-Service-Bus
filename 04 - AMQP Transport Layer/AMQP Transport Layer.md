---
tags: [moc, section]
---

# AMQP Transport Layer

> Hub for section 4. The four layers AMQP uses to multiplex many conversations over one TCP connection.

## What this section covers

Section 3 left us with one producer, one broker, one TCP connection, length-prefixed messages flowing safely. Section 4 breaks that picture, because real apps run **many conversations at once** through a single broker — orders, audit, replies, notifications — and need a way to keep them isolated without opening a TCP connection per conversation.

The answer is multiplexing, organised in three nested layers — Connection, Session, Link — that ride on top of TCP, with Frames as the unit of bytes that carries the multiplexing tag.

## Bridge from Section 3

Section 3 gave us length-prefixed messages on one TCP socket — but only **one conversation per socket**. Real apps need many.

```mermaid
flowchart LR
    S3[Section 3<br/>1 socket = 1 message stream]:::prev
    P[Real app needs:<br/>orders + audit + replies<br/>at the same time]:::need
    S4[Section 4<br/>multiplex many conversations<br/>over one socket]:::cur

    S3 --> P --> S4

    classDef prev fill:#f0f0f0,stroke:#888,color:#222
    classDef need fill:#fde2e2,stroke:#a83232,color:#5a0e0e
    classDef cur fill:#e8f1ff,stroke:#3b6fbd,color:#0a1f44
```

## Section flow — the four layers

Three nested layers ride on top of TCP. Frames are the byte units that carry the multiplexing tags.

```mermaid
flowchart TB
    TCP[(TCP socket<br/>ordered byte stream)]:::tcp

    subgraph CONN[Connection — the front door]
        direction TB
        CON_DESC[1 per TCP socket<br/>auth · version · heartbeat · max-frame-size]:::desc
    end

    subgraph SESS[Session — a meeting room]
        direction TB
        SESS_DESC[channel = which Session<br/>notebook of deliveries<br/>own flow-control window<br/>own error scope]:::desc
    end

    subgraph LNK[Link — a one-way pipe]
        direction TB
        LNK_DESC[handle = which Link<br/>direction sender or receiver<br/>fixed target/source queue<br/>own credit pool]:::desc
    end

    FR[/TRANSFER frame<br/>channel + handle + delivery-id/]:::frame

    TCP --> CONN --> SESS --> LNK --> FR

    classDef tcp fill:#d6f5e0,stroke:#2f7a4f
    classDef desc fill:#fff,stroke:#3b6fbd,color:#0a1f44
    classDef frame fill:#fff3d6,stroke:#b07f1f
```

## Routing on every TRANSFER frame

Three small integers, three layers of routing. Read them top-down to find the destination.

```mermaid
flowchart LR
    FR[/Frame on the wire/]:::frame
    CH[channel = 1<br/>→ Session #1]:::ch
    H[handle = 0<br/>→ Link 'orders sender']:::h
    D[delivery-id = 4721<br/>→ this specific message]:::d
    Q[(orders-queue)]:::q

    FR --> CH --> H --> D --> Q

    classDef frame fill:#fff3d6,stroke:#b07f1f
    classDef ch fill:#e8f1ff,stroke:#3b6fbd
    classDef h fill:#d6f5e0,stroke:#2f7a4f
    classDef d fill:#f0e0ff,stroke:#7a3fb0
    classDef q fill:#fff,stroke:#222
```

## Backpressure path (the credits story)

When the broker can't keep up, it just stops granting credits — and pressure flows backward all the way to the application code, automatically.

```mermaid
flowchart RL
    APP[Application code<br/>blocks on send]:::app
    LIB[Producer library<br/>send buffer fills]:::lib
    LINK[Link credits = 0<br/>cannot transmit]:::link
    BR[(Broker overwhelmed<br/>stops sending FLOW)]:::broker

    BR -.->|no credits granted| LINK
    LINK -.->|can't send| LIB
    LIB -.->|buffer full| APP

    classDef app fill:#fde2e2,stroke:#a83232
    classDef lib fill:#fff3d6,stroke:#b07f1f
    classDef link fill:#e8f1ff,stroke:#3b6fbd
    classDef broker fill:#d6f5e0,stroke:#2f7a4f
```

## The four layers, summarised

```
TCP socket          ──  ordered, lossless byte stream  (the OS provides this)
 └── Connection     ──  one per socket; auth, version, heartbeat, max-frame-size
      └── Session   ──  conversation context; channel# scoped per Connection;
      │                 notebook of deliveries (delivery-id, settlement state)
      │                 own flow-control window, own error scope
      │
      └── Link      ──  one-way pipe; handle# scoped per Session;
                        fixed direction (sender / receiver),
                        fixed address (target / source),
                        own credit pool for fine-grained flow control
```

Every TRANSFER frame on the wire carries three identifiers, one per layer:

```
(channel = which Session) → (handle = which Link) → (delivery-id = which message)
```

That is the whole multiplexing story: three nested namespaces, each with its own scope, each solved with one small integer per frame.

## Frame types across the section

The verbs (performatives) that appear in this section's frame bodies:

| Performative | Layer | Opens / closes / does |
|---|---|---|
| `OPEN` / `CLOSE` | Connection | Open / close the Connection |
| `BEGIN` / `END` | Session | Open / close a Session |
| `ATTACH` / `DETACH` | Link | Open / close a Link |
| `TRANSFER` | Link | Carry a message (or part of one) |
| `DISPOSITION` | Session | Settle a delivery (accepted/rejected/released/modified) |
| `FLOW` | Link / Session | Update credits or window state |

Header is always the same shape (size, DOFF, type, channel). The body's performative is what varies.

## Notes (in order)

- [[Connection]] — the front door: one per TCP socket, owns auth/version/heartbeat
- [[Frames]] — what AMQP actually writes to TCP, channel-tagged for routing
- [[Session]] — a conversation context, the notebook of deliveries
- [[Link]] — a one-way pipe inside a Session for actual message flow

## Where this fits

Section **4 of 11**. Section 3 explained why AMQP exists. This section explains how AMQP organises a single TCP pipe into many isolated conversations. Section 5 (Message Transfer) builds on top to show how a single message moves through Link → Session → Frames → wire and back.

[[Index]]
