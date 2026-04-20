# Reliable Distributed Systems with Temporal

> *"Understand WHY distributed systems fail → Learn HOW Temporal solves it → BUILD it yourself"*

A comprehensive course bridging **distributed systems theory** from Martin Kleppmann's [Designing Data-Intensive Applications (DDIA)](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-(2017).pdf) with **hands-on implementation** using [Temporal](https://temporal.io/) and TypeScript.

---

## 📚 What Is This?

This is a self-paced course that teaches you to build reliable distributed systems by connecting theory to practice:

- **Theory** — Each module explains distributed systems concepts from DDIA
- **Mental Model** — How Temporal addresses each problem and why
- **Build It** — TypeScript code you write, run, break, and fix

## 🗂️ Course Modules

| # | Module | DDIA Chapter | Temporal Concept |
|---|--------|-------------|-----------------|
| 0 | Getting Started | — | Environment Setup & Hello World |
| 1 | Foundations | Ch. 1 | Workflows, Activities, Workers, Event History |
| 2 | Data Models & Activities | Ch. 2–3 | Activity design, idempotency, error classification |
| 3 | Replication & Reliability | Ch. 5 | Durable execution, retries, delivery guarantees |
| 4 | Transactions & Saga | Ch. 7 | Saga pattern, compensation, CancellationScope |
| 5 | Fault Tolerance | Ch. 8 | Heartbeats, timeouts, cancellation delivery |
| 6 | Distributed Coordination | Ch. 9 | Signals, Queries, Updates, condition() |
| 7 | Batch & Stream Processing | Ch. 10–11 | Child Workflows, Continue-As-New, Schedules |
| 8 | Observability & Versioning | Ch. 12 | Patching API, replay testing, search attributes |
| 9 | Capstone Project | All | Full E-Commerce Order Processing System |

## ⏱️ Time Commitment

**~32 hours** over **5 weeks**

| Week | Modules | Hours |
|------|---------|-------|
| 1 | Module 1 + 2 | ~6 hrs |
| 2 | Module 3 + 4 | ~6 hrs |
| 3 | Module 5 + 6 | ~6 hrs |
| 4 | Module 7 + 8 | ~6 hrs |
| 5 | Capstone | ~8 hrs |

## 🛠️ Tech Stack

| Layer | Tool |
|-------|------|
| Runtime | [Temporal](https://temporal.io/) (local dev server via CLI) |
| Language | TypeScript 5.x with `@temporalio/sdk` |
| Runtime | Node.js 20+ |
| Observability | Temporal Web UI |
| Testing | Temporal's time-skipping test server |
| Docs | [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) |

## ✅ Prerequisites

- Basic TypeScript / JavaScript knowledge
- Familiarity with async/await and Promises
- Node.js 20+ and npm installed
- **No prior distributed systems experience required!**

## 🚀 Getting Started

### View the Course

```bash
# Clone the repo
git clone https://github.com/praveen-hari/distributed-systems-temporal-course.git
cd distributed-systems-temporal-course

# Create virtual environment and install MkDocs
python3 -m venv venv
source venv/bin/activate
pip install mkdocs-material

# Serve locally
mkdocs serve
```

Then open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser.

### Install Temporal CLI

```bash
# macOS
brew install temporal

# Verify
temporal --version
```

### Start the Dev Server

```bash
temporal server start-dev
```

Open the Temporal Web UI at [http://localhost:8233](http://localhost:8233).

## 📖 References

- [Designing Data-Intensive Applications (DDIA)](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-(2017).pdf) — Martin Kleppmann
- [Temporal Documentation](https://docs.temporal.io/)
- [Temporal TypeScript SDK](https://typescript.temporal.io/)
- [Temporal Community Slack](https://t.mp/slack)

## 📄 License

This course content is provided for educational purposes.

---

*"This course is designed to make you think like a distributed systems engineer, not just a Temporal developer."*
