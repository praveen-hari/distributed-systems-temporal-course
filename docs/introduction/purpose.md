# Why This Course?

## The Problem

Modern applications are distributed by nature — they span multiple services, databases, and networks. Yet most developers learn to build systems as if everything runs on a single machine:

- **Network calls are treated as reliable** — but networks fail, timeout, and deliver messages out of order
- **Retries are added as an afterthought** — leading to duplicate charges, duplicate emails, and data inconsistencies
- **Error handling is local** — but failures cascade across services in unpredictable ways
- **State is assumed to be consistent** — but distributed state is eventually consistent at best

The result? Systems that work in development but break in production under real-world conditions.

---

## The Theory Gap

Martin Kleppmann's *Designing Data-Intensive Applications* (DDIA) is widely regarded as the best resource for understanding distributed systems. It explains:

- Why networks are unreliable (Chapter 8)
- Why distributed transactions are hard (Chapter 7)
- Why clocks can't be trusted (Chapter 8)
- Why replication introduces consistency challenges (Chapter 5)
- Why consensus is the foundation of coordination (Chapter 9)

But DDIA is a **theory book** — it explains the problems brilliantly without prescribing specific implementation solutions.

---

## The Implementation Gap

On the other side, tools like Temporal provide powerful primitives for building reliable distributed systems:

- **Durable execution** — workflows survive crashes and restarts
- **Automatic retries** — with configurable backoff and error classification
- **Saga pattern** — distributed transactions with compensation
- **Signals and queries** — external coordination with running workflows

But Temporal's documentation focuses on **how to use the API**, not **why these patterns exist** or what distributed systems problems they solve.

---

## Bridging the Gap

This course connects the two:

| DDIA Concept | Real-World Problem | Temporal Solution |
|---|---|---|
| Unreliable networks (Ch. 8) | Requests fail silently | Activity retries with backoff |
| Replication lag (Ch. 5) | Stale reads after writes | Durable execution with event history |
| Distributed transactions (Ch. 7) | Partial failures across services | Saga pattern with compensation |
| Consensus (Ch. 9) | Coordinating distributed state | Signals, queries, and updates |
| Batch processing (Ch. 10) | Processing large datasets reliably | Child workflows and continue-as-new |

Every Temporal feature you learn is motivated by a real distributed systems problem. You'll understand **why** before you learn **how**.

---

## What Makes This Course Different

1. **Theory-first, not tool-first** — Every Temporal feature is motivated by a real distributed systems problem from DDIA
2. **Failure-driven learning** — You intentionally break things to see Temporal rescue them
3. **Progressive complexity** — Each module builds on the last
4. **Real patterns** — Saga, fan-out, heartbeats, versioning — production-grade from day one
5. **TypeScript throughout** — All examples use the Temporal TypeScript SDK with modern patterns

---

## Next Step

[:octicons-arrow-right-24: Set up your environment](setup.md)
