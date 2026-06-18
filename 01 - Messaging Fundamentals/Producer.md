---
tags: [fundamentals, role]
---

# Producer

> A **producer** is a program that creates a message and sends it to the broker.

## Interview-ready answer

A producer is any program that originates a message and sends it to the broker. It opens a connection, builds the message, pushes the bytes, and waits for the broker to acknowledge. Once acknowledged, the producer's job is done — it doesn't care who reads the message, when, or whether anyone reads it at all. This decoupling is one of the main reasons messaging systems exist.

## Why does it exist?

Something has to start the chain.

A user clicks "Place Order".
A sensor reads a temperature.
A payment clears.

Some piece of code has to notice this and say:

> *"This matters. Other parts of the system need to know."*

That code is the producer. It is the start of every message flow.

## What is it really?

A producer is just a normal program. Your own code.

- A Python script
- A C# service
- A Node.js server

There is no special "producer software". Any program that sends a message **is** a producer.

## What does it do?

The producer follows six steps:

1. Open a connection to the broker
2. Log in
3. Build a [[Messages|message]] in memory
4. Push the bytes to the broker
5. Wait for the broker to say *"got it"*
6. Forget the message and move on

Step 3 is your code. The Service Bus SDK handles the rest.

## In code

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

client = ServiceBusClient.from_connection_string(CONN_STR)
sender = client.get_queue_sender("orders")

sender.send_message(ServiceBusMessage('{"orderId": 123}'))
```

This Python program is the producer.

## What the producer does NOT care about

- Who reads the message
- When it gets read
- Whether anyone reads it at all
- What happens after

Once the broker says *"got it"*, the producer is done.

## A real example

A user clicks "Place Order" on a website.

The web server saves the order to its database. Then it sends an `OrderPlaced` message to a Service Bus queue.

The web server is the producer.

The broker says *"got it"*. The web server replies "Order placed!" to the browser. Done.

Later, a payments service and a shipping service will read the message and do their work. The web server doesn't know or care.

## Mental model

**A person dropping off a parcel at the post office.**

You walk in, hand the package to the clerk, get a receipt, walk out. You don't wait for the package to be delivered. You don't know who picks it up at the other end. The post office takes responsibility from the moment they hand you the receipt — the producer's "got it" ack from the broker is exactly that receipt.

## Common misconceptions

- **"The producer talks to the consumer."** No. The producer only ever talks to the broker. It usually has no idea consumers exist, or how many.
- **"`send_message` returns when the consumer processes it."** No. It returns when the **broker** acknowledges (ACK #1 from the [[Broker]] note). The consumer may not have read the message for hours.
- **"Producers are special infrastructure."** No. A producer is just any program calling `send_message`. Your web server, a CLI tool, a cron job — anything that originates a message is a producer.

## See also

- [[The Raw Substrate]]
- [[Messages]]
- [[Broker]]
- [[Consumer]]

## Index

[[Messaging Fundamentals]]
