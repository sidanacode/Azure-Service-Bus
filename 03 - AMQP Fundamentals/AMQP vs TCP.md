---
tags: [amqp, fundamentals]
---
	
# AMQP vs TCP

> **AMQP does not replace TCP. AMQP runs on top of TCP.** Each one does a different job. Take either one away and messaging breaks.

## Definition

AMQP and TCP are **two stacked layers**. TCP moves bytes between machines. AMQP turns those bytes into messages with delivery guarantees. AMQP is the upper layer; TCP is the lower one. AMQP **depends on** TCP.

## Problem it solves

Once you accept that AMQP exists, the obvious next question is: *"so does AMQP replace TCP, or what?"*

People get this wrong all the time. They picture AMQP as a competitor to TCP — a new way to move data. It is not. AMQP cannot move a single byte by itself. It hands every byte it produces to TCP and asks TCP to deliver it.

The relationship needs to be clear before any of the next concepts (frames, sessions, links, settlement) make sense. They are all built on the assumption that TCP is doing its job *underneath*.

## Why previous solution was insufficient

Why not just extend TCP to handle messaging?

TCP is **general-purpose**. The same TCP that carries your AMQP frames also carries your web traffic, SSH sessions, database queries, video streams. If TCP knew about messages, it would also have to know about web pages, SQL, and video — which is absurd. TCP is kept small and generic on purpose.

The fix is **layering**. TCP does one thing well: reliable byte delivery. AMQP sits above it and adds the messaging-specific rules. SSH sits above it and adds remote-shell rules. HTTP sits above it and adds request/response rules. Each upper-layer protocol picks the rules it needs without bloating the lower layer.

## Quick word: "frame"

A **frame** is the unit AMQP actually sends on the wire. Each frame starts with a small size field, followed by some header bytes, followed by the message body:

```
┌────────┬────────────────┬──────────────────────┐
│  size  │     header     │        body          │
└────────┴────────────────┴──────────────────────┘
```

The size up front tells the receiver *"this frame is N bytes long"*, so it knows where one frame ends and the next begins. That's how AMQP draws message boundaries on TCP's byte stream.

For now you only need this much: **frame = one chunk AMQP hands to TCP, with size + header + body.** The full structure is covered later in [[Message Boundaries]] and the section on AMQP transport.

## Responsibilities

Two layers, two jobs. The line between them is sharp.

| Layer | Job | What it knows | What it does NOT know |
|---|---|---|---|
| **AMQP** (upper) | Turn bytes into safely-delivered messages | Frames, sessions, settlement, credits | IP addresses, packet loss, retransmission |
| **TCP** (lower) | Move bytes between machines reliably | Connections, ordering, retransmission | What a "message" is, who the broker is |

When you call `send_message(order)`:

1. AMQP builds a frame: `[size][header][body]`.
2. AMQP hands those bytes to TCP and says *"deliver these."*
3. TCP breaks them into packets, handles retries, reassembly, ordering.
4. On the other side, TCP hands a clean byte stream to the broker's AMQP layer.
5. The broker's AMQP layer reads `size`, knows where the frame ends, decodes it as a message.

Neither layer reaches into the other's job. AMQP never touches packets. TCP never touches messages.

## Real-world example

An order is placed. Trace one message through both layers:

```
Producer side                        Broker side
──────────────                       ────────────
Web server:                          
 sender.send_message(order)
        │                            
        ▼                            
 AMQP layer:                         
   build frame                       
   [size=42][header][body]           
        │                            
        ▼                            
 TCP layer:                          ── network ──►   TCP layer:
   split into packets                                   reassemble bytes
   handle acks/retries                                  hand clean stream up
        │                                                       │
        ▼                                                       ▼
   bytes on the wire                                     AMQP layer:
                                                          read size
                                                          decode frame
                                                          → "OrderPlaced"
                                                          ack to producer's AMQP
                                                                │
                                                                ▼
                                                          broker saves to disk
```

The web server never thinks about packets. The TCP layer never thinks about orders. Each layer trusts the one below it.

## Mental model

**A mail truck on a highway.**

- **TCP is the highway.** It moves *anything* from A to B reliably. It does not care what is in the truck.
- **AMQP is the mail truck.** It carries sorted mail, addressed envelopes, return-receipt slips, delivery confirmations.

The highway without the mail truck is just an empty road — bytes go places but nobody knows what they mean.

The mail truck without the highway is a truck stuck in a parking lot — well-organized cargo with no way to move.

You need both, doing different jobs.

## Interview answer

AMQP and TCP are layered protocols. TCP is the lower layer — it provides reliable, ordered byte delivery between two machines. AMQP is the upper layer — it adds message framing, routing headers, application-level acknowledgements, multiplexing, and flow control on top of TCP's byte stream. AMQP runs on top of TCP and depends on it; TCP has no awareness of messaging. Keeping the layers separate means TCP stays small and general-purpose while AMQP focuses entirely on messaging concerns. The same separation is why HTTP, SSH, and TLS can all share TCP underneath — each is a different upper-layer protocol on the same transport.

## Common misconceptions

- **"AMQP replaces TCP."** No. Take TCP away and AMQP has no transport. AMQP frames travel inside TCP segments.
- **"TCP knows about messages."** No. TCP only sees a stream of bytes. The whole reason AMQP exists is that TCP has no concept of "where one message ends."
- **"AMQP knows about IP addresses or retransmission."** No. AMQP hands its bytes to TCP and trusts TCP to deal with the network. If a packet is lost and retransmitted, AMQP never finds out.
- **"AMQP could just be built into TCP."** It could be, in theory — and it would ruin TCP. TCP is general-purpose precisely because it stays out of application concerns. Layering keeps each protocol simple.
- **"You can swap TCP for UDP and AMQP still works."** Not really. AMQP assumes ordered, lossless byte delivery. UDP gives neither. AMQP-over-UDP would have to rebuild most of TCP from scratch — which is what QUIC does for HTTP/3, and it is a large project.

## Design tradeoff

**Why layered:** every layer does one job well. TCP can be reused by HTTP, SSH, TLS, AMQP. AMQP can focus only on messaging. Either layer can evolve independently — TCP has had decades of optimization without AMQP changing, and AMQP versions can change without breaking TCP.

**The advantage:** clear separation of concerns. Bugs and tuning at one layer do not leak into the other. A network engineer can debug TCP problems without knowing AMQP. An application engineer can reason about messages without knowing IP.

**The cost:**

1. **Two layers of overhead.** Every message pays both TCP framing (per packet) and AMQP framing (per frame).
2. **Two failure surfaces.** A failure can come from the network (TCP) or the broker application (AMQP). Diagnosing requires understanding both.
3. **Information hiding cuts both ways.** AMQP cannot see TCP-level signals like packet loss, so it cannot react to them directly — it only sees that bytes eventually arrive (or the connection drops).

For 99% of messaging work, the cost is invisible and the layering is exactly what you want.

## See also

- [[Why AMQP Exists]] — the gap layer above TCP fills
- [[TCP]] — what AMQP runs on top of
- [[Reliable Byte Transport]] — TCP's actual promise
- [[Message Boundaries]] — how AMQP draws frame edges on TCP's byte stream

## Index

[[AMQP Fundamentals]]
