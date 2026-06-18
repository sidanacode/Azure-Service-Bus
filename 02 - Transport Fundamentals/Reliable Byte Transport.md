---
tags: [transport, fundamentals]
---

# Reliable Byte Transport

> **Reliable byte transport** is the contract that says: bytes you send arrive at the other end in order, intact, exactly once, and without gaps — or the connection breaks and you find out.

## Interview-ready answer

Reliable byte transport is the guarantee layer that turns the unreliable internet into a clean ordered byte stream. The raw network loses, reorders, duplicates, and corrupts packets. Without a reliability layer, every application would have to detect and fix all four problems on its own, so the OS solves it once on everyone's behalf — that's what TCP implements. The cost is real: retransmits introduce latency, the sender holds copies of unacked bytes in memory, and a single lost packet can stall every byte queued behind it (head-of-line blocking). For business work like an `OrderPlaced` message going to a broker, this trade is correct — losing the order is far worse than waiting 200ms. For live video, it's the wrong trade, which is why streaming uses UDP instead.

## Why does it exist?

[[TCP]] gives you a clean pipe of bytes — but underneath it, the network is a mess. Packets get **lost**, **reordered**, **duplicated**, and **corrupted**. If the application had to handle all four itself, every program would reinvent the same machinery: sequence numbers, acks, retransmits, reorder buffers, dedup tables. So the OS solves it once. That solution is reliable byte transport.

## What is it really?

A **contract**, not a mechanism. The contract says four things:

1. Bytes you send will arrive (no loss)
2. They will arrive in the order you sent them (no reordering)
3. They will arrive exactly once (no duplicates)
4. They will arrive intact (no corruption)

If any of these can't be met, the connection is broken and the application is told.

TCP is the most common implementation. QUIC and SCTP implement the same contract differently. The *contract* is what matters for reasoning about the system.

## The four failure modes it hides

| Failure | What happens | How reliability fixes it |
|---|---|---|
| **Lost** | Router drops the packet under load | Retransmit when no ack arrives |
| **Reordered** | Packets take different network paths | Sequence numbers + reorder buffer |
| **Duplicated** | Router resends one it thought was lost | Drop packets with seen sequence numbers |
| **Corrupted** | Bit flip from electrical noise | Checksum on every packet; treat bad ones as lost |

## Head-of-line blocking — the cost of "in order"

The "in order" guarantee has a hidden price. If packet 2 is lost, packets 3 and 4 may already have arrived — but the OS won't release them to the application until 2 is retransmitted.

```
[1] arrives  ✅  → released
[2] LOST    ❌
[3] arrives  ✅  → held in buffer
[4] arrives  ✅  → held in buffer

  ... ~200ms wait, retransmit ...

[2] arrives  ✅  → NOW release [2][3][4]
```

One lost packet stalls every byte behind it. This is **head-of-line blocking** — the reason real-time video uses UDP instead. For a video call, a 200ms freeze is worse than a missing audio frame. For a payment message, the opposite is true.

## Reliability vs speed — the trade

| | Reliable (TCP) | Fast (UDP) |
|---|---|---|
| **Lost packets?** | Retransmitted | Just gone |
| **Order?** | Guaranteed | Whatever arrives |
| **Latency floor** | Higher (acks, retransmits) | Lower |
| **Memory cost** | Sender buffers unacked bytes | None |
| **Right for** | Orders, payments, messages | Video, voice, telemetry |

Every messaging system — Service Bus, Kafka, RabbitMQ — picks reliable. Losing an `OrderPlaced` to save 200ms is a terrible trade.

## Common misconceptions

- **"TCP's dedup means duplicate messages can't happen."** No. TCP dedups at the **packet** level so the byte stream isn't garbled. Duplicate *messages* (the broker redelivering after a missed ack) are a different problem, solved at the application layer by [[Idempotency]].
- **"Reliable = no failures."** Reliable means failures are *hidden* from the application — or the connection breaks cleanly. It does not mean the network is healthy.
- **"Faster network means TCP doesn't need to do this work."** Even on a perfect LAN, packets can still reorder (different NIC queues) and routers can still drop under load. Reliability is always doing work; you just usually don't notice.

## A real example

The web server sends `OrderPlaced` to the broker:

```
Web server                         Broker
    │                                │
    │ send("OrderPlaced{...}")       │
    │ ─── packet 1 ────►              │  ✅
    │ ─── packet 2 ────►              │  ❌ LOST
    │ ─── packet 3 ────►              │  ✅ buffered, not delivered
    │                                │
    │ (no ack for 2 → retransmit)    │
    │ ─── packet 2 ────►              │  ✅
    │                                │
    │                                │  delivers [1][2][3] to broker app
    │                                │  broker writes to disk
    │ ◄──── ACK #1 ────              │
```

The web server's code is one line:

```python
sender.send_message(ServiceBusMessage('{"orderId": 123}'))
```

It never sees the lost packet, the retransmit, or the reorder. Reliable byte transport hides all of it. The only signal it gets is [[Broker|ACK #1]] from the broker once the full message is on disk.

## Mental model

**A pneumatic tube system in an old building.** You drop a capsule in at floor 1. Inside the walls, capsules sometimes get stuck, take wrong turns, or arrive in pairs because someone thought one was lost. The building's maintenance system handles all of it behind the scenes. You just see capsules arrive at floor 5 — in the order you sent them, exactly once. You never see the chaos in the pipes.

## See also

- [[TCP]]
- [[Idempotency]]
- [[Broker]]
- [[Delivery Semantics]]

## Index

[[Transport Fundamentals]]
