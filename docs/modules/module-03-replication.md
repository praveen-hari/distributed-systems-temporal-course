# Module 3 — Replication & Reliability

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 5 — Replication  
    **Temporal Concepts:** Durable execution deep dive, Event History as a replicated log, retry semantics, delivery guarantees  
    **Time:** ~3 hours

---

## Part 1: Theory — Replication (DDIA Chapter 5)

### Why Replicate Data?

Kleppmann identifies three main reasons to replicate data across multiple machines:

1. **Keep data geographically close to users** — Reduce latency
2. **Allow the system to continue working if some parts fail** — Increase availability
3. **Scale out the number of machines that can serve read queries** — Increase read throughput

The fundamental challenge: **keeping replicas consistent** when data changes. All the difficulty in replication lies in handling *changes* to replicated data.

### Three Replication Approaches

#### Leader-Based (Single-Leader) Replication

```
┌──────────┐     Writes     ┌──────────┐
│  Client   │──────────────▶│  Leader   │
└──────────┘                └────┬─────┘
                                 │ Replication stream
                    ┌────────────┼────────────┐
                    ▼            ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Follower │ │ Follower │ │ Follower │
              └──────────┘ └──────────┘ └──────────┘
                    ▲            ▲             ▲
                    │            │             │
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  Client   │ │  Client   │ │  Client   │
              │  (reads)  │ │  (reads)  │ │  (reads)  │
              └──────────┘ └──────────┘ └──────────┘
```

- One replica is the **leader** (master/primary) — all writes go through it
- Other replicas are **followers** (slaves/secondaries) — they receive a replication stream
- Reads can go to any replica

This is the most common approach, used by PostgreSQL, MySQL, MongoDB, and many others.

#### Multi-Leader Replication

Multiple nodes accept writes. Used for multi-datacenter setups. Introduces **write conflicts** that must be resolved.

#### Leaderless Replication

Any replica can accept writes. Uses quorum reads/writes (e.g., Dynamo-style). No single point of failure for writes.

### Synchronous vs Asynchronous Replication

| Type | Behavior | Trade-off |
|------|----------|-----------|
| **Synchronous** | Leader waits for follower to confirm write | Guaranteed consistency, but slower and less available |
| **Asynchronous** | Leader doesn't wait for followers | Faster, more available, but followers may lag behind |

In practice, most systems use **semi-synchronous** replication: one follower is synchronous (guaranteed up-to-date), the rest are asynchronous.

### Replication Lag and Its Consequences

With asynchronous replication, followers may be behind the leader. This creates several consistency problems:

#### Read-After-Write Consistency

A user writes data, then reads it back — but the read goes to a follower that hasn't received the write yet.

```
User writes → Leader (success)
User reads  → Follower (stale — write not replicated yet)
Result: User doesn't see their own write!
```

#### Monotonic Reads

A user makes two reads, and the second read returns older data than the first (because it hit a different, more-lagged follower).

#### Consistent Prefix Reads

In a partitioned database, events may appear out of order if partitions replicate at different speeds.

### Handling Node Failures

#### Follower Failure: Catch-Up Recovery

When a follower recovers, it connects to the leader and requests all changes since its last known position in the replication log. This is called **catch-up recovery**.

#### Leader Failure: Failover

When the leader fails, one of the followers needs to be promoted. This process — **failover** — is fraught with danger:

- How do you determine the leader has failed? (Usually via timeout)
- How do you choose the new leader? (Election or appointed by controller)
- What happens to writes the old leader received but didn't replicate? (They may be lost)
- **Split-brain**: Two nodes both believe they are the leader

### The Write-Ahead Log (WAL)

Many databases use a **Write-Ahead Log** for crash recovery:

1. Before modifying data, write the intended change to a log
2. If the system crashes, replay the log to recover
3. The log is also used for replication — followers replay the leader's log

This concept — **durability through an append-only log** — is fundamental to understanding Temporal.

---

## Part 2: Mental Model — Temporal's Event History as a Replicated Log

### The Parallel to Database Replication

Temporal's architecture mirrors the concepts from DDIA Chapter 5 in a remarkable way:

| DDIA Concept | Temporal Equivalent |
|---|---|
| Write-Ahead Log (WAL) | Event History |
| Leader database | Temporal Service (persists events) |
| Follower catch-up recovery | Worker replay (re-executes workflow from history) |
| Replication stream | Task Queue (distributes work to Workers) |
| Failover to new leader | New Worker picks up where crashed Worker left off |

### The Event History Is Temporal's WAL

Just as a database writes changes to a WAL before applying them, Temporal records every workflow step as an **Event** before considering it complete:

```
Workflow Execution:
  1. Workflow starts        → WorkflowExecutionStarted event
  2. Activity scheduled     → ActivityTaskScheduled event
  3. Activity completes     → ActivityTaskCompleted event (with result)
  4. Another activity       → ActivityTaskScheduled event
  5. Activity completes     → ActivityTaskCompleted event (with result)
  6. Workflow completes     → WorkflowExecutionCompleted event
```

This log is **durable** — it survives Worker crashes, network partitions, and restarts.

### Worker Replay = Follower Catch-Up

When a Worker needs to restore workflow state (after a crash, cache eviction, or to continue after a long timer), it performs **replay** — exactly like a database follower catching up from the replication log:

```
┌─────────────────────────────────────────────────────────┐
│  Worker Replay (like follower catch-up)                  │
│                                                          │
│  1. Fetch Event History from Temporal Service            │
│  2. Re-execute workflow code from the beginning          │
│  3. For each activity call:                              │
│     - SDK checks: "Is there an ActivityTaskCompleted     │
│       event for this?"                                   │
│     - YES → Use the stored result (don't re-execute)     │
│     - NO  → This is new work, execute it for real        │
│  4. Workflow state is now restored                        │
└─────────────────────────────────────────────────────────┘
```

The key insight: **Activity results are never re-executed during replay**. Only the workflow orchestration logic is re-run. The stored results from the Event History are used instead.

### Delivery Guarantees

DDIA discusses delivery semantics extensively. Here's how Temporal maps:

| Guarantee | Meaning | Temporal Behavior |
|-----------|---------|-------------------|
| **At-most-once** | Message delivered 0 or 1 times | Not used — Temporal always retries |
| **At-least-once** | Message delivered 1 or more times | ✅ **Activity execution** — Activities may run more than once due to retries |
| **Effectively-once** | Effect happens exactly once | ✅ **Workflow execution** — The Event History ensures each step's *effect* is recorded once |

!!! warning "At-least-once, not exactly-once"
    Temporal provides **at-least-once** execution of Activities. An Activity may run multiple times (due to retries after failures). This is why **idempotency** (Module 2) is critical — your Activities must be safe to re-execute.

    However, the *workflow* achieves **effectively-once** semantics because the Event History records each step's completion. During replay, completed Activities are not re-executed — their stored results are used.

---

## Part 3: Deep Dive — The Retry Mechanism

### How Temporal Retries Work

When an Activity fails, Temporal doesn't just blindly retry. It follows a precise sequence:

```
Activity Attempt 1:
  Worker executes activity → Throws error
  Temporal records: ActivityTaskFailed (attempt 1)
  
  Wait: initialInterval (default 1s)

Activity Attempt 2:
  Worker executes activity → Throws error
  Temporal records: ActivityTaskFailed (attempt 2)
  
  Wait: initialInterval × backoffCoefficient (default 2s)

Activity Attempt 3:
  Worker executes activity → Success!
  Temporal records: ActivityTaskCompleted (with result)
```

Important: **Individual retry attempts are NOT separate events in the workflow history**. Only the final outcome (success or exhausted retries) creates a workflow-level event. This keeps the history compact.

### Retry Policy Configuration

```typescript
const { processPayment } = proxyActivities<typeof activities>({
  startToCloseTimeout: '30 seconds',
  retry: {
    initialInterval: '1s',        // Wait 1s before first retry
    backoffCoefficient: 2,         // Double the wait each time
    maximumInterval: '30s',        // Never wait more than 30s
    maximumAttempts: 5,            // Give up after 5 attempts
    nonRetryableErrorTypes: [      // Don't retry these errors
      'ValidationError',
      'NotFoundError',
    ],
  },
});
```

The retry timeline with these settings:

```
Attempt 1: Execute immediately
  → Fails
  Wait 1s (initialInterval)

Attempt 2: Execute
  → Fails
  Wait 2s (1s × 2)

Attempt 3: Execute
  → Fails
  Wait 4s (2s × 2)

Attempt 4: Execute
  → Fails
  Wait 8s (4s × 2)

Attempt 5: Execute
  → Fails
  → maximumAttempts reached → Activity fails permanently
  → Workflow receives ActivityFailure
```

### When NOT to Retry

Not all errors should be retried. Retrying a permanent error wastes resources and delays failure detection:

```typescript
import { ApplicationFailure } from '@temporalio/activity';

export async function chargeCard(cardNumber: string, amount: number): Promise<string> {
  const result = await paymentGateway.charge(cardNumber, amount);

  switch (result.status) {
    case 'success':
      return result.transactionId;

    case 'card_declined':
      // Permanent — retrying won't help
      throw ApplicationFailure.nonRetryable('Card declined');

    case 'insufficient_funds':
      // Permanent — retrying won't help
      throw ApplicationFailure.nonRetryable('Insufficient funds');

    case 'gateway_timeout':
      // Transient — Temporal will retry automatically
      throw new Error('Payment gateway timeout');

    case 'rate_limited':
      // Transient — Temporal will retry with backoff
      throw new Error('Rate limited');

    default:
      throw new Error(`Unexpected status: ${result.status}`);
  }
}
```

---

## Part 4: Workflow Error Handling

### Catching Activity Failures in Workflows

When an Activity exhausts all retries, the workflow receives an `ActivityFailure`. You can catch and handle it:

```typescript
import { proxyActivities, ApplicationFailure, log } from '@temporalio/workflow';
import type * as activities from '../activities/payment';

const { chargeCard } = proxyActivities<typeof activities>({
  startToCloseTimeout: '30 seconds',
  retry: {
    maximumAttempts: 3,
    nonRetryableErrorTypes: ['CardDeclinedError'],
  },
});

export async function paymentWorkflow(
  cardNumber: string,
  amount: number
): Promise<string> {
  try {
    const txnId = await chargeCard(cardNumber, amount);
    log.info('Payment successful', { txnId });
    return txnId;
  } catch (err) {
    if (err instanceof ApplicationFailure) {
      log.warn('Payment failed permanently', {
        type: err.type,
        message: err.message,
      });

      // You could try an alternative payment method here
      // or notify the user
    }
    throw err; // Re-throw to fail the workflow
  }
}
```

### Workflow Status Reference

Understanding workflow statuses helps you debug issues:

| Status | Meaning | What to Do |
|--------|---------|------------|
| `RUNNING` | Workflow in progress | Wait, or check if stalled (no Worker?) |
| `COMPLETED` | Successfully finished | Get result |
| `FAILED` | Error during execution | Check error message in Event History |
| `CANCELED` | Explicitly canceled | Review who canceled and why |
| `TERMINATED` | Force-stopped | Review who terminated and why |
| `TIMED_OUT` | Exceeded timeout | Increase timeout or optimize workflow |

### Common Error Types

| Error | Identifier | What Happened | Recovery |
|-------|-----------|---------------|----------|
| Non-determinism | TMPRL1100 | Replay doesn't match history | Fix code to match history → restart Worker |
| Deadlock | TMPRL1101 | Workflow blocked too long | Remove blocking operations from workflow code |
| Payload overflow | TMPRL1103 | Payload size limit exceeded | Reduce payload size, use external storage |
| Activity bug | — | Bug in activity code | Fix code → restart Worker → auto-retries |
| Task queue mismatch | — | Different queues in starter/Worker | Align task queue names |

---

## Part 5: Build It — Resilient Order Processing

Let's build a workflow that demonstrates Temporal's reliability features: retries, error handling, and crash recovery.

### The Scenario

An order processing workflow that:

1. Validates the order
2. Reserves inventory (may fail transiently)
3. Charges payment (may fail permanently or transiently)
4. Sends confirmation

We'll simulate various failure modes and observe how Temporal handles them.

### Activity Implementations

```typescript
// src/activities/order-processing.ts
import { ApplicationFailure, log } from '@temporalio/activity';

export interface Order {
  id: string;
  item: string;
  quantity: number;
  amount: number;
  email: string;
}

// Track attempts to demonstrate retry behavior
const inventoryAttempts = new Map<string, number>();

export async function validateOrder(order: Order): Promise<void> {
  log.info('Validating order', { orderId: order.id });

  if (!order.item || order.quantity <= 0) {
    throw ApplicationFailure.nonRetryable('Invalid order: missing item or invalid quantity');
  }
  if (order.amount <= 0) {
    throw ApplicationFailure.nonRetryable('Invalid order: amount must be positive');
  }

  log.info('Order validated', { orderId: order.id });
}

export async function reserveInventory(orderId: string, item: string, quantity: number): Promise<string> {
  log.info('Reserving inventory', { orderId, item, quantity });

  // Simulate transient failure on first attempt
  const attempts = (inventoryAttempts.get(orderId) ?? 0) + 1;
  inventoryAttempts.set(orderId, attempts);

  if (attempts <= 2) {
    log.warn('Inventory service temporarily unavailable', { orderId, attempt: attempts });
    throw new Error('Inventory service temporarily unavailable');
  }

  // Success on third attempt
  const reservationId = `res-${orderId}-${Date.now()}`;
  log.info('Inventory reserved', { orderId, reservationId });
  return reservationId;
}

export async function chargePayment(
  orderId: string,
  amount: number
): Promise<string> {
  log.info('Charging payment', { orderId, amount });

  // Simulate payment processing
  if (amount > 10000) {
    throw ApplicationFailure.nonRetryable(
      `Amount $${amount} exceeds maximum allowed`
    );
  }

  const transactionId = `txn-${orderId}-${Date.now()}`;
  log.info('Payment charged', { orderId, transactionId });
  return transactionId;
}

export async function sendConfirmation(
  email: string,
  orderId: string,
  transactionId: string
): Promise<void> {
  log.info('Sending confirmation email', { email, orderId, transactionId });
  // Simulate email sending
  log.info('Confirmation sent', { email });
}
```

### Workflow Definition

```typescript
// src/workflows/order-processing.ts
import { proxyActivities, log, sleep } from '@temporalio/workflow';
import type * as orderActivities from '../activities/order-processing';
import type { Order } from '../activities/order-processing';

const { validateOrder, reserveInventory, chargePayment, sendConfirmation } =
  proxyActivities<typeof orderActivities>({
    startToCloseTimeout: '10 seconds',
    retry: {
      initialInterval: '1s',
      backoffCoefficient: 2,
      maximumInterval: '10s',
      maximumAttempts: 5,
    },
  });

export interface OrderResult {
  orderId: string;
  reservationId: string;
  transactionId: string;
  status: 'completed' | 'failed';
}

export async function orderProcessingWorkflow(order: Order): Promise<OrderResult> {
  log.info('Starting order processing', { orderId: order.id });

  // Step 1: Validate (non-retryable errors fail immediately)
  await validateOrder(order);

  // Step 2: Reserve inventory (will retry on transient failures)
  log.info('Reserving inventory...', { orderId: order.id });
  const reservationId = await reserveInventory(order.id, order.item, order.quantity);

  // Step 3: Charge payment
  log.info('Charging payment...', { orderId: order.id });
  const transactionId = await chargePayment(order.id, order.amount);

  // Step 4: Send confirmation
  await sendConfirmation(order.email, order.id, transactionId);

  log.info('Order processing complete', { orderId: order.id });

  return {
    orderId: order.id,
    reservationId,
    transactionId,
    status: 'completed',
  };
}
```

### Worker

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as orderActivities from './activities/order-processing';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/order-processing'),
    activities: orderActivities,
    taskQueue: 'order-queue',
  });

  console.log('Worker started on task queue: order-queue');
  await worker.run();
}

run().catch((err) => {
  console.error('Worker failed:', err);
  process.exit(1);
});
```

### Client

```typescript
// src/client.ts
import { Client } from '@temporalio/client';
import { orderProcessingWorkflow } from './workflows/order-processing';
import type { Order } from './activities/order-processing';

async function run() {
  const client = new Client();

  // Test 1: Successful order (with transient inventory failures)
  console.log('=== Test 1: Order with transient failures ===');
  const order1: Order = {
    id: 'order-001',
    item: 'Mechanical Keyboard',
    quantity: 1,
    amount: 149.99,
    email: 'alice@example.com',
  };

  const result1 = await client.workflow.execute(orderProcessingWorkflow, {
    workflowId: `order-${order1.id}`,
    taskQueue: 'order-queue',
    args: [order1],
  });
  console.log('Result:', JSON.stringify(result1, null, 2));

  // Test 2: Invalid order (fails immediately, no retries)
  console.log('\n=== Test 2: Invalid order ===');
  const order2: Order = {
    id: 'order-002',
    item: '',
    quantity: 0,
    amount: 0,
    email: 'bob@example.com',
  };

  try {
    await client.workflow.execute(orderProcessingWorkflow, {
      workflowId: `order-${order2.id}`,
      taskQueue: 'order-queue',
      args: [order2],
    });
  } catch (err) {
    console.log('Expected failure:', (err as Error).message);
  }

  // Test 3: Amount too high (non-retryable payment error)
  console.log('\n=== Test 3: Payment exceeds limit ===');
  const order3: Order = {
    id: 'order-003',
    item: 'Luxury Watch',
    quantity: 1,
    amount: 50000,
    email: 'carol@example.com',
  };

  try {
    await client.workflow.execute(orderProcessingWorkflow, {
      workflowId: `order-${order3.id}`,
      taskQueue: 'order-queue',
      args: [order3],
    });
  } catch (err) {
    console.log('Expected failure:', (err as Error).message);
  }
}

run().catch(console.error);
```

### Run It and Observe

1. **Start the Temporal dev server:** `temporal server start-dev`
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`

**What to observe in the Web UI ([http://localhost:8233](http://localhost:8233)):**

- **`order-order-001`** — Completed. Expand the Event History and look for:
    - `ActivityTaskScheduled` for `reserveInventory`
    - `ActivityTaskFailed` (attempt 1 — transient error)
    - `ActivityTaskFailed` (attempt 2 — transient error)
    - `ActivityTaskCompleted` (attempt 3 — success!)
    - The workflow continued seamlessly after the retries

- **`order-order-002`** — Failed. The `validateOrder` activity threw a non-retryable error. No retry attempts in the history.

- **`order-order-003`** — Failed at `chargePayment`. Inventory was reserved successfully, but payment failed with a non-retryable error.

---

## Part 6: Experiments

### Experiment 1: Crash Recovery

This is the most powerful demonstration of Temporal's reliability:

1. Start the worker
2. Start a workflow for `order-001`
3. **Kill the worker** (Ctrl+C) immediately after you see "Reserving inventory..." in the logs
4. Check the Web UI — the workflow is still `RUNNING`
5. **Restart the worker**
6. Watch the workflow **resume from where it left off** — it doesn't re-validate the order or re-reserve inventory

This is durable execution in action. The Event History recorded each completed step, and the new Worker replayed the history to restore state.

### Experiment 2: Inspect the Event History via CLI

```bash
temporal workflow show --workflow-id order-order-001
```

Study the events carefully. You'll see the complete lifecycle:

- `WorkflowExecutionStarted` — with the order input
- `ActivityTaskScheduled` / `ActivityTaskCompleted` pairs for each activity
- `ActivityTaskFailed` events for the transient inventory failures
- `WorkflowExecutionCompleted` — with the final result

### Experiment 3: Multiple Workers

Start two workers simultaneously:

```bash
# Terminal 1
npx ts-node src/worker.ts

# Terminal 2
npx ts-node src/worker.ts
```

Run several workflows. Observe in the Web UI that different activities may be executed by different Workers. Temporal distributes work across all Workers polling the same Task Queue — this is horizontal scaling.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 5** teaches that replication is essential for reliability, but introduces consistency challenges. The Write-Ahead Log (WAL) is the foundation of crash recovery in databases.

    2. **Temporal's Event History is analogous to a WAL** — it's a durable, append-only log of every step in a workflow. Worker replay is analogous to follower catch-up recovery in database replication.

    3. **Temporal provides at-least-once Activity execution** — Activities may run more than once due to retries. This is why idempotency is critical. The workflow achieves effectively-once semantics through the Event History.

    4. **Retry policies** control how Temporal handles transient failures:
        - `initialInterval` + `backoffCoefficient` = exponential backoff
        - `maximumAttempts` = give up after N tries
        - `nonRetryableErrorTypes` = don't retry permanent errors

    5. **Error classification is essential** — Use `ApplicationFailure.nonRetryable()` for permanent errors (invalid input, not found). Let transient errors (network timeout, rate limit) retry automatically.

    6. **Crash recovery is automatic** — Kill a Worker mid-workflow, restart it, and the workflow resumes from the last completed step. No recovery code needed.

---

## Next Module

[:octicons-arrow-right-24: Module 4 — Transactions & the Saga Pattern](module-04-transactions.md)
