# Module 6 — Distributed Coordination: Signals, Queries & Updates

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 9 — Consistency and Consensus  
    **Temporal Concepts:** Signals, Queries, Updates, `condition()`, `Trigger`, the Entity Workflow pattern  
    **Time:** ~3 hours

---

## Part 1: Theory — Consistency and Consensus (DDIA Chapter 9)

Chapter 9 is the culmination of DDIA's exploration of distributed systems. It tackles the hardest questions: how do distributed nodes agree on anything?

### Linearizability

**Linearizability** (also called atomic consistency) is the strongest consistency guarantee. It means: the system behaves as if there is only one copy of the data, and all operations are atomic.

In a linearizable system:

- Once a write completes, all subsequent reads see that write
- There is a single, real-time ordering of all operations
- No stale reads, no split-brain

Linearizability is expensive — it requires coordination between nodes, which adds latency and reduces availability.

### The CAP Theorem

Kleppmann explains the CAP theorem (and why it's often misunderstood):

> In a network partition, you must choose between **Consistency** and **Availability**.

| Choice | Behavior During Partition | Example |
|--------|--------------------------|---------|
| **CP** (Consistency + Partition tolerance) | Reject requests to maintain consistency | HBase, ZooKeeper |
| **AP** (Availability + Partition tolerance) | Accept requests, risk inconsistency | Cassandra, DynamoDB |

!!! note "CAP is about partitions, not normal operation"
    The CAP theorem only applies during network partitions. During normal operation, you can have both consistency and availability. The real question is: what happens when things go wrong?

### Consensus

**Consensus** means getting several nodes to agree on something. It's the foundation of:

- **Leader election** — Which node is the leader?
- **Atomic commit** — Did all nodes commit the transaction?
- **Total order broadcast** — What order did events happen in?

Consensus algorithms (Raft, Paxos, Zab) solve these problems, but they're complex and have performance costs.

### Ordering Guarantees

DDIA discusses several ordering guarantees:

| Guarantee | Meaning |
|-----------|---------|
| **Total order** | All nodes see events in the same order |
| **Causal order** | Events that are causally related are seen in the correct order |
| **No ordering** | Events may be seen in any order |

Total order is the strongest but most expensive. Causal order is often sufficient and cheaper to implement.

### Coordination Services

Systems like ZooKeeper, etcd, and Consul provide coordination primitives:

- **Distributed locks** — Ensure only one process accesses a resource
- **Leader election** — Choose a single leader from a group
- **Configuration management** — Distribute configuration to all nodes
- **Service discovery** — Find which nodes provide a service

These are hard to build correctly. Kleppmann warns that even with these tools, distributed coordination is fraught with subtle bugs.

---

## Part 2: Mental Model — Workflows as Coordination Primitives

### The Insight

Temporal workflows are, in essence, **durable coordination primitives**. A running workflow is:

- A **stateful entity** that maintains consistent state
- A **coordinator** that orchestrates activities across services
- A **consensus point** — the Event History provides a total ordering of events

Temporal's message-passing features (Signals, Queries, Updates) turn workflows into interactive coordination points that external systems can communicate with.

### Signals, Queries, and Updates

| Feature | Direction | Modifies State? | Returns Result? | Blocks? | Durable? |
|---------|-----------|-----------------|-----------------|---------|----------|
| **Signal** | External → Workflow | ✅ Yes | ❌ No | ❌ No (fire-and-forget) | ✅ Yes (in history) |
| **Query** | External → Workflow | ❌ No | ✅ Yes | ❌ No (immediate) | ❌ No (not in history) |
| **Update** | External → Workflow | ✅ Yes | ✅ Yes | ✅ Yes (waits for handler) | ✅ Yes (in history) |

The mental model:

- **Query** to **peek** — Read workflow state without changing it
- **Signal** to **push** — Send data to a workflow (fire-and-forget)
- **Update** to **pop** — Modify state and get a confirmed result back

```
                    ┌─────────────────────────┐
                    │     Running Workflow      │
                    │                           │
  Signal ──push──▶  │  ┌─────────────────────┐ │
  (fire & forget)   │  │   Workflow State     │ │
                    │  │                       │ │
  Query ──peek──▶   │  │  items: [...]        │ │  ──▶ returns state
  (read-only)       │  │  status: "pending"   │ │
                    │  │  approved: false      │ │
  Update ──pop──▶   │  │                       │ │  ──▶ returns result
  (modify + result) │  └─────────────────────┘ │
                    └─────────────────────────┘
```

### When to Use Each

| Scenario | Use |
|----------|-----|
| Human approval (approve/reject) | **Signal** |
| Adding items to a workflow's queue | **Signal** |
| Dashboard showing workflow progress | **Query** |
| Health check / monitoring | **Query** |
| Add item and get back the new count | **Update** |
| Validated state change with confirmation | **Update** (with validator) |

---

## Part 3: Signals — Fire-and-Forget Communication

### Defining and Handling Signals

```typescript
import { defineSignal, setHandler, condition } from '@temporalio/workflow';

// Define signals with their argument types
export const approveSignal = defineSignal<[boolean]>('approve');
export const addItemSignal = defineSignal<[string]>('addItem');

export async function orderWorkflow(): Promise<string> {
  let approved = false;
  const items: string[] = [];

  // Register signal handlers — they mutate workflow state
  setHandler(approveSignal, (value) => {
    approved = value;
  });

  setHandler(addItemSignal, (item) => {
    items.push(item);
  });

  // Wait until approved
  await condition(() => approved);

  return `Processed ${items.length} items`;
}
```

### Sending Signals from a Client

```typescript
import { Client } from '@temporalio/client';
import { approveSignal, addItemSignal } from './workflows/order';

const client = new Client();
const handle = client.workflow.getHandle('order-123');

// Send signals (fire-and-forget — no response)
await handle.signal(addItemSignal, 'Widget A');
await handle.signal(addItemSignal, 'Widget B');
await handle.signal(approveSignal, true);
```

### Sending Signals via CLI

```bash
temporal workflow signal \
  --workflow-id order-123 \
  --name addItem \
  --input '"Widget C"'

temporal workflow signal \
  --workflow-id order-123 \
  --name approve \
  --input 'true'
```

### The `condition()` Function

`condition()` is how workflows **wait** for state changes. It blocks until the predicate returns `true`:

```typescript
// Wait indefinitely until approved
await condition(() => approved);

// Wait with a timeout — returns true if condition met, false if timed out
const gotApproval = await condition(() => approved, '24 hours');

if (gotApproval) {
  return 'approved';
} else {
  return 'auto-rejected due to timeout';
}
```

`condition()` is efficient — it doesn't poll. The SDK re-evaluates the predicate whenever the workflow state changes (e.g., after a signal handler runs).

---

## Part 4: Queries — Read-Only State Inspection

### Defining and Handling Queries

```typescript
import { defineQuery, setHandler } from '@temporalio/workflow';

export const getStatusQuery = defineQuery<string>('getStatus');
export const getItemsQuery = defineQuery<string[]>('getItems');
export const getProgressQuery = defineQuery<number>('getProgress');

export async function orderWorkflow(): Promise<string> {
  let status = 'pending';
  const items: string[] = [];

  setHandler(getStatusQuery, () => status);
  setHandler(getItemsQuery, () => [...items]); // Return a copy
  setHandler(getProgressQuery, () => items.length);

  // ... workflow logic that updates status and items ...

  status = 'completed';
  return 'done';
}
```

!!! danger "Queries must NEVER modify state"
    Query handlers are **read-only**. They must not modify workflow state, call activities, start timers, or have any side effects. Violating this causes non-determinism errors on replay.

    ```typescript
    // ❌ BAD — Modifies state in a query handler
    setHandler(getStatusQuery, () => {
      queryCount++;  // NEVER do this!
      return status;
    });

    // ✅ GOOD — Read-only
    setHandler(getStatusQuery, () => status);
    ```

### Querying from a Client

```typescript
const handle = client.workflow.getHandle('order-123');

const status = await handle.query(getStatusQuery);
console.log('Status:', status);  // "pending"

const items = await handle.query(getItemsQuery);
console.log('Items:', items);    // ["Widget A", "Widget B"]
```

### Querying via CLI

```bash
temporal workflow query \
  --workflow-id order-123 \
  --name getStatus
```

---

## Part 5: Updates — Synchronous State Modification

Updates combine the best of signals and queries: they modify state **and** return a result, synchronously.

### Defining Updates with Validators

```typescript
import { defineUpdate, setHandler } from '@temporalio/workflow';

// Define update: return type = number, argument type = [string]
export const addItemUpdate = defineUpdate<number, [string]>('addItem');

export async function orderWorkflow(): Promise<string> {
  const items: string[] = [];
  let completed = false;

  setHandler(
    addItemUpdate,
    // Handler: modifies state and returns result
    (item: string) => {
      items.push(item);
      return items.length;  // Return new count to caller
    },
    {
      // Validator: rejects invalid input BEFORE it enters history
      validator: (item: string) => {
        if (!item || item.trim().length === 0) {
          throw new Error('Item name cannot be empty');
        }
        if (items.length >= 10) {
          throw new Error('Order is full (max 10 items)');
        }
      },
    }
  );

  await condition(() => completed);
  return `Order with ${items.length} items`;
}
```

!!! warning "Validator rules"
    Validators must NOT:

    - Mutate workflow state
    - Call activities, sleep, or any blocking operation
    - Have side effects

    They are read-only checks. Throw an error to reject; return normally to accept.

### Sending Updates from a Client

```typescript
const handle = client.workflow.getHandle('order-123');

// Update returns a result synchronously
const count = await handle.executeUpdate(addItemUpdate, { args: ['Widget A'] });
console.log('Item count:', count);  // 1

const count2 = await handle.executeUpdate(addItemUpdate, { args: ['Widget B'] });
console.log('Item count:', count2);  // 2

// Validator rejects invalid input
try {
  await handle.executeUpdate(addItemUpdate, { args: [''] });
} catch (err) {
  console.log('Rejected:', err.message);  // "Item name cannot be empty"
}
```

### Sending Updates via CLI

```bash
temporal workflow update execute \
  --workflow-id order-123 \
  --name addItem \
  --input '"Widget C"'
```

---

## Part 6: Build It — Order Management System

Let's build a complete order management workflow that uses all three message-passing features.

### Shared Types

```typescript
// src/shared/types.ts
export interface OrderItem {
  name: string;
  price: number;
  quantity: number;
}

export interface OrderState {
  orderId: string;
  items: OrderItem[];
  status: 'draft' | 'submitted' | 'approved' | 'rejected' | 'fulfilled';
  total: number;
  submittedAt?: string;
  approvedBy?: string;
}
```

### Workflow Definition

```typescript
// src/workflows/order-management.ts
import {
  defineSignal, defineQuery, defineUpdate,
  setHandler, condition, log,
  proxyActivities,
} from '@temporalio/workflow';
import type * as orderActivities from '../activities/order';
import type { OrderItem, OrderState } from '../shared/types';

// --- Signal Definitions ---
export const submitOrderSignal = defineSignal('submitOrder');
export const approveOrderSignal = defineSignal<[string]>('approveOrder');  // approver name
export const rejectOrderSignal = defineSignal<[string]>('rejectOrder');    // reason

// --- Query Definitions ---
export const getOrderStateQuery = defineQuery<OrderState>('getOrderState');
export const getItemCountQuery = defineQuery<number>('getItemCount');

// --- Update Definitions ---
export const addItemUpdate = defineUpdate<OrderState, [OrderItem]>('addItem');
export const removeItemUpdate = defineUpdate<OrderState, [string]>('removeItem');  // item name

// --- Activity Proxies ---
const { fulfillOrder, sendOrderNotification } = proxyActivities<typeof orderActivities>({
  startToCloseTimeout: '1 minute',
});

export async function orderManagementWorkflow(orderId: string): Promise<OrderState> {
  // --- Workflow State ---
  const state: OrderState = {
    orderId,
    items: [],
    status: 'draft',
    total: 0,
  };

  function recalculateTotal() {
    state.total = state.items.reduce(
      (sum, item) => sum + item.price * item.quantity, 0
    );
  }

  // --- Query Handlers (read-only) ---
  setHandler(getOrderStateQuery, () => ({ ...state, items: [...state.items] }));
  setHandler(getItemCountQuery, () => state.items.length);

  // --- Update Handlers (modify + return) ---
  setHandler(
    addItemUpdate,
    (item: OrderItem) => {
      state.items.push(item);
      recalculateTotal();
      log.info('Item added', { orderId, item: item.name, total: state.total });
      return { ...state, items: [...state.items] };
    },
    {
      validator: (item: OrderItem) => {
        if (state.status !== 'draft') {
          throw new Error(`Cannot add items: order is ${state.status}`);
        }
        if (!item.name) throw new Error('Item name is required');
        if (item.price <= 0) throw new Error('Price must be positive');
        if (item.quantity <= 0) throw new Error('Quantity must be positive');
      },
    }
  );

  setHandler(
    removeItemUpdate,
    (itemName: string) => {
      state.items = state.items.filter((i) => i.name !== itemName);
      recalculateTotal();
      log.info('Item removed', { orderId, item: itemName, total: state.total });
      return { ...state, items: [...state.items] };
    },
    {
      validator: (itemName: string) => {
        if (state.status !== 'draft') {
          throw new Error(`Cannot remove items: order is ${state.status}`);
        }
        if (!state.items.find((i) => i.name === itemName)) {
          throw new Error(`Item "${itemName}" not found`);
        }
      },
    }
  );

  // --- Signal Handlers ---
  setHandler(submitOrderSignal, () => {
    if (state.status === 'draft' && state.items.length > 0) {
      state.status = 'submitted';
      state.submittedAt = new Date().toISOString();
      log.info('Order submitted', { orderId, total: state.total });
    }
  });

  setHandler(approveOrderSignal, (approver: string) => {
    if (state.status === 'submitted') {
      state.status = 'approved';
      state.approvedBy = approver;
      log.info('Order approved', { orderId, approver });
    }
  });

  setHandler(rejectOrderSignal, (reason: string) => {
    if (state.status === 'submitted') {
      state.status = 'rejected';
      log.info('Order rejected', { orderId, reason });
    }
  });

  // --- Main Workflow Logic ---

  // Phase 1: Wait for order to be submitted
  log.info('Order created, waiting for items and submission', { orderId });
  await condition(() => state.status === 'submitted');

  // Phase 2: Wait for approval or rejection (with 48-hour timeout)
  log.info('Order submitted, waiting for approval', { orderId });
  const gotDecision = await condition(
    () => state.status === 'approved' || state.status === 'rejected',
    '48 hours'
  );

  if (!gotDecision) {
    state.status = 'rejected';
    log.info('Order auto-rejected due to timeout', { orderId });
  }

  // Phase 3: Fulfill if approved
  if (state.status === 'approved') {
    await fulfillOrder(orderId, state.items);
    state.status = 'fulfilled';
    log.info('Order fulfilled', { orderId });
  }

  // Notify regardless of outcome
  await sendOrderNotification(orderId, state.status);

  return state;
}
```

### Activity Implementations

```typescript
// src/activities/order.ts
import { log } from '@temporalio/activity';
import type { OrderItem } from '../shared/types';

export async function fulfillOrder(orderId: string, items: OrderItem[]): Promise<void> {
  log.info('Fulfilling order', {
    orderId,
    itemCount: items.length,
    total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
  });
  // Simulate fulfillment processing
  log.info('Order fulfilled successfully', { orderId });
}

export async function sendOrderNotification(
  orderId: string,
  status: string
): Promise<void> {
  log.info('Sending order notification', { orderId, status });
  // Simulate notification
  log.info('Notification sent', { orderId, status });
}
```

### Worker

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as orderActivities from './activities/order';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/order-management'),
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

### Interactive Client

```typescript
// src/client.ts
import { Client } from '@temporalio/client';
import {
  orderManagementWorkflow,
  addItemUpdate, removeItemUpdate,
  submitOrderSignal, approveOrderSignal,
  getOrderStateQuery, getItemCountQuery,
} from './workflows/order-management';

async function run() {
  const client = new Client();
  const orderId = 'order-001';

  // Start the workflow
  console.log('Starting order workflow...');
  const handle = await client.workflow.start(orderManagementWorkflow, {
    workflowId: orderId,
    taskQueue: 'order-queue',
    args: [orderId],
  });

  // Add items via Updates (get confirmation back)
  console.log('\n--- Adding items via Updates ---');
  let state = await handle.executeUpdate(addItemUpdate, {
    args: [{ name: 'Laptop', price: 999.99, quantity: 1 }],
  });
  console.log(`Added Laptop. Items: ${state.items.length}, Total: $${state.total}`);

  state = await handle.executeUpdate(addItemUpdate, {
    args: [{ name: 'Mouse', price: 29.99, quantity: 2 }],
  });
  console.log(`Added Mouse. Items: ${state.items.length}, Total: $${state.total}`);

  // Query current state
  console.log('\n--- Querying state ---');
  const currentState = await handle.query(getOrderStateQuery);
  console.log('Order status:', currentState.status);
  console.log('Item count:', await handle.query(getItemCountQuery));

  // Try adding invalid item (validator rejects)
  console.log('\n--- Testing validator ---');
  try {
    await handle.executeUpdate(addItemUpdate, {
      args: [{ name: '', price: 0, quantity: 0 }],
    });
  } catch (err) {
    console.log('Validator rejected:', (err as Error).message);
  }

  // Submit order via Signal
  console.log('\n--- Submitting order ---');
  await handle.signal(submitOrderSignal);

  // Try adding item after submission (validator rejects)
  try {
    await handle.executeUpdate(addItemUpdate, {
      args: [{ name: 'Keyboard', price: 79.99, quantity: 1 }],
    });
  } catch (err) {
    console.log('Cannot add after submit:', (err as Error).message);
  }

  // Approve order via Signal
  console.log('\n--- Approving order ---');
  await handle.signal(approveOrderSignal, 'Manager Alice');

  // Wait for workflow to complete
  const finalState = await handle.result();
  console.log('\n--- Final State ---');
  console.log(JSON.stringify(finalState, null, 2));
}

run().catch(console.error);
```

### Run It

1. **Start the Temporal dev server:** `temporal server start-dev`
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`

You'll see the full lifecycle: items added (with confirmation), state queried, order submitted, approved, and fulfilled.

---

## Part 7: Experiments

### Experiment 1: Interactive via CLI

Start the workflow from the client, but interact with it via CLI:

```bash
# Query the current state
temporal workflow query --workflow-id order-001 --name getOrderState

# Add an item via Update
temporal workflow update execute \
  --workflow-id order-001 \
  --name addItem \
  --input '{"name": "Monitor", "price": 499.99, "quantity": 1}'

# Submit the order
temporal workflow signal --workflow-id order-001 --name submitOrder

# Approve the order
temporal workflow signal \
  --workflow-id order-001 \
  --name approveOrder \
  --input '"Manager Bob"'
```

### Experiment 2: Rejection and Timeout

1. Start a workflow, add items, submit it
2. Send a reject signal instead of approve:
    ```bash
    temporal workflow signal \
      --workflow-id order-001 \
      --name rejectOrder \
      --input '"Budget exceeded"'
    ```
3. Or don't send any signal — the 48-hour timeout will auto-reject (use time-skipping in tests to verify)

### Experiment 3: Observe Event History

Check the Web UI for `order-001`. You'll see:

- `WorkflowExecutionUpdateAccepted` events for each Update
- `WorkflowExecutionSignaled` events for signals
- No events for queries (they're not persisted)

This demonstrates the durability difference: signals and updates are in the history, queries are not.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 9** teaches that distributed coordination is fundamentally hard — consensus, linearizability, and ordering guarantees all have costs. Coordination services (ZooKeeper, etcd) provide primitives but are complex to use correctly.

    2. **Temporal workflows are durable coordination primitives** — they maintain consistent state, provide total ordering via the Event History, and support external communication through Signals, Queries, and Updates.

    3. **Three message-passing features:**
        - **Signal** — Fire-and-forget state mutation (durable, in history)
        - **Query** — Read-only state inspection (not in history)
        - **Update** — Synchronous state mutation with result and optional validator (durable, in history)

    4. **`condition()`** is how workflows wait for state changes — it blocks until a predicate is true, with optional timeout. It's efficient (no polling) and durable.

    5. **Validators on Updates** reject invalid input **before** it enters the Event History. They must be read-only and non-blocking.

    6. **The Entity Workflow pattern** — A long-running workflow that maintains state and responds to signals/queries/updates is a powerful pattern for modeling stateful entities (orders, users, sessions).

---

## Next Module

[:octicons-arrow-right-24: Module 7 — Batch & Stream Processing](module-07-batch-stream.md)
