# Capstone Project: Distributed E-Commerce System

!!! note "Coming Soon"
    This module will be available after completing Modules 1–8.

The capstone project integrates every concept from the course into a complete **E-Commerce Order Processing System** built with Temporal and TypeScript.

---

## What You'll Build

```
Customer Places Order
        │
        ▼
[Order Workflow] ─── Signal: Cancel/Update
        │
        ├──► [Payment Activity] (retry + idempotency)
        │
        ├──► [Inventory Saga] (reserve → compensate on failure)
        │
        ├──► [Shipping Child Workflow] (heartbeat, long-running)
        │
        ├──► [Notification Activity] (at-least-once, idempotent)
        │
        └──► [Analytics Batch Job] (scheduled, continue-as-new)
```

## Concepts Applied

- **Module 1–2**: Project structure, activities, workflows
- **Module 3**: Retry policies, durable execution
- **Module 4**: Saga pattern with compensation
- **Module 5**: Heartbeats, timeouts, fault tolerance
- **Module 6**: Signals, queries, updates for order management
- **Module 7**: Child workflows, continue-as-new for analytics
- **Module 8**: Versioning, observability, search attributes
