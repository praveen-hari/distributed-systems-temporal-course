# Introduction

Welcome to **Reliable Distributed Systems with Temporal** — a hands-on course that bridges distributed systems theory with practical implementation using TypeScript and Temporal.

---

## Who Is This Course For?

This course is designed for:

- **Backend developers** who want to understand why distributed systems fail and how to build reliable ones
- **TypeScript/Node.js developers** looking to build production-grade distributed applications
- **Engineers** who have heard of concepts like "eventual consistency" or "saga pattern" but want to truly understand and implement them
- **Anyone** who wants to move beyond simple REST APIs into the world of durable, fault-tolerant systems

---

## What You'll Learn

By the end of this course, you will:

1. **Understand** the fundamental challenges of distributed systems (from DDIA by Martin Kleppmann)
2. **Implement** solutions using Temporal's durable execution model
3. **Build** production-ready patterns: retries, sagas, heartbeats, signals, versioning
4. **Debug** distributed system failures with confidence
5. **Design** systems that survive crashes, network partitions, and deployment changes

---

## What You'll Build

Across the 8 modules and capstone project, you'll progressively build:

| Module | What You Build |
|--------|---------------|
| Module 1 | A user signup workflow that survives crashes |
| Module 2 | Activities with proper data handling and idempotency |
| Module 3 | A workflow that demonstrates durable execution and retry policies |
| Module 4 | A travel booking saga with compensation logic |
| Module 5 | Long-running activities with heartbeats and timeout strategies |
| Module 6 | An order management system with signals, queries, and updates |
| Module 7 | A document processing pipeline with child workflows and continue-as-new |
| Module 8 | Versioned workflows with observability and search attributes |
| Capstone | A full e-commerce order processing system combining everything |

---

## The Two Pillars of This Course

### 📖 Designing Data-Intensive Applications (DDIA)

Martin Kleppmann's book is widely regarded as the best resource for understanding distributed systems. It explains *why* distributed systems are hard — unreliable networks, inconsistent clocks, partial failures, and the impossibility results that constrain what's achievable.

We use DDIA as our **theory foundation**. Each module maps to specific chapters.

### ⚡ Temporal

Temporal is a durable execution platform that provides practical solutions to the problems DDIA describes. Instead of writing complex retry logic, state machines, and recovery code, you write straightforward TypeScript functions and Temporal handles the rest.

We use Temporal as our **implementation platform**. Every theory concept gets a working code example.

---

## Prerequisites

Before starting, ensure you have:

- [x] Basic TypeScript / JavaScript knowledge
- [x] Familiarity with async/await and Promises
- [x] Understanding of what a database is
- [x] Node.js 20+ and npm installed on your machine
- [ ] **No prior distributed systems experience required!**

---

## Getting Started

<div class="grid cards" markdown>

-   :material-help-circle:{ .lg .middle } **Why This Course?**

    ---

    Understand the gap between theory and practice that this course bridges

    [:octicons-arrow-right-24: Why This Course](purpose.md)

-   :material-cog:{ .lg .middle } **Environment Setup**

    ---

    Install Temporal CLI, Node.js, TypeScript SDK, and verify everything works

    [:octicons-arrow-right-24: Setup Guide](setup.md)

</div>
