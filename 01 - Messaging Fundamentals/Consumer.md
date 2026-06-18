---
tags: [fundamentals, role]
---

# Consumer

> A **consumer** is a program that pulls messages from the broker and processes them.

## Interview-ready answer

A consumer is any program that reads messages from the broker and acts on them. It connects to the broker, asks for messages, does the work, and confirms completion so the broker can delete the message. Consumers pull rather than get pushed — they control their own pace, which means a slow or busy consumer never drops messages, since unread messages sit safely on the broker until it asks for them.

## Why does it exist?

The producer sent a message. The broker saved it.

Now what?

Someone has to **do something** with it.

A message that sits on the broker forever is useless. It needs to be picked up, read, and acted on.

That's the consumer's job.

## What is it really?

A consumer is just a normal program. Your own code.

- A Python script that processes orders
- A C# service that sends emails
- A Node.js worker that updates inventory

There is no special "consumer software". Any program that reads a message from the broker **is** a consumer.

## What does it do?

Six steps:

1. Open a connection to the broker
2. Log in
3. Ask the broker *"any messages for me?"*
4. Receive the bytes
5. Do the work
6. Tell the broker *"done — you can delete it"*

Step 5 is your code. The SDK handles the rest.

## In code

```python
from azure.servicebus import ServiceBusClient

client = ServiceBusClient.from_connection_string(CONN_STR)
receiver = client.get_queue_receiver("orders")

for msg in receiver:
    print(f"Got order: {msg}")
    # do the work here
    receiver.complete_message(msg)
```

The `complete_message()` call is the **second ack** from the [[Broker]] note — *"I'm done, delete it."*

## Consumers PULL, they don't get pushed

The broker doesn't randomly send messages out. The consumer asks, and the broker delivers.

This matters because:

- The consumer controls its own pace
- If the consumer is busy, it just doesn't ask for more
- If the consumer is down, messages sit safely on disk

The broker waits patiently. It never forces work onto a consumer that isn't ready.

## A real example

The broker has `OrderPlaced` saved on disk.

The payments service starts up. It connects to the broker as a consumer.
It asks: *"any orders?"*
The broker sends `OrderPlaced`.
Payments charges the credit card.
Payments tells the broker *"done"*.
The broker deletes the message.

## Mental model

**A worker picking jobs off a shelf.**

Jobs (messages) sit on a shelf at a workshop (the broker). The worker walks up, takes one off the shelf, does the work, and ticks it off the list. The worker never grabs a new job until they're ready. If they're sick, the job stays on the shelf — nothing is lost. If they finish a job but forget to tick it off, the workshop assumes it wasn't done and gives it to someone else. That's exactly how consumer pull and ack work.

## Common misconceptions

- **"Consumers receive messages automatically."** No. The consumer **asks** for them. The broker waits to be asked. Even SDK "push" loops are really polling underneath.
- **"`for msg in receiver:` finishes when there are no messages."** No. It blocks waiting for new messages. The consumer is a long-running process, not a one-shot script.
- **"Reading a message removes it from the broker."** No. The broker holds the message under a temporary lock until the consumer calls `complete_message()` (ACK #2). If the consumer crashes mid-process, the lock expires and the message goes back to the queue for someone else.
- **"More consumers always means faster processing."** Up to a point. After that, you're bottlenecked by the broker, the database the consumers write to, or external APIs. Adding consumers past that point just creates contention.

## See also

- [[The Raw Substrate]]
- [[Producer]]
- [[Messages]]
- [[Broker]]

## Index

[[Messaging Fundamentals]]
