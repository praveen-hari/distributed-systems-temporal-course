# Module 1 — Foundations of Distributed Systems

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 1 — Reliability, Scalability, Maintainability  
    **Temporal Concepts:** Core architecture, Workflows, Activities, Workers, Task Queues, Event History  
    **Time:** ~3 hours

---

## Part 1: Theory — The Three Pillars (DDIA Chapter 1)

Martin Kleppmann opens *Designing Data-Intensive Applications* by identifying three concerns that matter in most software systems:

### Reliability

> "The system should continue to work correctly even when things go wrong."

A reliable system:

- Performs the function the user expects
- Tolerates the user making mistakes or using the software in unexpected ways
- Performs well enough under expected load
- Prevents unauthorized access and abuse

Things that can go wrong are called **faults**, and systems that anticipate faults and cope with them are called **fault-tolerant**. Note that a fault is not the same as a failure:

| Term | Meaning |
|------|---------|
| **Fault** | One component of the system deviating from its spec |
| **Failure** | The system as a whole stops providing the required service |

The goal is to design systems where faults do not lead to failures.

#### Types of Faults

**Hardware faults** — Hard disks crash, RAM becomes faulty, power grids have blackouts. These are typically handled with redundancy: RAID for disks, dual power supplies, hot-swappable CPUs.

**Software faults** — Bugs that lie dormant until triggered by unusual circumstances. A software bug in one component can cause cascading failures. Examples include:

- A process that uses up shared resources (CPU, memory, disk, network bandwidth)
- A service that the system depends on slows down or returns corrupted responses
- Cascading failures where a small fault triggers faults in other components

**Human errors** — Humans are unreliable. Configuration errors by operators are a leading cause of outages. Approaches to mitigate:

- Design systems that minimize opportunities for error (well-designed APIs, admin interfaces)
- Provide sandbox environments where people can explore safely
- Test thoroughly at all levels: unit tests, integration tests, end-to-end tests
- Allow quick and easy recovery from human errors (fast rollback, gradual rollouts)
- Set up detailed monitoring (telemetry) so you can detect problems early

### Scalability

> "As the system grows, there should be reasonable ways of dealing with that growth."

Scalability is not a one-dimensional label — you can't say "X is scalable" or "Y doesn't scale." It's about asking specific questions:

- "If the system grows in a particular way, what are our options for coping with the growth?"
- "How can we add computing resources to handle the additional load?"

**Describing load** — Load can be described with *load parameters*: requests per second, ratio of reads to writes, number of simultaneously active users, hit rate on a cache, etc.

**Describing performance** — Once you've described load, you can investigate what happens when load increases:

- If you increase a load parameter and keep system resources unchanged, how is performance affected?
- If you increase a load parameter, how much do you need to increase resources to keep performance unchanged?

**Latency vs response time** — Response time is what the client sees (includes network delays and queueing). Latency is the duration a request is waiting to be handled. They are not the same.

!!! tip "Percentiles matter more than averages"
    The mean response time is not a good metric — it doesn't tell you how many users experience delays. Use **percentiles**: p50 (median), p95, p99, p999. Amazon describes response time requirements in terms of the 99.9th percentile because the customers with the slowest requests often have the most data — and they're often the most valuable customers.

### Maintainability

> "Over time, many different people will work on the system. They should all be able to work on it productively."

Three design principles for maintainability:

- **Operability** — Make it easy for operations teams to keep the system running smoothly
- **Simplicity** — Make it easy for new engineers to understand the system
- **Evolvability** — Make it easy for engineers to make changes to the system in the future

---

## Part 2: Mental Model — How Temporal Addresses These Pillars

Now let's connect DDIA's three pillars to Temporal's architecture.

### Temporal's Answer to Reliability

In traditional systems, if a process crashes mid-execution, the state is lost. You need to write extensive error handling, retry logic, and recovery code. This is exactly the problem Kleppmann describes — faults leading to failures.

Temporal solves this with **Durable Execution**:

> Once started, an application's main function executes to completion, whether that takes minutes, hours, days, weeks, or even years.
>
> — [Temporal Documentation](https://docs.temporal.io/evaluate/why-temporal)

If a crash occurs, Temporal guarantees your application picks up right where it left off. The platform tracks progress, not your code.

### Temporal's Answer to Scalability

Temporal separates **what to do** (your Workflow and Activity code) from **who does it** (Workers). You can run multiple Workers — dozens, hundreds, or thousands — to handle increased load. Workers poll Task Queues for work, and the Temporal Service distributes tasks across available Workers.

### Temporal's Answer to Maintainability

Temporal provides:

- **Operability** — A Web UI for monitoring workflows, a CLI for management, and full Event History for debugging
- **Simplicity** — Business logic is expressed as straightforward code (Workflows-as-Code), not complex state machines or message queue configurations
- **Evolvability** — Versioning APIs allow safe code changes to running workflows

---

## Part 3: Temporal's Core Architecture

Let's understand the building blocks of a Temporal application.

### The Big Picture

A Temporal application has two main parts:

1. **Your application** — Workflows, Activities, and Workers (code you write)
2. **The Temporal Service** — Orchestrates execution, persists state, manages Task Queues

```
┌─────────────────────────────────────────────────────────────────┐
│                     Temporal Service                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  Event History   │  │   Task Queues   │  │   Visibility   │  │
│  │  (Durable Log)   │  │  (Work Router)  │  │   (Search)     │  │
│  └─────────────────┘  └─────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Poll / Complete
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Worker                                   │
│  ┌─────────────────────────┐  ┌──────────────────────────────┐  │
│  │   Workflow Definitions  │  │   Activity Implementations   │  │
│  │   (Deterministic)       │  │   (Non-deterministic OK)     │  │
│  └─────────────────────────┘  └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

!!! important "The Temporal Service does NOT run your code"
    A common misconception is that the Temporal Service runs your code. In fact, **Workers run your code**. The Temporal Service only orchestrates what needs to be done and persists the Event History. Your data stays within your infrastructure.

### Workflows

A Workflow is your business logic defined in code — a sequence of steps outlining your process. In the TypeScript SDK, a Workflow is an async function exported from a workflow file:

```typescript
import { proxyActivities } from '@temporalio/workflow';
import type * as activities from '../activities/order';

const { reserveInventory, chargePayment, shipOrder } = proxyActivities<typeof activities>({
  startToCloseTimeout: '5 minutes',
});

export async function orderWorkflow(orderId: string): Promise<string> {
  await reserveInventory(orderId);
  await chargePayment(orderId);
  await shipOrder(orderId);
  return `Order ${orderId} completed`;
}
```

Key characteristics of Workflows:

- **Durable** — If the Worker crashes after `reserveInventory` completes, Temporal will resume from that point, not re-execute it
- **Deterministic** — Workflow code must produce the same sequence of commands on every execution (more on this below)
- **Long-running** — Workflows can run for minutes, hours, days, or even years

### Activities

Activities are the individual units of work in your Workflow. They handle interactions with the outside world — things that are prone to failure:

```typescript
// activities/order.ts
export async function reserveInventory(orderId: string): Promise<void> {
  // Call inventory service API
  const response = await fetch(`https://inventory.example.com/reserve/${orderId}`, {
    method: 'POST',
  });
  if (!response.ok) {
    throw new Error(`Failed to reserve inventory for order ${orderId}`);
  }
}

export async function chargePayment(orderId: string): Promise<void> {
  // Call payment gateway
  // ...
}

export async function shipOrder(orderId: string): Promise<void> {
  // Call shipping service
  // ...
}
```

Key characteristics of Activities:

- **Non-deterministic is OK** — Activities can make network calls, read files, generate random values, use the current time
- **Automatically retried** — If an Activity fails, Temporal retries it based on your configuration
- **Results are persisted** — Once an Activity completes, its result is stored in the Event History and never re-executed during replay

### Workers

Workers are long-running processes that poll Task Queues and execute your Workflow and Activity code:

```typescript
// worker.ts
import { Worker } from '@temporalio/worker';
import * as activities from './activities/order';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/order'),
    activities,
    taskQueue: 'order-queue',
  });

  console.log('Worker started, polling on task queue: order-queue');
  await worker.run();
}

run().catch((err) => {
  console.error('Worker failed:', err);
  process.exit(1);
});
```

Key characteristics of Workers:

- **You host them** — Workers run in your infrastructure, not on Temporal's servers
- **Scalable** — Run multiple Workers to handle more load
- **Stateless** — Workers don't store state; all state is in the Temporal Service's Event History

### Task Queues

Task Queues are named queues that connect Clients to Workers. When a Client starts a Workflow, it specifies a Task Queue. Workers poll specific Task Queues for work.

```typescript
// Client specifies the task queue
const result = await client.workflow.execute(orderWorkflow, {
  workflowId: 'order-123',
  taskQueue: 'order-queue',  // ← Must match the Worker's task queue
  args: ['order-123'],
});
```

Task Queues enable:

- **Routing** — Different Workers can handle different types of work
- **Load balancing** — Multiple Workers on the same queue share the load
- **Isolation** — Separate queues for separate concerns

### Event History

The Event History is a complete, durable log of everything that has happened in a Workflow Execution. Every time your Workflow code executes an Activity, starts a timer, or completes, the Temporal Service records it as an Event.

This is the foundation of Temporal's reliability. If a Worker crashes, a new Worker can replay the Event History to reconstruct the Workflow's state and resume from where it left off.

---

## Part 4: The Determinism Contract

This is the most important concept to understand in Temporal.

### Why Workflows Must Be Deterministic

Temporal achieves durability through **History Replay**. When a Worker needs to restore workflow state (after a crash, cache eviction, or to continue after a long timer), it **re-executes the workflow code from the beginning**. But instead of re-running Activities, it uses results stored in the Event History.

```
Initial Execution:
  Code runs → Generates Commands → Server stores as Events

Replay (Recovery):
  Code runs again → Generates Commands → SDK compares to Events
  If match: Use stored results, continue
  If mismatch: NondeterminismError!
```

### Commands and Events

Every workflow operation generates a **Command** that becomes an **Event**:

| Workflow Code | Command Generated | Event Stored |
|--------------|-------------------|--------------|
| Execute activity | `ScheduleActivityTask` | `ActivityTaskScheduled` |
| Sleep/timer | `StartTimer` | `TimerStarted` |
| Child workflow | `StartChildWorkflowExecution` | `ChildWorkflowExecutionStarted` |
| Complete workflow | `CompleteWorkflowExecution` | `WorkflowExecutionCompleted` |

### What Breaks Determinism

If the workflow code produces different Commands during replay than what's stored in the Event History, Temporal raises a `NondeterminismError` and the workflow becomes blocked.

**Example of non-deterministic code:**

```typescript
// ❌ BAD — Date.now() returns different values on replay
// (Note: TypeScript SDK actually replaces Date.now() with a
// deterministic version, but this illustrates the concept)
export async function badWorkflow(): Promise<string> {
  // In a language without sandbox protection, this would break:
  if (new Date().getHours() < 12) {
    await morningActivity();
  } else {
    await afternoonActivity();
  }
  return 'done';
}
```

### Sources of Non-Determinism

| Source | Why It's Dangerous | Solution |
|--------|-------------------|----------|
| Current time | Different on each execution | Use SDK-provided time (TypeScript auto-replaces `Date.now()`) |
| Random values | Different on each execution | Use SDK-provided random (TypeScript auto-replaces `Math.random()`) |
| Network calls / I/O | Results may differ | Move to Activities |
| File system access | Files may change | Move to Activities |
| External state | State may change | Move to Activities |

### The TypeScript SDK's V8 Sandbox

The TypeScript SDK provides automatic protection by running workflows in an **isolated V8 sandbox**:

- `Math.random()` → replaced with a deterministic seeded PRNG
- `Date.now()` → replaced with the workflow task timestamp (advances only on durable operations like `sleep()`)
- `setTimeout` → replaced with a deterministic timer
- Node.js modules (`fs`, `http`, etc.) → **blocked** from import

This means many common sources of non-determinism are handled automatically. However, it's still your responsibility to ensure workflow code doesn't contain other sources of non-determinism.

!!! tip "The Golden Rule"
    **Workflows orchestrate. Activities execute.** Keep all I/O, network calls, and side effects in Activities. Workflows should only contain orchestration logic — deciding what to do, in what order, and how to handle results.

---

## Part 5: Build It — Your First Real Workflow

Let's build a workflow that demonstrates Temporal's core value: **surviving failures**.

### The Scenario

We'll build a simple user signup workflow that:

1. Validates the user's email
2. Creates the user account
3. Sends a welcome email

In a traditional system, if step 2 succeeds but step 3 fails, you'd have a user with no welcome email and no easy way to retry just step 3.

### Project Structure

```
src/
├── activities/
│   └── signup.ts       # Activity implementations
├── workflows/
│   └── signup.ts       # Workflow definition
├── worker.ts           # Worker setup
└── client.ts           # Client to start the workflow
```

### Step 1: Define the Activities

```typescript
// src/activities/signup.ts

export interface UserInput {
  email: string;
  name: string;
}

export async function validateEmail(email: string): Promise<boolean> {
  console.log(`Validating email: ${email}`);
  // Simulate validation — in production, this might call an email verification API
  const isValid = email.includes('@') && email.includes('.');
  if (!isValid) {
    throw new Error(`Invalid email: ${email}`);
  }
  return true;
}

export async function createAccount(user: UserInput): Promise<string> {
  console.log(`Creating account for: ${user.email}`);
  // Simulate account creation — in production, this writes to a database
  const userId = `user-${Date.now()}`;
  console.log(`Account created with ID: ${userId}`);
  return userId;
}

export async function sendWelcomeEmail(email: string, userId: string): Promise<void> {
  console.log(`Sending welcome email to: ${email} (user: ${userId})`);
  // Simulate sending email — in production, this calls an email service
  // Simulate occasional failure to demonstrate retries
  if (Math.random() < 0.3) {
    throw new Error('Email service temporarily unavailable');
  }
  console.log(`Welcome email sent to: ${email}`);
}
```

!!! note "Activities can use non-deterministic code"
    Notice that `createAccount` uses `Date.now()` and `sendWelcomeEmail` uses `Math.random()`. This is perfectly fine in Activities — they run outside the workflow sandbox and their results are persisted in the Event History.

### Step 2: Define the Workflow

```typescript
// src/workflows/signup.ts
import { proxyActivities, log } from '@temporalio/workflow';
import type * as activities from '../activities/signup';

const { validateEmail, createAccount, sendWelcomeEmail } = proxyActivities<typeof activities>({
  startToCloseTimeout: '1 minute',
  retry: {
    maximumAttempts: 3,
  },
});

export async function signupWorkflow(email: string, name: string): Promise<string> {
  log.info('Starting signup workflow', { email, name });

  // Step 1: Validate email
  await validateEmail(email);
  log.info('Email validated', { email });

  // Step 2: Create account
  const userId = await createAccount({ email, name });
  log.info('Account created', { userId });

  // Step 3: Send welcome email
  await sendWelcomeEmail(email, userId);
  log.info('Welcome email sent', { email });

  return userId;
}
```

Key things to notice:

- **`import type * as activities`** — Type-only import, critical for the V8 sandbox
- **`proxyActivities`** — Creates proxies that schedule Activities on the Temporal Service instead of calling them directly
- **`startToCloseTimeout`** — Maximum time for a single Activity attempt
- **`retry.maximumAttempts`** — How many times to retry a failed Activity
- **`log` from `@temporalio/workflow`** — Replay-aware logging that suppresses duplicate messages during replay

### Step 3: Set Up the Worker

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as activities from './activities/signup';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/signup'),
    activities,
    taskQueue: 'signup-queue',
  });

  console.log('Worker started, polling on task queue: signup-queue');
  await worker.run();
}

run().catch((err) => {
  console.error('Worker failed:', err);
  process.exit(1);
});
```

### Step 4: Start the Workflow

```typescript
// src/client.ts
import { Client } from '@temporalio/client';
import { signupWorkflow } from './workflows/signup';

async function run() {
  const client = new Client();

  console.log('Starting signup workflow...');

  const result = await client.workflow.execute(signupWorkflow, {
    workflowId: 'signup-user-1',
    taskQueue: 'signup-queue',
    args: ['alice@example.com', 'Alice'],
  });

  console.log(`Signup completed! User ID: ${result}`);
}

run().catch((err) => {
  console.error('Client failed:', err);
  process.exit(1);
});
```

### Step 5: Run It

1. **Ensure the Temporal dev server is running:**
    ```bash
    temporal server start-dev
    ```

2. **Start the worker** (in a new terminal):
    ```bash
    npx ts-node src/worker.ts
    ```

3. **Execute the workflow** (in another terminal):
    ```bash
    npx ts-node src/client.ts
    ```

4. **Open the Web UI** at [http://localhost:8233](http://localhost:8233) and click on the `signup-user-1` workflow to see its Event History.

---

## Part 6: Experiment — Break Things

The best way to understand Temporal's reliability is to **intentionally break things**.

### Experiment 1: Kill the Worker Mid-Execution

1. Start the worker
2. Start a workflow
3. **Kill the worker** (Ctrl+C) while the workflow is running
4. Check the Web UI — the workflow is still `RUNNING`
5. **Restart the worker** — the workflow resumes from where it left off

This demonstrates **durable execution**: the Temporal Service persisted the Event History, and the new Worker replayed it to restore state.

### Experiment 2: Watch Activity Retries

The `sendWelcomeEmail` activity has a 30% chance of failure. Run the workflow multiple times and watch the Web UI:

- Click on a workflow execution
- Expand the Event History
- Look for `ActivityTaskFailed` events followed by `ActivityTaskScheduled` events — these are automatic retries

### Experiment 3: Inspect the Event History via CLI

```bash
temporal workflow show --workflow-id signup-user-1
```

You'll see every Event in the workflow's history:

```
  1  WorkflowExecutionStarted
  2  WorkflowTaskScheduled
  3  WorkflowTaskStarted
  4  WorkflowTaskCompleted
  5  ActivityTaskScheduled       # validateEmail scheduled
  6  ActivityTaskStarted
  7  ActivityTaskCompleted       # validateEmail completed
  8  WorkflowTaskScheduled
  9  WorkflowTaskStarted
  10 WorkflowTaskCompleted
  11 ActivityTaskScheduled       # createAccount scheduled
  ...
```

Each Event is a durable record. This is the log that enables replay and recovery.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 1** teaches that reliable systems must handle hardware faults, software faults, and human errors — the goal is preventing faults from becoming failures
    
    2. **Temporal's Durable Execution** ensures that once a workflow starts, it runs to completion even if Workers crash — the Temporal Service persists every step in the Event History
    
    3. **The four building blocks** of a Temporal application:
        - **Workflows** — Deterministic orchestration logic
        - **Activities** — Non-deterministic units of work (I/O, API calls)
        - **Workers** — Processes that execute your code
        - **Task Queues** — Named queues connecting Clients to Workers
    
    4. **The Determinism Contract** — Workflows must produce the same Commands on every execution because of History Replay. The TypeScript SDK's V8 sandbox provides automatic protection for common cases (`Math.random()`, `Date.now()`, `setTimeout`)
    
    5. **The Golden Rule** — Workflows orchestrate, Activities execute. All I/O belongs in Activities.

---

## Next Module

[:octicons-arrow-right-24: Module 2 — Data Models & Activity Design](module-02-data-models.md)
