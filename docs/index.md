---
hide:
  - navigation
---

# Reliable Distributed Systems with Temporal

<div style="text-align: center; margin: 2rem 0;">
<em style="font-size: 1.3rem;">"Understand WHY distributed systems fail → Learn HOW Temporal solves it → BUILD it yourself"</em>
</div>

A comprehensive course bridging **distributed systems theory** from Martin Kleppmann's *Designing Data-Intensive Applications* with **hands-on Temporal + TypeScript implementation**.

---

## :material-school: Course Philosophy

Every module follows a **3-part structure**:

!!! tip "The Learning Loop"
    1. :material-book-open-variant: **Theory** — DDIA concepts explained clearly
    2. :material-head-lightbulb: **Mental Model** — What it means in practice
    3. :material-hammer-wrench: **Build It** — Temporal + TypeScript code you can run

---

## :material-book-open-variant: About DDIA — Designing Data-Intensive Applications

<div class="grid" markdown>

<div markdown>

[*Designing Data-Intensive Applications*](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-(2017).pdf) by **Martin Kleppmann** (O'Reilly, 2017) is widely regarded as the definitive guide to distributed systems. It explains the fundamental challenges that every distributed system must face — and why there are no easy answers.

</div>

<div markdown>

!!! quote "The core insight"
    "Data-intensive applications are pushing the boundaries of what is possible by making use of these technological developments. An application is *data-intensive* if data is its primary challenge — the quantity of data, the complexity of data, or the speed at which it is changing."

</div>

</div>

### DDIA Chapter Overview

| Part | Chapters | Key Concepts |
|------|----------|-------------|
| **I. Foundations** | Ch. 1 — Reliability, Scalability, Maintainability | The three pillars of good systems; faults vs failures; percentiles |
| | Ch. 2 — Data Models & Query Languages | Relational vs document vs graph; schema-on-write vs schema-on-read |
| | Ch. 3 — Storage & Retrieval | B-Trees vs LSM-Trees; column-oriented storage; data warehousing |
| | Ch. 4 — Encoding & Evolution | JSON/Protobuf/Avro; forward & backward compatibility; schema evolution |
| **II. Distributed Data** | Ch. 5 — Replication | Leader-follower; replication lag; failover; Write-Ahead Log |
| | Ch. 6 — Partitioning | Key-range vs hash partitioning; rebalancing; secondary indexes |
| | Ch. 7 — Transactions | ACID; distributed transactions (2PC); the Saga pattern |
| | Ch. 8 — The Trouble with Distributed Systems | Unreliable networks & clocks; partial failures; the Two Generals Problem |
| | Ch. 9 — Consistency & Consensus | Linearizability; CAP theorem; Raft/Paxos; coordination services |
| **III. Derived Data** | Ch. 10 — Batch Processing | Unix pipes → MapReduce; fan-out/fan-in; exactly-once processing |
| | Ch. 11 — Stream Processing | Event logs; event time vs processing time; windowing; state management |
| | Ch. 12 — The Future of Data Systems | Immutable event logs; schema evolution; designing for evolvability |

!!! info "You don't need to read DDIA before starting"
    Each module in this course explains the relevant DDIA concepts from scratch. Having the book is helpful for deeper reading, but **not required**.

---

## :material-view-list: Course Modules

| # | Module | DDIA Chapter | Temporal Concept |
|---|--------|-------------|-----------------|
| 0 | [Getting Started](introduction/setup.md) | — | Environment Setup & Hello World |
| 1 | [Foundations](modules/module-01-foundations.md) | Ch. 1 | Dev Setup & First Workflow |
| 2 | [Data Models & Activities](modules/module-02-data-models.md) | Ch. 2–3 | Activities & State |
| 3 | [Replication & Reliability](modules/module-03-replication.md) | Ch. 5 | Durable Execution & Retries |
| 4 | [Transactions & Saga](modules/module-04-transactions.md) | Ch. 7 | Saga Pattern |
| 5 | [Fault Tolerance](modules/module-05-fault-tolerance.md) | Ch. 8 | Heartbeats & Timeouts |
| 6 | [Distributed Coordination](modules/module-06-coordination.md) | Ch. 9 | Signals, Queries & Updates |
| 7 | [Batch & Stream Processing](modules/module-07-batch-stream.md) | Ch. 10–11 | Continue-As-New & Child Workflows |
| 8 | [Observability & Versioning](modules/module-08-observability.md) | Ch. 12 | Visibility, Tracing, Versioning |
| 9 | [Capstone Project](capstone/index.md) | All | Full Distributed E-Commerce System |

---

## :material-clock-outline: Time Commitment

**~32 hours total** · **5 weeks** · **TypeScript primary**

| Week | Modules | Hours |
|------|---------|-------|
| 1 | Module 1 + 2 | ~6 hrs |
| 2 | Module 3 + 4 | ~6 hrs |
| 3 | Module 5 + 6 | ~6 hrs |
| 4 | Module 7 + 8 | ~6 hrs |
| 5 | Capstone | ~8 hrs |

---

## :material-tools: Tech Stack

| Layer | Tool |
|-------|------|
| **Runtime** | Temporal (local dev server via CLI) |
| **Language** | TypeScript 5.x with `@temporalio/sdk` |
| **Runtime** | Node.js 20+ |
| **Database** | PostgreSQL (for activities) |
| **Observability** | Temporal Web UI + OpenTelemetry |
| **Testing** | Vitest + Temporal's time-skipping test server |
| **Docs** | MkDocs Material |

---

## :material-clipboard-check: Prerequisites

- [x] Basic TypeScript / JavaScript knowledge
- [x] Familiarity with HTTP/REST APIs
- [x] Understanding of what a database is
- [x] Node.js & npm installed
- [ ] **No prior distributed systems experience required!**

---

## :material-rocket-launch: Getting Started

<div class="grid cards" markdown>

-   :material-book-open-page-variant:{ .lg .middle } **Introduction**

    ---

    Learn who this course is for, why it exists, and what you'll build

    [:octicons-arrow-right-24: Read the Introduction](introduction/index.md)

-   :material-cog:{ .lg .middle } **Environment Setup**

    ---

    Install Temporal CLI, Node.js, and scaffold your first project

    [:octicons-arrow-right-24: Setup Guide](introduction/setup.md)

-   :material-head-lightbulb:{ .lg .middle } **Why This Course?**

    ---

    Understand how this course bridges DDIA theory with Temporal practice

    [:octicons-arrow-right-24: Why This Course](introduction/purpose.md)

-   :material-book-open-variant:{ .lg .middle } **Start Module 1**

    ---

    Learn the foundations of distributed systems and write your first workflow

    [:octicons-arrow-right-24: Module 1: Foundations](modules/module-01-foundations.md)

</div>

---

<div style="text-align: center; margin-top: 2rem;">
<em>"This course is designed to make you think like a distributed systems engineer, not just a Temporal developer."</em>
</div>
