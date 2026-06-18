---
tags: [fundamentals, message-type]
---

# Query

> A **query** is a message that asks for information and waits for a reply.

## Interview-ready answer

A query is a message that requests information from another service. Unlike commands, events, or notifications, a query **expects a response** — the sender waits until it gets the answer. A query must never change state; it only reads. This separation between read and write operations is the core idea behind the CQRS pattern (Command Query Responsibility Segregation). In practice, queries usually go through direct API calls rather than a message bus, since the sender needs a fast reply.

## Why does it exist?

The other three intents are all **one-way**:

- **[[Command]]** — *"do this"* — sender doesn't wait for the result
- **[[Event]]** — *"this happened"* — sender expects nothing back
- **[[Notification]]** — *"FYI"* — sender expects nothing back

But what if the sender actually needs an answer?

> *"Hey inventory service, how many of product X are in stock?"*
> *"Hey user service, what's user 42's email address?"*
> *"Hey pricing service, what's the discount for this customer?"*

The sender has a question. They need the answer to do their work.

That's a query.

## What is a query?

A query says: **"tell me something."**

It is a request for information.

The sender asks. The receiver responds. The sender waits.

Examples:

- `GetUserById`
- `CheckInventory`
- `GetOrderStatus`
- `CalculatePrice`

Notice the names. They start with verbs like `Get`, `Check`, `Calculate`, `Find` — words about **looking up**, not changing.

## Key properties

### Two messages, not one

A query is actually a **pair**:

```
[Sender] ──── GetUserById ────► [Receiver]
[Sender] ◄──── UserFound  ────  [Receiver]
```

The first message is the question. The second is the answer.

### The sender waits

Unlike all other intents, the sender cannot move on until it has the answer.

### A query NEVER changes state

This is the rule. A query only reads.

- ✅ Reading the user's email — query
- ❌ Updating the user's email — not a query (that's a command)

If a message both reads and changes things, it's a command, not a query.

### Queries are usually fast

Because the sender is waiting, queries need to come back quickly.

## Why queries usually skip the message bus

> *"If queries need a fast reply, why use messaging at all? Why not just call an API directly?"*

This is a fair question.

Most of the time, queries **don't** go through a message bus.

For looking up data, an HTTP/REST or gRPC call is usually better:

- Faster (no broker hop)
- Direct (caller knows where the data is)
- Built-in request/response shape

So in practice:

- **Commands and events** → message bus (Service Bus, Kafka)
- **Queries** → direct API call

Queries through a message bus exist (especially in CQRS architectures), but it's not the default. Service Bus supports it via a feature called **request-reply**, covered later if needed.

## CQRS

You'll hear this term: **CQRS** — Command Query Responsibility Segregation.

It's a design pattern that separates:

- **Commands** — change state
- **Queries** — read state

The point is: don't mix them. A method should either change something or read something — never both.

This is why "query" is recognized as its own message intent. It deserves its own concept.

## The four intents — final picture

| | Command | Event | Notification | Query |
|---|---------|-------|--------------|-------|
| **Says** | "do this" | "this happened" | "FYI" | "tell me something" |
| **Audience** | one service | anyone | known recipient | one service |
| **Direction** | internal | internal | often external | internal |
| **Expects response?** | maybe ack | no | no | **yes — the answer** |
| **Changes state?** | yes | no | no | **no** |

## The shape to remember

> A query says **"tell me something."**
>
> It is a **request for information**.
>
> The sender **waits** for the answer.
>
> A query **never changes** state.

## Mental model

**A librarian lookup.**

You walk to the librarian and ask *"do you have this book?"* You stand there and wait for the answer. The librarian doesn't change anything — they only check the records and reply. You can't move on until you hear yes or no. Queries are librarian lookups: read-only, request-and-wait, no shelves rearranged.

## Common misconceptions

- **"Queries belong on a message bus."** Usually not. Because the sender waits for a reply, queries are almost always done over direct API calls (HTTP, gRPC). Service Bus supports request-reply, but it's the exception, not the default.
- **"A query can have side effects if they're small."** No. The moment a query changes state — even logging, even caching — it's no longer a query in the CQRS sense. Mixing read and write breaks the pattern.
- **"Commands and queries are interchangeable if you ignore the response."** No. The split is about **intent and side effects**, not whether the caller waits. A command changes state; a query never does.

## See also

- [[Messages]]
- [[Command]]
- [[Event]]
- [[Notification]]

## Index

[[Messaging Fundamentals]]
