---
tags: [amqp, transport]
---

# AMQP Protocol Header

> **The very first 8 bytes both sides send on a fresh TCP socket — a fixed pattern that lets two programs confirm "we both speak AMQP 1.0" before either sends anything that depends on AMQP rules.** It is not a frame. Frames cannot exist yet.

## Definition

The **AMQP protocol header** is a fixed 8-byte sequence — `AMQP\x00\x01\x00\x00` — that both peers send the moment a TCP socket is open. Each side reads the other's 8 bytes; if they match, both sides switch their state machine into "AMQP 1.0 mode" and start exchanging [[Frames|frames]]. If they don't match, the socket is closed before anything else happens.

Byte-by-byte:

```
A    M    Q    P    \x00 \x01 \x00 \x00
0x41 0x4D 0x51 0x50 0x00 0x01 0x00 0x00
└── protocol name ──┘ ┌── protocol-id
                      │   0 = plain AMQP
                      │   2 = AMQP over TLS
                      │   3 = AMQP-SASL (auth first)
                      └── major.minor.revision = 1.0.0
```

## Problem it solves

[[TCP]] has just opened a byte pipe. Both sides can send bytes — but neither side knows what the other intends to do with them.

- The producer doesn't know if it dialled the right port. The thing answering on `:5671` could be an HTTP server, an MQTT broker, a misconfigured proxy, a load balancer in the wrong mode.
- The broker doesn't know if the incoming bytes are an AMQP client, a curl request, a port scanner, or random garbage.

Sending an AMQP frame as the first thing is a non-starter — frames have a length-prefix wire format that *only makes sense if both sides have agreed to speak AMQP*. You can't ask "do you speak AMQP?" inside an AMQP frame; that's the question you're trying to answer.

## Why previous solution was insufficient

Two earlier ideas don't work:

- **"Just send the OPEN frame first."** OPEN is a frame, and frames are AMQP-specific. If the peer isn't actually an AMQP broker (wrong port, wrong protocol), it has no idea what to do with your bytes — and you have no idea if its reply is an AMQP frame or an HTTP error page. You need agreement *before* the framed conversation starts.
- **"Use a different TCP port per protocol."** Ports help (5672 for plain AMQP, 5671 for AMQP-over-TLS) but don't solve the problem on their own. A misconfigured server, a tunneled connection, or an upgrade across protocol versions can still leave you talking to something other than what you expected. The header is a final, in-band check.

Pipe-level "are we compatible?" must be answered *before* any pipe-level work starts.

## Responsibilities

The protocol header has exactly one job: **fingerprint the protocol before either side commits to the framed conversation.**

Specifically:

- **Confirm the language.** `AMQP` ASCII bytes prove the peer is at least claiming AMQP. A non-AMQP server will respond with garbage or close — both detectable in milliseconds.
- **Pin the version.** `\x01\x00\x00` says "AMQP 1.0.0." If the peer only speaks 0-9-1 (RabbitMQ classic) or some future 2.x, both sides know immediately and can hang up cleanly.
- **Pick the auth mode.** `\x00` = plain AMQP, no SASL. `\x03` = AMQP-SASL — *do auth first, then re-greet with `\x00`*. `\x02` = AMQP over a TLS tunnel that's already been established underneath.
- **Engage framing.** After both 8-byte headers have been exchanged successfully, both sides flip a state-machine switch: *"from now on, every byte on this socket is part of a frame."*

What the header does *not* do:

- It does not authenticate. SASL handles that (and carries its own performatives once SASL is in play).
- It does not negotiate `max-frame-size`, `channel-max`, or heartbeats. Those live in the [[Connection|OPEN frame]], which is the *first* real frame after framing engages.
- It is never sent again. There is exactly one header exchange per TCP connection, at the very start.

## Real-world example

A Service Bus producer connects to `mybroker.servicebus.windows.net:5671`. Three handshakes happen, stacked:

```
Time   │  TCP layer (OS)              │  TLS layer                  │  AMQP layer
───────┼──────────────────────────────┼─────────────────────────────┼──────────────────────────────
 t0    │  open socket                 │                             │
 t1-3  │  SYN / SYN-ACK / ACK         │                             │
 t4    │  ✅ TCP pipe open             │                             │
 t5-8  │                              │  TLS 1.2/1.3 handshake      │
 t9    │                              │  ✅ encrypted tunnel         │
 t10   │                              │                             │  Producer: "AMQP\x03\x01\x00\x00" ──►
 t11   │                              │                             │  Broker:   ◄── "AMQP\x03\x01\x00\x00"
 t12   │                              │                             │  ✅ both agree: AMQP 1.0, do SASL first
 t13   │                              │                             │  SASL exchange (SAS token / cert)
 t14   │                              │                             │  Producer: "AMQP\x00\x01\x00\x00" ──►
 t15   │                              │                             │  Broker:   ◄── "AMQP\x00\x01\x00\x00"
 t16   │                              │                             │  ✅ re-greeted, framing now engaged
 t17   │                              │                             │  Producer: OPEN frame ──►
 t18   │                              │                             │  Broker:   ◄── OPEN frame
 t19   │                              │                             │  ✅ Connection open
```

Note the **two greeting exchanges** when SASL is in play: one with protocol-id `\x03` ("auth first"), then SASL itself, then a second exchange with `\x00` to switch into the post-auth framed conversation. On a port without SASL, only the second exchange happens.

After t16, both sides know: "the next bytes I read are a frame header, not random data." That single state flip is what makes the rest of the AMQP handshake (OPEN, BEGIN, ATTACH, TRANSFER…) even parseable.

## Mental model

> **The header is "ENGLISH?" — "ENGLISH!" before the actual conversation starts.**

You walk up to a stranger. You can't ask "what language do you speak?" in a language they might not speak — that's circular. So you say one short word: *"English?"*. They either reply *"English!"* and the conversation begins, or they say nothing and you walk away. The greeting itself isn't part of the conversation; it's the test for whether a conversation is even possible.

The protocol header is the same shape, made of bytes:

| Real life | AMQP wire |
|---|---|
| Yell "ENGLISH?" | Send `AMQP\x00\x01\x00\x00` |
| Listen for "ENGLISH!" | Read 8 bytes, compare |
| Match → start the conversation in English | Switch into framed AMQP mode |
| No match → walk away | Close the socket |

Or, in file-format terms: the header is the **magic bytes** for a TCP socket. PNG starts with `89 50 4E 47`, ZIP with `PK\x03\x04`, and AMQP with `AMQP\x00\x01\x00\x00` — same idea, fingerprint-first so a parser knows what it's looking at before it starts looking.

## Where this sits in the handshake

The header is **step 0** of the bigger handshake covered in Section 5 ([[Handshake Choreography]]). The full sequence on a fresh connection:

```
1. TCP 3-way handshake          (OS)
2. TLS handshake                (only on port 5671)
3. AMQP protocol header (8B)    ◄── this note
4. SASL exchange                (only if protocol-id was \x03)
5. AMQP protocol header (8B)    (re-greet after SASL with protocol-id \x00)
6. OPEN frame exchange          (Connection negotiation)
7. BEGIN frame exchange         (Session opens)
8. ATTACH frame exchange        (Link opens)
9. FLOW frame                   (initial credit grant)
... TRANSFER frames flow ...
```

Steps 3 and 5 are the protocol header. Everything from step 6 onward is **frames** — which is why framing is something that *engages* after the header succeeds, not a separate phase you can put on a flow chart between OPEN and BEGIN.

## Interview answer

The AMQP protocol header is a fixed 8-byte sequence — `AMQP\x00\x01\x00\x00` — that both peers send on a fresh TCP socket before any AMQP frame is exchanged. It exists because TCP only delivers bytes; it doesn't tell you what protocol the bytes are in. The header is a fingerprint: each side reads the other's 8 bytes, confirms it says AMQP version 1.0, and only then engages framing and proceeds to OPEN. If the bytes don't match, the socket is closed cleanly — better to bail than send frames into something that isn't an AMQP peer. It's the same idea as magic bytes at the start of a file format, applied to a TCP socket. Auth (SASL) and Connection-level negotiation (max-frame-size, heartbeat) come *after* the header succeeds, not inside it.

## Common misconceptions

- **"The protocol header is the first frame."** No. Frames are an AMQP-layer concept that only makes sense once both sides have agreed to speak AMQP. The header is the *agreement* — it can't itself depend on the agreement existing. It is plain bytes, sent and parsed by hand on both sides.
- **"Framing happens before OPEN as its own step."** Framing isn't a step you take; it's a mode you switch on. The protocol header turns it on. From the moment the header is accepted, every subsequent byte is part of some frame — OPEN is just the first frame on top of that mode.
- **"You can skip the header and send OPEN directly to save a round trip."** No. There is no "AMQP 1.0 fast path" — both peers refuse to interpret bytes as frames until they've seen the header. Sending OPEN first will get your TCP connection closed.
- **"The header authenticates the connection."** No. The header confirms protocol compatibility only. SASL handles auth, and SASL only runs when the header is sent with protocol-id `\x03`.
- **"`AMQP\x00\x01\x00\x00` is a TCP-level thing."** No. TCP doesn't know about it. It's just 8 application-layer bytes that both endpoints know to send and parse, before passing anything else to the AMQP library.

## See also

- [[TCP]] — the byte transport underneath
- [[TCP Connections]] — the OS handshake the header rides on
- [[Connection]] — the AMQP layer the header opens the door to
- [[Frames]] — the wire format that engages once the header succeeds
- [[Handshake Choreography]] — the full multi-frame opening sequence

## Index

[[AMQP Transport Layer]]
