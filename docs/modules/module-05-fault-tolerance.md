# Module 5 — Fault Tolerance: Timeouts, Heartbeats & Cancellation

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 8 — The Trouble with Distributed Systems  
    **Temporal Concepts:** Timeout types, activity heartbeating, cancellation delivery, `CancellationScope`, resumable activities  
    **Time:** ~3 hours

---

## Part 1: Theory — The Trouble with Distributed Systems (DDIA Chapter 8)

Chapter 8 is one of the most important chapters in DDIA. Kleppmann systematically dismantles the assumptions we make about networked systems.

### Unreliable Networks

Most distributed systems use **asynchronous packet networks** — there is no guarantee that a message will arrive, or arrive within any particular time frame.

When you send a request and don't receive a response, you have **no way of knowing** what happened:

```
┌──────────┐                    ┌──────────┐
│  Client   │───── Request ────▶│  Server   │
│           │                    │           │
│           │     ??? No         │  Maybe:   │
│           │◀── Response ──     │  - Crashed │
│           │                    │  - Slow    │
│           │                    │  - Reply   │
│           │                    │    lost    │
└──────────┘                    └──────────┘
```

The request could have been lost. The server could have crashed. The response could have been lost. The server could be slow. **You cannot distinguish these cases.**

### Timeouts: The Only Tool

Since you can't tell the difference between a slow response and no response, the only practical approach is to use **timeouts**:

- If you don't receive a response within some time, you assume something is wrong
- But choosing the right timeout is hard:
    - **Too short** → False positives (declare nodes dead when they're just slow)
    - **Too long** → Slow detection of actual failures

There is no "correct" timeout value. It depends on your network characteristics, load patterns, and tolerance for false positives.

### Unreliable Clocks

Kleppmann explains that clocks in distributed systems are fundamentally unreliable:

- **Time-of-day clocks** — Can jump forward or backward (NTP synchronization, leap seconds)
- **Monotonic clocks** — Only useful for measuring elapsed time on a single machine, not for comparing across machines

This means you cannot reliably use timestamps to determine the order of events across different machines. This is why Temporal uses **logical ordering** (Event History sequence numbers) rather than wall-clock time.

### Process Pauses

A process can be paused at any time for an unpredictable duration:

- **Garbage collection** — Stop-the-world GC pauses can last seconds
- **Virtual machine suspension** — Live migration of VMs
- **Context switching** — OS scheduling under heavy load
- **Disk I/O** — Swapping to disk when memory is full

During a pause, the process doesn't know it's been paused. When it resumes, it may think only milliseconds have passed when minutes have elapsed.

### The Two Generals Problem

Two generals on opposite sides of a valley need to coordinate an attack. They can only communicate by sending messengers through the valley, but messengers may be captured. Neither general can ever be certain the other received their message.

This is analogous to network communication: **you can never be certain a message was received**. This fundamental impossibility constrains what distributed systems can achieve.

### Partial Failures

The defining characteristic of distributed systems: **some parts can fail while others continue working**. Unlike a single computer (which either works or doesn't), a distributed system can be in a state of **partial failure** — and these failures are often **nondeterministic**.

---

## Part 2: Mental Model — How Temporal Handles Unreliable Networks

### Temporal's Timeout Strategy

Temporal addresses the timeout problem from DDIA Chapter 8 with a layered timeout system. Instead of one timeout, Temporal provides **four distinct timeout types**, each serving a different purpose:

```
┌─────────────────────────────────────────────────────────────┐
│  Workflow Execution Timeout                                  │
│  (Total time for the entire workflow, including retries      │
│   and continue-as-new)                                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Workflow Run Timeout                                   │  │
│  │  (Time for a single run, before continue-as-new)        │  │
│  │                                                         │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │  Schedule-To-Close Timeout                        │   │  │
│  │  │  (Total time for activity, including all retries) │   │  │
│  │  │                                                   │   │  │
│  │  │  ┌────────────────────────────────────────────┐   │   │  │
│  │  │  │  Start-To-Close Timeout                     │   │   │  │
│  │  │  │  (Time for a single activity attempt)       │   │   │  │
│  │  │  │                                             │   │   │  │
│  │  │  │  ┌──────────────────────────────────────┐   │   │   │  │
│  │  │  │  │  Heartbeat Timeout                    │   │   │   │  │
│  │  │  │  │  (Time between heartbeat calls)       │   │   │   │  │
│  │  │  │  └──────────────────────────────────────┘   │   │   │  │
│  │  │  └────────────────────────────────────────────┘   │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Timeout | Scope | What It Limits | When to Set |
|---------|-------|---------------|-------------|
| **Workflow Execution** | Entire workflow | Total time including retries and continue-as-new | Rarely — only if you have a hard business deadline |
| **Workflow Run** | Single run | Time before continue-as-new | Rarely needed |
| **Schedule-To-Close** | Activity (all retries) | Total time from scheduling to final completion | When you have an overall deadline for an activity |
| **Start-To-Close** | Activity (single attempt) | Duration of one attempt | **Always set this** — it's required |
| **Heartbeat** | Between heartbeats | Time between `heartbeat()` calls | For long-running activities that heartbeat |

### Heartbeating: Detecting Stuck Activities

DDIA Chapter 8 explains that timeouts are the only way to detect failures. Temporal's **heartbeat mechanism** is a sophisticated implementation of this principle.

Without heartbeating, Temporal can only detect activity failure when the `startToCloseTimeout` expires. For a 30-minute activity, that means waiting 30 minutes to discover it's stuck.

With heartbeating, Temporal detects failure within the `heartbeatTimeout` period — typically seconds:

```
Without heartbeat:
  Activity starts → ... 30 minutes of silence ... → Timeout! Retry.

With heartbeat (heartbeatTimeout: 30s):
  Activity starts → heartbeat → heartbeat → heartbeat → ... silence ...
  30 seconds later → Timeout! Retry immediately.
```

### Cancellation Delivery via Heartbeat

A critical detail: **cancellation is delivered to activities through the heartbeat mechanism**. When a workflow cancels an activity:

1. The Temporal Service marks the activity for cancellation
2. The next time the activity calls `heartbeat()`, the SDK throws `CancelledFailure`
3. The activity can catch this and perform cleanup

**If an activity never heartbeats, it will never know it's been cancelled.** It will run to completion, wasting resources.

---

## Part 3: Activity Heartbeating in TypeScript

### Basic Heartbeating

```typescript
import { heartbeat } from '@temporalio/activity';

export async function processRecords(records: string[]): Promise<number> {
  let processed = 0;

  for (const record of records) {
    await processOneRecord(record);
    processed++;

    // Report progress — also enables cancellation detection
    heartbeat(processed);
  }

  return processed;
}
```

Configure the heartbeat timeout when creating the activity proxy:

```typescript
const { processRecords } = proxyActivities<typeof activities>({
  startToCloseTimeout: '30 minutes',
  heartbeatTimeout: '30 seconds',  // If no heartbeat for 30s, activity is timed out
});
```

### Resumable Activities with Heartbeat Details

Heartbeat details **persist across retries**. If a Worker crashes mid-activity, the next attempt can read the last heartbeat details and resume from where it left off:

```typescript
import { heartbeat, activityInfo, CancelledFailure } from '@temporalio/activity';

export async function processLargeDataset(datasetId: string): Promise<string> {
  const info = activityInfo();

  // Resume from last heartbeat if this is a retry
  const startIndex: number = info.heartbeatDetails ?? 0;

  const records = await fetchRecords(datasetId);

  try {
    for (let i = startIndex; i < records.length; i++) {
      await processRecord(records[i]);

      // Heartbeat with current progress
      // On retry, activityInfo().heartbeatDetails will be this value
      heartbeat(i + 1);
    }

    return `Processed ${records.length} records`;
  } catch (err) {
    if (err instanceof CancelledFailure) {
      // Activity was cancelled — perform cleanup
      await cleanup(datasetId);
    }
    throw err;
  }
}
```

The flow when a Worker crashes:

```
Attempt 1 (Worker A):
  Process record 0 → heartbeat(1)
  Process record 1 → heartbeat(2)
  Process record 2 → heartbeat(3)
  💥 Worker crashes!

Attempt 2 (Worker B):
  activityInfo().heartbeatDetails → 3
  startIndex = 3
  Process record 3 → heartbeat(4)   ← Resumes from where it left off!
  Process record 4 → heartbeat(5)
  ...
  Done!
```

### Handling Activity Cancellation

Activities must **opt in** to receive cancellation. There are two approaches:

#### Approach 1: Check via heartbeat

```typescript
import { heartbeat, CancelledFailure } from '@temporalio/activity';

export async function cancellableWork(items: string[]): Promise<string[]> {
  const results: string[] = [];

  try {
    for (const item of items) {
      const result = await processItem(item);
      results.push(result);
      heartbeat(results.length);  // CancelledFailure thrown here if cancelled
    }
  } catch (err) {
    if (err instanceof CancelledFailure) {
      // Return partial results on cancellation
      return results;
    }
    throw err;
  }

  return results;
}
```

#### Approach 2: Race against cancellation promise

```typescript
import { Context, CancelledFailure } from '@temporalio/activity';

export async function longRunningActivity(): Promise<void> {
  // Heartbeat in background so cancellation can be delivered
  let heartbeatEnabled = true;
  (async () => {
    while (heartbeatEnabled) {
      await Context.current().sleep(5000);
      Context.current().heartbeat();
    }
  })().catch(() => {});

  try {
    await Promise.race([
      Context.current().cancelled,  // Rejects with CancelledFailure
      doExpensiveWork(),
    ]);
  } catch (err) {
    if (err instanceof CancelledFailure) {
      await cleanup();
    }
    throw err;
  } finally {
    heartbeatEnabled = false;
  }
}
```

---

## Part 4: Workflow-Level Cancellation

### CancellationScope

Cancellation scopes control how cancellation propagates within a workflow:

```typescript
import { CancellationScope, sleep } from '@temporalio/workflow';

export async function scopedWorkflow(): Promise<void> {
  // Non-cancellable scope — runs even if workflow is cancelled
  await CancellationScope.nonCancellable(async () => {
    await criticalCleanup();
  });

  // Timeout scope — cancels inner work after timeout
  await CancellationScope.withTimeout('5 minutes', async () => {
    await longRunningActivity();
  });
}
```

### Cleanup on Cancellation with try/finally

```typescript
import { CancellationScope } from '@temporalio/workflow';

export async function workflowWithCleanup(): Promise<void> {
  await acquireResource();
  try {
    await doWork();
  } finally {
    // Cleanup runs even on cancellation
    await CancellationScope.nonCancellable(async () => {
      await releaseResource();
    });
  }
}
```

!!! warning "Without `CancellationScope.nonCancellable`, cleanup may not run"
    If a workflow is cancelled, any `await` in the `finally` block will also be cancelled. Wrapping cleanup in `CancellationScope.nonCancellable` ensures it completes.

---

## Part 5: Build It — File Processing Pipeline

Let's build a long-running file processing activity that demonstrates heartbeating, progress tracking, resumability, and cancellation.

### The Scenario

A workflow that processes a large CSV file:

1. Downloads the file (simulated)
2. Processes each row with heartbeating and progress tracking
3. Supports resuming from the last checkpoint after Worker crashes
4. Handles cancellation gracefully

### Activity Implementations

```typescript
// src/activities/file-processing.ts
import { heartbeat, activityInfo, CancelledFailure, log } from '@temporalio/activity';

export interface ProcessingProgress {
  rowsProcessed: number;
  totalRows: number;
  lastProcessedId: string;
}

export interface ProcessingResult {
  totalProcessed: number;
  errors: number;
  duration: string;
}

// Simulate a CSV file with rows
function generateRows(count: number): Array<{ id: string; data: string }> {
  return Array.from({ length: count }, (_, i) => ({
    id: `row-${i}`,
    data: `Data for row ${i}`,
  }));
}

export async function processFile(
  fileId: string,
  totalRows: number
): Promise<ProcessingResult> {
  const info = activityInfo();
  const startTime = Date.now();

  // Resume from last heartbeat if this is a retry
  const lastProgress: ProcessingProgress | undefined = info.heartbeatDetails;
  const startRow = lastProgress?.rowsProcessed ?? 0;

  if (startRow > 0) {
    log.info('Resuming from checkpoint', {
      fileId,
      startRow,
      totalRows,
    });
  } else {
    log.info('Starting file processing', { fileId, totalRows });
  }

  const rows = generateRows(totalRows);
  let errors = 0;

  try {
    for (let i = startRow; i < rows.length; i++) {
      // Simulate processing time (50ms per row)
      await new Promise((resolve) => setTimeout(resolve, 50));

      // Simulate occasional processing errors (non-fatal)
      if (Math.random() < 0.05) {
        errors++;
        log.warn('Error processing row, skipping', { rowId: rows[i].id });
        continue;
      }

      // Heartbeat with progress details
      // This enables: cancellation detection + resumability
      const progress: ProcessingProgress = {
        rowsProcessed: i + 1,
        totalRows,
        lastProcessedId: rows[i].id,
      };
      heartbeat(progress);

      // Log progress every 20%
      if ((i + 1) % Math.ceil(totalRows / 5) === 0) {
        log.info('Processing progress', {
          fileId,
          processed: i + 1,
          total: totalRows,
          percent: Math.round(((i + 1) / totalRows) * 100),
        });
      }
    }

    const duration = `${((Date.now() - startTime) / 1000).toFixed(1)}s`;
    log.info('File processing complete', { fileId, totalRows, errors, duration });

    return { totalProcessed: totalRows - errors, errors, duration };

  } catch (err) {
    if (err instanceof CancelledFailure) {
      log.info('Processing cancelled', {
        fileId,
        processedSoFar: lastProgress?.rowsProcessed ?? 0,
      });
    }
    throw err;
  }
}

export async function generateReport(
  fileId: string,
  result: ProcessingResult
): Promise<string> {
  log.info('Generating report', { fileId });

  const report = [
    `File Processing Report: ${fileId}`,
    `Total Processed: ${result.totalProcessed}`,
    `Errors: ${result.errors}`,
    `Duration: ${result.duration}`,
  ].join('\n');

  return report;
}

export async function sendNotification(
  userId: string,
  report: string
): Promise<void> {
  log.info('Sending notification', { userId });
  // Simulate sending email/notification
  log.info('Notification sent', { userId });
}
```

### Workflow Definition

```typescript
// src/workflows/file-processing.ts
import { proxyActivities, log, CancellationScope } from '@temporalio/workflow';
import type * as fileActivities from '../activities/file-processing';

// Long-running activity with heartbeat
const { processFile } = proxyActivities<typeof fileActivities>({
  startToCloseTimeout: '30 minutes',
  heartbeatTimeout: '30 seconds',  // Detect stuck activity within 30s
  retry: {
    maximumAttempts: 3,
  },
});

// Short activities with standard timeout
const { generateReport, sendNotification } = proxyActivities<typeof fileActivities>({
  startToCloseTimeout: '1 minute',
});

export interface FileProcessingInput {
  fileId: string;
  totalRows: number;
  userId: string;
}

export async function fileProcessingWorkflow(input: FileProcessingInput): Promise<string> {
  log.info('Starting file processing workflow', {
    fileId: input.fileId,
    totalRows: input.totalRows,
  });

  // Step 1: Process the file (long-running, heartbeated)
  const result = await processFile(input.fileId, input.totalRows);

  // Step 2: Generate report
  const report = await generateReport(input.fileId, result);

  // Step 3: Send notification (non-cancellable — always notify)
  await CancellationScope.nonCancellable(async () => {
    await sendNotification(input.userId, report);
  });

  log.info('File processing workflow complete', { fileId: input.fileId });
  return report;
}
```

Notice the **different timeout configurations**:

- `processFile` — Long `startToCloseTimeout` (30 min) with `heartbeatTimeout` (30s) for early failure detection
- `generateReport` / `sendNotification` — Short `startToCloseTimeout` (1 min), no heartbeat needed

### Worker and Client

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as fileActivities from './activities/file-processing';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/file-processing'),
    activities: fileActivities,
    taskQueue: 'file-processing-queue',
  });

  console.log('Worker started on task queue: file-processing-queue');
  await worker.run();
}

run().catch((err) => {
  console.error('Worker failed:', err);
  process.exit(1);
});
```

```typescript
// src/client.ts
import { Client } from '@temporalio/client';
import { fileProcessingWorkflow } from './workflows/file-processing';

async function run() {
  const client = new Client();

  console.log('Starting file processing workflow...');

  const handle = await client.workflow.start(fileProcessingWorkflow, {
    workflowId: 'file-process-001',
    taskQueue: 'file-processing-queue',
    args: [{
      fileId: 'dataset-2026-04.csv',
      totalRows: 100,
      userId: 'user-alice',
    }],
  });

  console.log(`Workflow started: ${handle.workflowId}`);
  console.log('Waiting for result...');

  const result = await handle.result();
  console.log('Report:\n', result);
}

run().catch(console.error);
```

### Run It

1. **Start the Temporal dev server:** `temporal server start-dev`
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`

Watch the worker logs — you'll see progress updates at 20%, 40%, 60%, 80%, 100%.

---

## Part 6: Experiments

### Experiment 1: Crash Recovery with Heartbeat Resumption

This is the most impressive demonstration:

1. Start the workflow with `totalRows: 500` (takes ~25 seconds)
2. Watch the progress logs
3. **Kill the worker** (Ctrl+C) at around 40% progress
4. Check the Web UI — workflow is still `RUNNING`
5. **Restart the worker**
6. Watch the logs — processing **resumes from the last heartbeat checkpoint**, not from the beginning!

The heartbeat details (`ProcessingProgress`) were persisted by Temporal. The new Worker reads them via `activityInfo().heartbeatDetails` and skips already-processed rows.

### Experiment 2: Cancel a Running Workflow

1. Start a workflow with `totalRows: 1000`
2. While it's processing, cancel it via CLI:
    ```bash
    temporal workflow cancel --workflow-id file-process-001
    ```
3. Watch the worker logs — the activity detects cancellation at the next `heartbeat()` call and throws `CancelledFailure`
4. The notification still sends (wrapped in `CancellationScope.nonCancellable`)

### Experiment 3: Heartbeat Timeout Detection

1. Modify `processFile` to add a long sleep simulating a stuck activity:
    ```typescript
    if (i === 10) {
      await new Promise(resolve => setTimeout(resolve, 60000)); // Stuck for 60s
    }
    ```
2. Run the workflow — after 30 seconds (the `heartbeatTimeout`), Temporal times out the activity and retries it
3. The retry resumes from the last heartbeat (row 10), not from the beginning

### Experiment 4: Compare With and Without Heartbeat

Run two workflows:

- One with `heartbeatTimeout: '30 seconds'` — detects stuck activity in 30s
- One without heartbeat timeout — must wait for the full `startToCloseTimeout` (30 minutes!)

This demonstrates why heartbeating is essential for long-running activities.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 8** teaches that networks are unreliable, clocks can't be trusted, and processes can pause unpredictably. Timeouts are the only practical tool for failure detection, but choosing the right timeout is hard.

    2. **Temporal's layered timeout system** addresses this with four timeout types at different scopes — from workflow-level down to heartbeat-level. Always set `startToCloseTimeout`. Use `heartbeatTimeout` for long-running activities.

    3. **Heartbeating serves three purposes:**
        - **Failure detection** — Detect stuck activities within seconds, not minutes
        - **Cancellation delivery** — Activities learn about cancellation via heartbeat
        - **Progress resumption** — Heartbeat details persist across retries, enabling activities to resume from checkpoints

    4. **Cancellation must be explicitly handled:**
        - Activities: heartbeat regularly and catch `CancelledFailure`
        - Workflows: use `CancellationScope.nonCancellable` for cleanup that must complete

    5. **The heartbeat details pattern** enables resumable activities — on retry after a crash, the activity reads `activityInfo().heartbeatDetails` and skips already-completed work

---

## Next Module

[:octicons-arrow-right-24: Module 6 — Distributed Coordination](module-06-coordination.md)
