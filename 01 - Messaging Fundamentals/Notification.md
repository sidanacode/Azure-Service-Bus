---
tags: [fundamentals, message-type]
---

# Notification

> A **notification** is a message that informs a known audience, usually external to the system.

## Interview-ready answer

A notification is a message that informs a specific known audience — typically a person or an external system. Unlike an [[Event]], which announces a fact to anyone listening inside the system, a notification has a clear intended recipient and usually crosses the boundary out of the system. Examples include order confirmation emails, password-change alerts, and admin warnings. The sender expects no response and the system still works if the notification is lost, though the recipient simply isn't informed.

## Why does it exist?

You already know two message intents:

- **[[Command]]** — *"do this"* — sent to one specific receiver
- **[[Event]]** — *"this happened"* — anyone interested can listen

But there's a middle case.

Sometimes the sender wants to inform **specific** parties, but doesn't care what they do about it. It's not "do this thing", and it's not a fact for the world — it's a heads-up to someone.

> *"Hey user, your order is ready."*
> *"Hey admin, disk usage hit 90%."*
> *"Hey customer, your password was changed."*

That kind of message is a notification.

## What is a notification?

A notification says: **"FYI — here's something you should know."**

The sender wants the recipient to be aware of something. They are not asking the recipient to do anything specific.

Examples:

- `OrderReady` — tell the customer their order is ready
- `LowDiskSpaceAlert` — tell the admin disk is filling up
- `PasswordChanged` — tell the user their password was just changed
- `BackupCompleted` — tell the ops team the backup finished

## Notification vs Event

This is the part most people get confused about. Both are informational. Both don't expect a response.

The difference is **audience and direction**.

| | Event | Notification |
|---|-------|--------------|
| **Audience** | the system — any service that wants to listen | a known specific recipient |
| **Sender awareness** | doesn't know who's listening | knows exactly who it's for |
| **Where it goes** | stays inside the system | usually leaves the system (to a human or external service) |
| **How it's sent** | published to a topic | sent to a specific recipient |

> An **event** lets the system know something happened.
>
> A **notification** informs a known audience, usually external.

## The classic example

A user places an order:

1. The web server publishes `OrderPlaced` → **event**
2. Payments, shipping, analytics, and email all react inside the system
3. The email service then sends `OrderConfirmationEmail` → **notification**
4. The email reaches the user's inbox — outside the system

Same data, different intent, different audience.

## Key properties

### One sender, one specific audience

A notification is for someone in particular. Not a broadcast to anyone curious.

### The sender doesn't expect a response

Same as events. The sender informs and moves on.

### Often involves an external channel

Email. SMS. Push notification. Webhook. Slack.

The thing that makes a notification "notification-y" is that it usually leaves the message bus to reach a recipient outside the system.

### Lost notifications don't break the system

If a notification fails, the system still works. The user just isn't informed.

This is different from a command, where failure means the work didn't get done.

## In code

```python
ServiceBusMessage(
    body='{"userId": 42, "email": "u@x.com", "orderId": 123}',
    subject="SendOrderConfirmation"
)
```

This message goes to the email service. The email service reads it and emails the user.

The Service Bus message is **internal**. The actual notification (the email) leaves the system.

## Where the line blurs

In real systems, notification and event sometimes overlap. Teams aren't always strict.

Some treat notifications as a **special type of event** — one specifically meant for end users. Others treat them as a separate intent.

Don't get hung up on the line. The core idea is:

> A notification is a message that informs a known audience — usually a human or an external system.

## The three intents so far

| | Command | Event | Notification |
|---|---------|-------|--------------|
| **Says** | "do this" | "this happened" | "FYI" |
| **Audience** | one specific service | anyone interested | known recipient |
| **Where it goes** | internal | internal | often external |
| **Sender knows recipient?** | yes | no | yes |
| **Expects response?** | yes (action) | no | no |

## Mental model

**A doorbell at a specific apartment.**

The building (the system) doesn't ring every doorbell — only the one for apartment 4B (the specific recipient). The visitor (sender) knows exactly who they're notifying. The resident may answer, may ignore it, may not even be home. The visitor doesn't expect a response — they just wanted the resident to know someone's at the door. Notifications are doorbells aimed at one known person.

## Common misconceptions

- **"A notification is just an email."** No. The notification is the **message** asking the email service to send the email. The email itself is the channel through which the recipient is reached. Confusing the two breaks the layering.
- **"Notifications and events are the same."** They overlap, but the line is **audience**. An event is broadcast to anyone in the system; a notification targets one known recipient (often outside the system). The same fact can produce both.
- **"Notifications must be delivered or the system fails."** No. A lost notification means the recipient isn't informed — the underlying business work still completed. This is different from a command, where missed delivery = work not done.

## See also

- [[Messages]]
- [[Command]]
- [[Event]]
- [[Query]]

## Index

[[Messaging Fundamentals]]
