# Azure Service Bus — From the Wire Up

A bottom-up curriculum: **TCP bytes → AMQP messages → Azure Service Bus.**

These notes are written for engineers who want to *understand* Service Bus, not just call its SDK. Every concept is introduced problem-first: what breaks without it, why earlier solutions weren't enough, and what the layer above gets to assume because of it.

> Built in [Obsidian](https://obsidian.md) — internal links use `[[wikilink]]` syntax. They won't be clickable on GitHub, but each linked note exists in the same section folder.

## How this is organized

11 sections, each in its own folder with a map-of-content (MoC) note. The order matters — earlier sections build the foundation later ones rely on.

Legend: ✅ done · 🚧 in progress · ⏳ not started

| # | Section | Status |
|---|---|---|
| 1 | [Messaging Fundamentals](./01%20-%20Messaging%20Fundamentals/) | ✅ |
| 2 | [Transport Fundamentals](./02%20-%20Transport%20Fundamentals/) | ✅ |
| 3 | [AMQP Fundamentals](./03%20-%20AMQP%20Fundamentals/) | ✅ |
| 4 | [AMQP Transport Layer](./04%20-%20AMQP%20Transport%20Layer/) | ✅ |
| 5 | [AMQP Message Transfer](./05%20-%20AMQP%20Message%20Transfer/) | ✅ |
| 6 | [AMQP Message Lifecycle](./06%20-%20AMQP%20Message%20Lifecycle/) | ✅ |
| 7 | [AMQP Message Structure](./07%20-%20AMQP%20Message%20Structure/) | ✅ |
| 8 | Service Bus Core Concepts | 🚧 |
| 9 | Service Bus Message Processing | ⏳ |
| 10 | Service Bus Advanced | ⏳ |
| 11 | Service Bus Architecture & Ops | ⏳ |

The [full index](./Index.md) has every note listed in reading order, plus a top-level Mermaid flow showing the whole curriculum at a glance.

## Visual aids

Every section MoC has two Mermaid diagrams:

- **Bridge from previous section** — what carried over and which gap forced the next layer
- **Section flow** — the section's core idea in one picture

These render natively on GitHub and in Obsidian. The curriculum-wide flow lives in [Index.md](./Index.md).

Section 7 also includes a [Wire Walkthrough](./07%20-%20AMQP%20Message%20Structure/Wire%20Walkthrough.md) — a synthesis note that traces a single message end-to-end through SDK → frame → TCP → broker → fsync → DISPOSITION, with length-prefix framing explained at three nested levels.

## Note structure

Every concept note follows the same eight-section shape:

1. **Definition** — what it is, in one paragraph
2. **Problem it solves** — why it had to exist
3. **Why previous solution was insufficient** — what broke without it
4. **Responsibilities** — what it owns vs. what it doesn't
5. **Real-world example** — concrete walkthrough
6. **Mental model** — the analogy that sticks
7. **Interview answer** — the tight version
8. **Common misconceptions** — what people get wrong

## Reading on the web

GitHub renders Markdown well enough for everything to be readable. For the full experience (graph view, backlinks, working `[[wikilinks]]`), clone the repo and open the folder as an Obsidian vault.

```bash
git clone https://github.com/sidanacode/Azure-Service-Bus.git
```

## Author

[Tanish Sidana](https://github.com/sidanacode) — written while teaching myself Service Bus from first principles. If you spot something wrong or unclear, open an issue.
