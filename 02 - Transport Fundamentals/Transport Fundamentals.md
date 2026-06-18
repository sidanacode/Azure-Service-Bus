---
tags: [moc, section]
---

# Transport Fundamentals

> Hub for section 2. The wire underneath everything.

## What this section covers

Before AMQP makes sense, we need to be solid on the floor it stands on — TCP. This section answers two things:

1. What does TCP actually deliver?
2. What does TCP **not** deliver, and why does that matter for AMQP?

## Notes (in order)

- [[TCP]] — reliable byte transport; what it gives, what it doesn't
- [[Reliable Byte Transport]] — what "reliable" really means (the four failure modes, head-of-line blocking, the speed tradeoff)
- [[TCP Connections]] — handshake, state, teardown

## Where this fits

Section **2 of 11**. Section 1 was messaging concepts. Section 3 is AMQP — which exists to fill the gaps TCP leaves open.

[[Index]]
