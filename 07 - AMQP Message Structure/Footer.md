---
tags:
  - amqp
  - message-structure
---

# Footer

> **Footer is the message's tail section** — an open dictionary at the very end, after the body, used for fields computed *over* the body itself: checksums, signatures, digests. Service Bus runs over TLS on TCP, so transport-level integrity already covers everything Footer would; the SDK doesn't expose Footer and you'll essentially never see it in Service Bus traces. It exists for cross-network federation where end-to-end message integrity above the transport layer is needed.

## Definition

**Footer** is the 7th and final section of an AMQP message — an **open dictionary** of string keys to scalar values, positioned after the [[Body]]. Like [[Message Annotations]] and [[Application Properties]], its keys are not standardised by AMQP. Unlike them, it sits at the *tail* of the message rather than the head.

The position is the design decision — Footer fields are typically values that can only be computed once the body has been fully serialised (checksums, signatures, message digests).

## Problem it solves

Some metadata can't be written until the entire body has been processed. The most obvious examples:

- **Checksum over the body** — you can't write the checksum byte until you've hashed every body byte.
- **Cryptographic signature** — same constraint; the signature is over the entire serialised body.
- **Trailing message digest** — for streaming verification.

If these had to live in [[Header]] or [[Message Annotations]] (which sit *before* the body on the wire), the producer couldn't stream a large message — it would have to fully buffer the body first to compute the trailing field, then go back and write it before the body. That defeats streaming.

The fix: a section at the **end** of the message. Producer streams body, computes the checksum/signature as it goes, writes the result into Footer once the body finishes. Receiver does the inverse.

This is the same pattern as TCP segment checksums (appended after payload) and HTTP/1.1 chunked encoding's `Trailer:` headers.

## Why previous solution was insufficient

A naive design would put everything before the body — Header, Properties, Annotations all up front, then body, done. But:

1. **Streaming breaks** — for a 100MB message, the producer would have to buffer the entire body, hash it, write the hash up front, then stream the body. Massive memory cost and added latency.
2. **No way to do end-to-end integrity** — without a trailing field, there's nowhere natural to put a value computed over the body.

Footer solves both by positioning a small open dict at the tail.

## Responsibilities

Footer owns:

- **Body-derived integrity values** — checksums, digests, signatures.
- **End-to-end fields above transport** — values that need to survive across multiple AMQP brokers, beyond the per-hop TLS that protects each connection.

What Footer does NOT own:

- **Routing or filtering metadata** — that's [[Application Properties]] / [[Message Annotations]].
- **Body content itself** — that's the [[Body]].
- **Anything Service Bus uses in practice** — Service Bus doesn't populate Footer.

## Why Service Bus doesn't use it

Service Bus mandates **TLS** (over TCP). TLS provides:

- **Integrity** via MAC over each TLS record — bit-flips are detected.
- **Authenticity** via the TLS handshake — peers are who they claim to be.
- **Replay protection** via TLS sequence numbers.

A Footer checksum or signature would be **redundant work for the same guarantee** Service Bus already provides at the transport layer. The SDK doesn't expose Footer construction, and packet captures of normal Service Bus traffic never contain it.

The case where Footer matters is **end-to-end above transport**: a message signed by the producer, forwarded through multiple brokers (each with its own TLS connection), arriving at the consumer. The consumer verifies the producer's signature is still valid — proving no broker on the path tampered with the message, even though each TLS leg only protected its own hop. This is unusual in Service Bus deployments but standard in cross-organisation AMQP federation.

## Mental model

> **Footer is a wax seal on the back of an envelope.**
>
> [[Header]] is the shipping label. [[Properties]] is the printed delivery address. [[Application Properties]] is sticky notes. [[Message Annotations]] is customs stamps. [[Body]] is the letter inside, sealed.
>
> **Footer is a wax seal pressed onto the back flap** — a tamper-evident mark applied after the letter is sealed inside. Lets the recipient verify *"this envelope hasn't been opened or replaced in transit"*, even if it passed through five postal services.
>
> Useful for international diplomatic mail. Pointless for ordinary post when the courier is trusted (TLS). That's why Service Bus, which trusts its TLS-protected couriers, doesn't bother sealing the envelope.

## Interview answer

Footer is the final section of an AMQP message — an open dictionary at the tail, positioned after the [[Body]] specifically so it can carry fields computed *over* the body (checksums, signatures, digests). The trailing position lets producers stream a large body and write the computed integrity value once streaming is done, the same pattern as TCP segment checksums and HTTP/1.1 trailers. In Service Bus, Footer is essentially unused — TLS over TCP already provides per-hop integrity and authenticity, so a Footer-level checksum would be redundant work for the same guarantee. Footer matters in cross-network federation scenarios where end-to-end integrity above the transport layer is needed (a message signed by the producer, forwarded through multiple brokers, verified by the consumer regardless of how many TLS hops it crossed).

## Common misconceptions

- **"Footer is a standard part of every AMQP message."** No. Footer is optional, like every section except Body. Most messages don't have one.
- **"I should put my custom metadata in Footer."** No. Custom application metadata goes in [[Application Properties]]. Footer is specifically for body-derived integrity values.
- **"Service Bus exposes Footer in the SDK."** No. Service Bus doesn't surface Footer construction or reading. If you need end-to-end signing in Service Bus, you'd typically embed it in the body or Application Properties.
- **"Footer is the same as Message Annotations, just at the end."** Position-wise yes (both open dicts), purpose-wise no. Annotations are broker metadata read on receipt; Footer is integrity data computed over the body. Different lifecycles.
- **"Footer protects against tampering automatically."** Only if a producer actually populates it AND the consumer verifies it. Putting random bytes in Footer doesn't do anything by itself.

## See also

- [[Body]] — Footer values are computed over Body bytes
- [[Message Annotations]] — the broker's open dict at the *head* of the message; Footer is the open dict at the *tail*
- [[AMQP Message Structure]] — the 6-section overview Footer completes
- [[AMQP Protocol Header]] — the protocol-level greeting; conceptually paired with Footer (one starts the conversation, one closes the message)

## Index

[[AMQP Message Structure]]
