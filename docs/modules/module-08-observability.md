# Module 8 — Observability & Versioning

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 12 — The Future of Data Systems  
    **Temporal Concepts:** Workflow versioning (`patched`/`deprecatePatch`), replay testing, search attributes, replay-aware logging, testing  
    **Time:** ~3 hours

---

## Part 1: Theory — The Future of Data Systems (DDIA Chapter 12)

### Designing for Evolvability

Kleppmann's final chapter looks forward. A key theme: **systems must evolve over time**, and the design must accommodate change.

> "The most important thing is not the tools themselves, but the principles behind them."

Key ideas from Chapter 12:

**Immutable event logs** — Instead of mutable state, record an immutable log of events. The current state is derived by replaying the log. This enables:

- Auditing (what happened and when)
- Debugging (replay to reproduce issues)
- Evolution (reprocess events with new logic)

**Schema evolution** — Data formats change over time. Systems must handle:

- Old code reading new data (forward compatibility)
- New code reading old data (backward compatibility)

**Designing for operability** — Production systems need:

- **Observability** — Understand what the system is doing (logs, metrics, traces)
- **Debuggability** — Reproduce and diagnose issues
- **Deployability** — Ship changes safely without breaking running processes

### The Three Pillars of Observability

| Pillar | What It Provides | Example |
|--------|-----------------|---------|
| **Logs** | Discrete events with context | "Order 123 payment failed: card declined" |
| **Metrics** | Aggregated numerical measurements | "p99 latency: 250ms, error rate: 0.1%" |
| **Traces** | Request flow across services | "Order 123: API → Workflow → Payment → DB" |

---

## Part 2: Mental Model — Evolving Temporal Workflows

### The Versioning Problem

Temporal workflows can run for minutes, hours, days, or even years. During that time, you **will** need to change the workflow code. But changing code for running workflows is dangerous:

```
Original Code (recorded in history):
  await activityA()
  await activityB()

Updated Code (during replay):
  await activityA()
  await activityC()  ← Different! NondeterminismError!
```

The Worker replays the Event History to restore state. If the new code produces different Commands than what's in the history, Temporal raises a `NondeterminismError` and the workflow becomes blocked.

### Three Versioning Approaches

| Approach | How It Works | Best For |
|----------|-------------|----------|
| **Patching API** | Branch code based on patch markers in history | Small, incremental changes |
| **Workflow Type Versioning** | Create new workflow function (e.g., `orderWorkflowV2`) | Major rewrites |
| **Worker Versioning** | Route workflows to specific Worker builds | Frequent deployments, short-lived workflows |

### What Changes Require Versioning?

| Change | Requires Versioning? |
|--------|---------------------|
| Adding/removing/reordering activities | ✅ Yes |
| Changing which activity is called | ✅ Yes |
| Changing activity implementations | ❌ No (activities aren't replayed) |
| Changing activity arguments | ❌ No |
| Changing retry policies | ❌ No |
| Changing timer durations | ❌ No |
| Adding new signal/query/update handlers | ❌ No (additive changes are safe) |

---

## Part 3: The Patching API

### The `patched()` Function

```typescript
import { patched } from '@temporalio/workflow';

export async function orderWorkflow(orderId: string): Promise<string> {
  await validateOrder(orderId);

  if (patched('add-fraud-check')) {
    // New code path: added fraud check
    await checkFraud(orderId);
  }
  // Old workflows (without the patch marker) skip the fraud check

  await chargePayment(orderId);
  await shipOrder(orderId);
  return 'completed';
}
```

**How `patched()` works:**

- **First execution** (new workflow): Returns `true`, inserts a marker into the Event History
- **Replay with marker**: Returns `true` (takes new path)
- **Replay without marker**: Returns `false` (takes old path)

### The Three-Step Patching Process

This is critical to follow correctly:

#### Step 1: Patch In — Deploy Both Paths

```typescript
import { patched } from '@temporalio/workflow';

export async function shippingWorkflow(orderId: string): Promise<void> {
  if (patched('email-instead-of-fax')) {
    await sendEmail(orderId);    // New code
  } else {
    await sendFax(orderId);      // Old code (for replay)
  }
  await sleep('1 day');
}
```

At this point:

- **New workflows** → `patched()` returns `true` → sends email
- **Old workflows replaying** → `patched()` returns `false` → sends fax (matches history)

#### Step 2: Deprecate — Remove Old Code

Once **all old workflows have completed**, deprecate the patch:

```typescript
import { deprecatePatch } from '@temporalio/workflow';

export async function shippingWorkflow(orderId: string): Promise<void> {
  deprecatePatch('email-instead-of-fax');
  await sendEmail(orderId);
  await sleep('1 day');
}
```

`deprecatePatch()` records a marker that doesn't fail replay when absent — a transition period.

#### Step 3: Remove — Clean Up

After **all deprecated workflows have completed**, remove the patch entirely:

```typescript
export async function shippingWorkflow(orderId: string): Promise<void> {
  await sendEmail(orderId);
  await sleep('1 day');
}
```

### Finding Workflows by Version

Use List Filters to check before removing old code:

```bash
# Find running workflows with the patch
temporal workflow list --query \
  'WorkflowType = "shippingWorkflow" AND ExecutionStatus = "Running" AND TemporalChangeVersion = "email-instead-of-fax"'

# Find running workflows WITHOUT the patch (pre-patch)
temporal workflow list --query \
  'WorkflowType = "shippingWorkflow" AND ExecutionStatus = "Running" AND TemporalChangeVersion IS NULL'
```

!!! danger "Never skip steps"
    Removing old code while old workflows are still running causes `NondeterminismError`. Always verify no running workflows exist before moving to the next step.

### Workflow Type Versioning

For major rewrites, create a new workflow function:

```typescript
// Original
export async function orderWorkflow(order: Order): Promise<string> {
  // v1 implementation
}

// New version
export async function orderWorkflowV2(order: Order): Promise<string> {
  // v2 implementation — completely different logic
}
```

Both are registered with the Worker. New workflows use V2, old ones continue on V1.

---

## Part 4: Observability

### Replay-Aware Logging

Temporal's workflow logger automatically suppresses duplicate messages during replay:

```typescript
// In workflows — replay-aware
import { log } from '@temporalio/workflow';

export async function orderWorkflow(orderId: string): Promise<string> {
  log.info('Processing order', { orderId });
  // During replay, this log is suppressed (already happened)

  const result = await processPayment(orderId);
  log.info('Payment processed', { orderId, result });

  return result;
}
```

```typescript
// In activities — contextual metadata added automatically
import { log } from '@temporalio/activity';

export async function processPayment(orderId: string): Promise<string> {
  log.info('Charging payment', { orderId });
  return 'txn-123';
}
```

**Log levels:** `log.debug()`, `log.info()`, `log.warn()`, `log.error()`

!!! tip "Workflow `log` vs `console.log`"
    Use `log` from `@temporalio/workflow` for production observability — it's replay-aware and routes through sinks. For temporary debugging, `console.log()` is fine — it's direct and immediate, but `log` may lose messages on workflow errors.

### Custom Logger with Winston

```typescript
import winston from 'winston';
import { DefaultLogger, Runtime } from '@temporalio/worker';

const winstonLogger = winston.createLogger({
  level: 'debug',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'temporal.log' }),
  ],
});

const logger = new DefaultLogger('DEBUG', (entry) => {
  winstonLogger.log({
    label: entry.meta?.activityId ? 'activity'
         : entry.meta?.workflowId ? 'workflow'
         : 'worker',
    level: entry.level.toLowerCase(),
    message: entry.message,
    timestamp: Number(entry.timestampNanos / 1_000_000n),
    ...entry.meta,
  });
});

Runtime.install({ logger });
```

### Search Attributes

Search attributes make workflows **searchable and filterable** in the Web UI and via the API:

```typescript
// Set at workflow start (client-side)
await client.workflow.start(orderWorkflow, {
  workflowId: `order-${orderId}`,
  taskQueue: 'orders',
  args: [order],
  searchAttributes: {
    OrderId: [orderId],
    CustomerType: ['premium'],
    OrderTotal: [99.99],
  },
});
```

```typescript
// Update from within the workflow
import { upsertSearchAttributes } from '@temporalio/workflow';

export async function orderWorkflow(order: Order): Promise<string> {
  upsertSearchAttributes({ OrderStatus: ['processing'] });

  await processOrder(order);

  upsertSearchAttributes({ OrderStatus: ['completed'] });
  return 'done';
}
```

```typescript
// Query workflows by search attributes
for await (const wf of client.workflow.list({
  query: 'OrderStatus = "processing" AND CustomerType = "premium"',
})) {
  console.log(`Workflow ${wf.workflowId} is still processing`);
}
```

### Prometheus Metrics

```typescript
import { Runtime } from '@temporalio/worker';

Runtime.install({
  telemetryOptions: {
    metrics: {
      prometheus: {
        bindAddress: '127.0.0.1:9091',
      },
    },
  },
});
```

---

## Part 5: Testing

### Test Environment Setup

```typescript
import { TestWorkflowEnvironment } from '@temporalio/testing';
import { Worker } from '@temporalio/worker';

describe('Order Workflow', () => {
  let testEnv: TestWorkflowEnvironment;

  before(async () => {
    testEnv = await TestWorkflowEnvironment.createLocal();
  });

  after(async () => {
    await testEnv?.teardown();
  });

  it('processes order successfully', async () => {
    const { client, nativeConnection } = testEnv;

    const worker = await Worker.create({
      connection: nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('./workflows'),
      activities: require('./activities'),
    });

    await worker.runUntil(async () => {
      const result = await client.workflow.execute(orderWorkflow, {
        taskQueue: 'test',
        workflowId: 'test-order-1',
        args: ['order-123'],
      });
      expect(result).toEqual('completed');
    });
  });
});
```

### Activity Mocking

Replace real activities with mocks to test workflow logic in isolation:

```typescript
const worker = await Worker.create({
  connection: nativeConnection,
  taskQueue: 'test',
  workflowsPath: require.resolve('./workflows'),
  activities: {
    // Mock implementations
    chargePayment: async (orderId: string) => `mock-txn-${orderId}`,
    sendEmail: async () => {},
    checkInventory: async () => true,
  },
});
```

### Testing Signals and Queries

```typescript
it('handles approval signal', async () => {
  await worker.runUntil(async () => {
    const handle = await client.workflow.start(approvalWorkflow, {
      taskQueue: 'test',
      workflowId: 'test-approval',
    });

    // Query initial state
    const status = await handle.query(getStatusQuery);
    expect(status).toEqual('pending');

    // Send approval signal
    await handle.signal(approveSignal);

    // Wait for completion
    const result = await handle.result();
    expect(result).toEqual('approved');
  });
});
```

### Testing Failure Cases

```typescript
it('handles payment failure', async () => {
  const worker = await Worker.create({
    connection: nativeConnection,
    taskQueue: 'test',
    workflowsPath: require.resolve('./workflows'),
    activities: {
      chargePayment: async () => {
        throw ApplicationFailure.nonRetryable('Card declined');
      },
    },
  });

  await worker.runUntil(async () => {
    try {
      await client.workflow.execute(orderWorkflow, {
        workflowId: 'test-failure',
        taskQueue: 'test',
      });
      fail('Expected workflow to fail');
    } catch (err) {
      expect(err).toBeInstanceOf(WorkflowFailedError);
    }
  });
});
```

### Replay Testing

Replay tests verify that code changes are compatible with existing workflow histories:

```typescript
import { Worker } from '@temporalio/worker';
import fs from 'fs';

it('replays workflow history without errors', async () => {
  // Load history exported from Web UI or CLI
  const history = JSON.parse(
    await fs.promises.readFile('./fixtures/order-workflow-history.json', 'utf8')
  );

  // This throws if current code is incompatible with the history
  await Worker.runReplayHistory(
    { workflowsPath: require.resolve('./workflows') },
    history
  );
});
```

**How to export history:**

```bash
# Export from CLI
temporal workflow show --workflow-id order-123 --output json > fixtures/order-workflow-history.json
```

!!! tip "Always replay test before deploying workflow changes"
    Save histories from production workflows and replay them against your new code before deploying. This catches non-determinism errors before they reach production.

### Activity Testing in Isolation

```typescript
import { MockActivityEnvironment } from '@temporalio/testing';

it('processes payment correctly', async () => {
  const env = new MockActivityEnvironment();
  const result = await env.run(chargePayment, 'order-123', 99.99);
  expect(result).toMatch(/^txn-/);
});

it('handles cancellation', async () => {
  const env = new MockActivityEnvironment();
  setTimeout(() => env.cancel(), 100);

  try {
    await env.run(longRunningActivity, 'input');
    fail('Expected cancellation');
  } catch (err) {
    expect(err).toBeInstanceOf(CancelledFailure);
  }
});
```

---

## Part 6: Build It — Versioned Order Workflow with Observability

Let's build a workflow that demonstrates versioning and observability in practice.

### The Scenario

We have an existing order workflow. We need to:

1. Add a fraud check step (requires patching)
2. Add search attributes for visibility
3. Add replay-aware logging
4. Write tests including replay tests

### Version 1: Original Workflow

```typescript
// src/workflows/order-v1.ts
import { proxyActivities, log, upsertSearchAttributes } from '@temporalio/workflow';
import type * as orderActivities from '../activities/order-ops';

const { validateOrder, chargePayment, shipOrder, sendConfirmation } =
  proxyActivities<typeof orderActivities>({
    startToCloseTimeout: '1 minute',
  });

export async function orderWorkflow(orderId: string, amount: number): Promise<string> {
  log.info('Starting order workflow', { orderId, amount });
  upsertSearchAttributes({ OrderStatus: ['started'], OrderAmount: [amount] });

  await validateOrder(orderId);
  upsertSearchAttributes({ OrderStatus: ['validated'] });

  const txnId = await chargePayment(orderId, amount);
  log.info('Payment charged', { orderId, txnId });
  upsertSearchAttributes({ OrderStatus: ['paid'] });

  await shipOrder(orderId);
  upsertSearchAttributes({ OrderStatus: ['shipped'] });

  await sendConfirmation(orderId);
  upsertSearchAttributes({ OrderStatus: ['completed'] });

  log.info('Order completed', { orderId });
  return txnId;
}
```

### Version 2: Add Fraud Check (Patched)

```typescript
// src/workflows/order-v2.ts (replaces order-v1.ts)
import { proxyActivities, log, upsertSearchAttributes, patched } from '@temporalio/workflow';
import type * as orderActivities from '../activities/order-ops';

const { validateOrder, checkFraud, chargePayment, shipOrder, sendConfirmation } =
  proxyActivities<typeof orderActivities>({
    startToCloseTimeout: '1 minute',
  });

export async function orderWorkflow(orderId: string, amount: number): Promise<string> {
  log.info('Starting order workflow', { orderId, amount });
  upsertSearchAttributes({ OrderStatus: ['started'], OrderAmount: [amount] });

  await validateOrder(orderId);
  upsertSearchAttributes({ OrderStatus: ['validated'] });

  // PATCH: Add fraud check for new workflows
  if (patched('add-fraud-check')) {
    log.info('Running fraud check', { orderId });
    await checkFraud(orderId, amount);
    upsertSearchAttributes({ OrderStatus: ['fraud-checked'] });
  }

  const txnId = await chargePayment(orderId, amount);
  log.info('Payment charged', { orderId, txnId });
  upsertSearchAttributes({ OrderStatus: ['paid'] });

  await shipOrder(orderId);
  upsertSearchAttributes({ OrderStatus: ['shipped'] });

  await sendConfirmation(orderId);
  upsertSearchAttributes({ OrderStatus: ['completed'] });

  log.info('Order completed', { orderId });
  return txnId;
}
```

### Activities

```typescript
// src/activities/order-ops.ts
import { ApplicationFailure, log } from '@temporalio/activity';

export async function validateOrder(orderId: string): Promise<void> {
  log.info('Validating order', { orderId });
}

export async function checkFraud(orderId: string, amount: number): Promise<void> {
  log.info('Checking fraud', { orderId, amount });
  if (amount > 10000) {
    throw ApplicationFailure.nonRetryable('Fraud detected: amount too high');
  }
}

export async function chargePayment(orderId: string, amount: number): Promise<string> {
  log.info('Charging payment', { orderId, amount });
  return `txn-${orderId}-${Date.now()}`;
}

export async function shipOrder(orderId: string): Promise<void> {
  log.info('Shipping order', { orderId });
}

export async function sendConfirmation(orderId: string): Promise<void> {
  log.info('Sending confirmation', { orderId });
}
```

### Tests

```typescript
// src/__tests__/order-workflow.test.ts
import { TestWorkflowEnvironment } from '@temporalio/testing';
import { Worker } from '@temporalio/worker';
import { WorkflowFailedError } from '@temporalio/client';
import { orderWorkflow } from '../workflows/order-v2';

describe('Order Workflow', () => {
  let testEnv: TestWorkflowEnvironment;

  before(async () => {
    testEnv = await TestWorkflowEnvironment.createLocal();
  });

  after(async () => {
    await testEnv?.teardown();
  });

  it('completes successfully with fraud check', async () => {
    const { client, nativeConnection } = testEnv;

    const worker = await Worker.create({
      connection: nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows/order-v2'),
      activities: {
        validateOrder: async () => {},
        checkFraud: async () => {},
        chargePayment: async (id: string) => `mock-txn-${id}`,
        shipOrder: async () => {},
        sendConfirmation: async () => {},
      },
    });

    await worker.runUntil(async () => {
      const result = await client.workflow.execute(orderWorkflow, {
        taskQueue: 'test',
        workflowId: 'test-order-success',
        args: ['order-1', 99.99],
      });
      expect(result).toMatch(/^mock-txn-/);
    });
  });

  it('fails on fraud detection', async () => {
    const { client, nativeConnection } = testEnv;

    const worker = await Worker.create({
      connection: nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows/order-v2'),
      activities: {
        validateOrder: async () => {},
        checkFraud: async (_id: string, amount: number) => {
          if (amount > 10000) throw new Error('Fraud detected');
        },
        chargePayment: async (id: string) => `mock-txn-${id}`,
        shipOrder: async () => {},
        sendConfirmation: async () => {},
      },
    });

    await worker.runUntil(async () => {
      try {
        await client.workflow.execute(orderWorkflow, {
          taskQueue: 'test',
          workflowId: 'test-order-fraud',
          args: ['order-2', 50000],
        });
        throw new Error('Expected failure');
      } catch (err) {
        expect(err).toBeInstanceOf(WorkflowFailedError);
      }
    });
  });
});
```

### Replay Test

```typescript
// src/__tests__/replay.test.ts
import { Worker } from '@temporalio/worker';
import fs from 'fs';

describe('Replay Compatibility', () => {
  it('v2 code replays v1 history correctly', async () => {
    // This history was exported from a v1 workflow execution
    const history = JSON.parse(
      await fs.promises.readFile('./fixtures/order-v1-history.json', 'utf8')
    );

    // Should NOT throw — v2 code with patched() handles v1 histories
    await Worker.runReplayHistory(
      { workflowsPath: require.resolve('../workflows/order-v2') },
      history
    );
  });
});
```

**Export a v1 history for testing:**

```bash
# Run a v1 workflow, then export its history
temporal workflow show --workflow-id order-v1-test --output json > fixtures/order-v1-history.json
```

---

## Part 7: Experiments

### Experiment 1: Deploy a Breaking Change (Without Patching)

1. Start a workflow with v1 code
2. While it's running (add a `sleep('1 minute')` to make it long-running), change the code to v2 **without** using `patched()`
3. Restart the Worker
4. Watch the workflow fail with `NondeterminismError`
5. Revert the code, restart the Worker — the workflow auto-recovers

### Experiment 2: Deploy Safely with Patching

1. Start a workflow with v1 code
2. Deploy v2 code **with** `patched('add-fraud-check')`
3. Restart the Worker
4. The old workflow replays correctly (skips fraud check)
5. Start a new workflow — it includes the fraud check

### Experiment 3: Search Attributes in the Web UI

1. Run several order workflows with different amounts
2. In the Web UI, use the search bar:
    ```
    OrderStatus = "paid" AND OrderAmount > 100
    ```
3. See how search attributes enable powerful filtering

### Experiment 4: Replay Test Workflow

1. Export a workflow history: `temporal workflow show --workflow-id order-1 --output json > history.json`
2. Run the replay test against it
3. Make a breaking change to the workflow code
4. Run the replay test again — it fails, catching the non-determinism before production

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 12** teaches that systems must evolve over time. Immutable event logs, schema evolution, and observability are essential for long-lived systems.

    2. **Workflow versioning** is necessary because running workflows replay their Event History. Three approaches:
        - **Patching API** (`patched`/`deprecatePatch`) — For incremental changes. Three-step process: patch in → deprecate → remove.
        - **Workflow Type Versioning** — For major rewrites. Create `workflowV2`.
        - **Worker Versioning** — For deployment-level control with Build IDs.

    3. **Not all changes require versioning** — Activity implementations, arguments, retry policies, and timer durations can change freely. Only changes that alter the Command sequence need versioning.

    4. **Observability in Temporal:**
        - `log` from `@temporalio/workflow` — Replay-aware logging
        - Search attributes — Filterable workflow metadata via `upsertSearchAttributes`
        - Prometheus metrics — Worker health monitoring
        - Event History — The ultimate debugging tool

    5. **Testing is essential:**
        - `TestWorkflowEnvironment.createLocal()` — Full test environment
        - Activity mocking — Test workflow logic in isolation
        - `Worker.runReplayHistory()` — Verify code changes are compatible with existing histories
        - `MockActivityEnvironment` — Test activities in isolation

    6. **Always replay test before deploying workflow changes** — Export production histories and replay them against new code to catch non-determinism errors.

---

## Next: Capstone Project

[:octicons-arrow-right-24: Capstone — Full Distributed E-Commerce System](../capstone/index.md)
