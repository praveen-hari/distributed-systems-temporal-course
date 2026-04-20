# Module 7 — Batch & Stream Processing

!!! abstract "Module Overview"
    **DDIA Reference:** Chapters 10–11 — Batch Processing & Stream Processing  
    **Temporal Concepts:** Child Workflows, Continue-As-New, Schedules, fan-out/fan-in patterns  
    **Time:** ~3 hours

---

## Part 1: Theory — Batch & Stream Processing (DDIA Chapters 10–11)

### Batch Processing (Chapter 10)

Kleppmann traces the history of batch processing from Unix pipes to MapReduce:

**Unix philosophy:** Small tools that do one thing well, connected by pipes.

```
cat /var/log/access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -5
```

This pipeline processes data in stages, each stage transforming the output of the previous one. The key insight: **the output of one stage is the input of the next**.

**MapReduce** applies this principle at scale:

```
Input Data
    │
    ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│  Map 1   │   │  Map 2   │   │  Map 3   │   ← Fan-out (parallel)
└────┬────┘   └────┬────┘   └────┬────┘
     │              │              │
     └──────────────┼──────────────┘
                    ▼
              ┌──────────┐
              │  Shuffle  │   ← Group by key
              └────┬─────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Reduce 1 │  │ Reduce 2 │  │ Reduce 3 │  ← Fan-in (aggregate)
└─────────┘  └─────────┘  └─────────┘
                    │
                    ▼
              Output Data
```

Key characteristics of batch processing:

- **Bounded input** — The dataset has a known, finite size
- **Derived data** — Output is computed from input; can be recomputed if needed
- **Fault tolerance** — If a task fails, it can be retried (input is immutable)

### Stream Processing (Chapter 11)

Stream processing extends batch processing to **unbounded data** — data that arrives continuously with no defined end.

Key concepts:

- **Event** — An immutable record of something that happened at a particular time
- **Producer** — Generates events
- **Consumer** — Processes events
- **Topic/Stream** — A named channel of events

**Event time vs processing time:**

| Concept | Meaning |
|---------|---------|
| **Event time** | When the event actually occurred |
| **Processing time** | When the system processes the event |

These can differ significantly due to network delays, buffering, and processing backlogs.

**Windowing** — Grouping events by time for aggregation:

| Window Type | Description |
|-------------|-------------|
| **Tumbling** | Fixed-size, non-overlapping (e.g., every 5 minutes) |
| **Hopping** | Fixed-size, overlapping (e.g., 5-minute window every 1 minute) |
| **Sliding** | Events within a time range of each other |
| **Session** | Events grouped by activity, with gaps between sessions |

### The Challenge: Exactly-Once Processing

Both batch and stream processing face the challenge of **exactly-once semantics**:

- **At-most-once** — Process each event 0 or 1 times (may lose events)
- **At-least-once** — Process each event 1 or more times (may duplicate)
- **Exactly-once** — Process each event exactly 1 time (hardest to achieve)

Achieving exactly-once requires either idempotent operations or transactional processing.

---

## Part 2: Mental Model — Temporal's Processing Patterns

### Child Workflows = MapReduce Tasks

Temporal's **Child Workflows** map directly to the fan-out/fan-in pattern from MapReduce:

```
Parent Workflow (Coordinator)
    │
    ├──▶ Child Workflow 1 (process chunk A)
    ├──▶ Child Workflow 2 (process chunk B)
    ├──▶ Child Workflow 3 (process chunk C)
    │
    └── Collect results (fan-in)
```

Why use Child Workflows instead of Activities for fan-out?

| Feature | Activity | Child Workflow |
|---------|----------|----------------|
| Own Event History | ❌ Shares parent's | ✅ Separate history |
| Independent failure | ❌ Fails parent | ✅ Can fail independently |
| Independent retry | ❌ Parent's retry policy | ✅ Own retry policy |
| Can be cancelled independently | ❌ | ✅ |
| History size impact | Adds to parent | No impact on parent |

!!! note "When NOT to use Child Workflows"
    Don't use Child Workflows just to break complex logic into smaller pieces — standard programming abstractions (functions, classes) work fine for that. Use Child Workflows when you need **separate failure domains**, **separate histories**, or **independent lifecycle control**.

### Continue-As-New = Stream Processing

For unbounded/infinite workflows (like a stream processor), the Event History would grow without limit. **Continue-As-New** solves this by "restarting" the workflow with fresh history while preserving state:

```
Workflow Execution 1 (history: 10,000 events)
    │
    │ continueAsNew(currentState)
    ▼
Workflow Execution 2 (history: 0 events)
    │ (same workflow ID, fresh history)
    │ (receives currentState as input)
    │
    │ continueAsNew(currentState)
    ▼
Workflow Execution 3 (history: 0 events)
    ...
```

This is analogous to a stream processor's **checkpointing** — periodically saving state and starting fresh.

### Schedules = Cron Jobs

Temporal **Schedules** provide cron-like recurring workflow execution — perfect for periodic batch jobs:

```
Schedule: "Run daily at 2 AM"
    │
    ├── 2026-04-20 02:00 → Start dailyReportWorkflow
    ├── 2026-04-21 02:00 → Start dailyReportWorkflow
    ├── 2026-04-22 02:00 → Start dailyReportWorkflow
    └── ...
```

---

## Part 3: Child Workflows

### Basic Child Workflow

```typescript
import { executeChild, workflowInfo } from '@temporalio/workflow';

export async function parentWorkflow(items: string[]): Promise<string[]> {
  const results: string[] = [];

  for (const item of items) {
    const result = await executeChild(processItemWorkflow, {
      args: [item],
      workflowId: `${workflowInfo().workflowId}-item-${item}`,
    });
    results.push(result);
  }

  return results;
}

export async function processItemWorkflow(item: string): Promise<string> {
  // This has its own Event History
  const validated = await validateItem(item);
  const enriched = await enrichItem(validated);
  const stored = await storeItem(enriched);
  return stored;
}
```

### Parallel Fan-Out / Fan-In

```typescript
import { executeChild, workflowInfo } from '@temporalio/workflow';

export async function fanOutWorkflow(chunks: string[][]): Promise<number> {
  // Fan-out: start all child workflows in parallel
  const results = await Promise.all(
    chunks.map((chunk, index) =>
      executeChild(processChunkWorkflow, {
        args: [chunk],
        workflowId: `${workflowInfo().workflowId}-chunk-${index}`,
      })
    )
  );

  // Fan-in: aggregate results
  const totalProcessed = results.reduce((sum, count) => sum + count, 0);
  return totalProcessed;
}

export async function processChunkWorkflow(chunk: string[]): Promise<number> {
  let processed = 0;
  for (const item of chunk) {
    await processItem(item);
    processed++;
  }
  return processed;
}
```

### Parent Close Policies

What happens to child workflows when the parent completes or fails?

```typescript
import { executeChild, ParentClosePolicy } from '@temporalio/workflow';

// Default: child is terminated when parent closes
await executeChild(childWorkflow, {
  args: [input],
  parentClosePolicy: ParentClosePolicy.TERMINATE,
});

// Child continues running independently
await executeChild(childWorkflow, {
  args: [input],
  parentClosePolicy: ParentClosePolicy.ABANDON,
});

// Cancellation is requested but not forced
await executeChild(childWorkflow, {
  args: [input],
  parentClosePolicy: ParentClosePolicy.REQUEST_CANCEL,
});
```

| Policy | Behavior | Use When |
|--------|----------|----------|
| `TERMINATE` (default) | Child is killed immediately | Child's work is meaningless without parent |
| `ABANDON` | Child continues independently | Child should complete regardless of parent |
| `REQUEST_CANCEL` | Cancellation requested, child can clean up | Child needs graceful shutdown |

---

## Part 4: Continue-As-New

### The Problem: Unbounded History

Temporal has a practical limit on Event History size:

- **Max 50MB** for complete history (aim for <10MB)
- **~10,000+ events** is a good threshold to consider continue-as-new

For long-running or infinite workflows, the history grows without bound. Continue-As-New resets it.

### Basic Continue-As-New

```typescript
import { continueAsNew, workflowInfo, log } from '@temporalio/workflow';

interface MonitorState {
  checksPerformed: number;
  lastStatus: string;
  alertsSent: number;
}

export async function monitoringWorkflow(state: MonitorState): Promise<string> {
  while (true) {
    // Do work
    const status = await checkHealth();
    state.checksPerformed++;
    state.lastStatus = status;

    if (status === 'unhealthy') {
      await sendAlert(state);
      state.alertsSent++;
    }

    await sleep('1 minute');

    // Check if we should continue-as-new
    const info = workflowInfo();
    if (info.continueAsNewSuggested || info.historyLength > 10000) {
      log.info('Continuing as new', {
        historyLength: info.historyLength,
        checksPerformed: state.checksPerformed,
      });
      await continueAsNew<typeof monitoringWorkflow>(state);
      // Code after continueAsNew never executes
    }
  }
}
```

Key points:

- **`workflowInfo().continueAsNewSuggested`** — Temporal tells you when it's time
- **`workflowInfo().historyLength`** — Check the current history size
- **`continueAsNew<typeof workflow>(state)`** — Pass current state to the new execution
- Code after `continueAsNew` **never executes** — it's like a `return`

### Continue-As-New Preserves Identity

The new workflow execution has:

- ✅ Same **Workflow ID**
- ✅ Same **Task Queue**
- ❌ New **Run ID** (different execution)
- ❌ Fresh **Event History** (starts at 0)

From the outside, it looks like the same workflow. The Workflow ID remains constant.

---

## Part 5: Schedules

### Creating a Schedule

```typescript
import { Client, ScheduleOverlapPolicy } from '@temporalio/client';

const client = new Client();

const schedule = await client.schedule.create({
  scheduleId: 'daily-report',
  spec: {
    // Run every day at 2 AM UTC
    intervals: [{ every: '1 day' }],
  },
  action: {
    type: 'startWorkflow',
    workflowType: 'dailyReportWorkflow',
    taskQueue: 'reports',
    args: [],
  },
  policies: {
    // What to do if previous run is still going when next is scheduled
    overlap: ScheduleOverlapPolicy.SKIP,
  },
});
```

### Overlap Policies

| Policy | Behavior |
|--------|----------|
| `SKIP` | Don't start new if previous is still running |
| `BUFFER_ONE` | Buffer one execution, start when previous completes |
| `BUFFER_ALL` | Buffer all, run sequentially |
| `CANCEL_OTHER` | Cancel the running execution, start new |
| `TERMINATE_OTHER` | Terminate the running execution, start new |
| `ALLOW_ALL` | Start new regardless (parallel executions) |

### Managing Schedules

```typescript
const handle = client.schedule.getHandle('daily-report');

// Pause during maintenance
await handle.pause('Maintenance window');

// Resume
await handle.unpause();

// Trigger immediately (outside schedule)
await handle.trigger();

// Delete the schedule
await handle.delete();
```

---

## Part 6: Build It — Document Processing Pipeline

Let's build a document processing pipeline that demonstrates child workflows (fan-out/fan-in), continue-as-new (for monitoring), and schedules.

### The Scenario

A system that:

1. Receives a batch of document IDs
2. Fans out processing to child workflows (one per document)
3. Collects results (fan-in)
4. A monitoring workflow runs continuously, checking pipeline health

### Activity Implementations

```typescript
// src/activities/document.ts
import { log } from '@temporalio/activity';

export interface Document {
  id: string;
  content: string;
  wordCount: number;
}

export interface ProcessingResult {
  documentId: string;
  wordCount: number;
  status: 'processed' | 'failed';
  error?: string;
}

export async function fetchDocument(documentId: string): Promise<Document> {
  log.info('Fetching document', { documentId });
  // Simulate fetching from storage
  return {
    id: documentId,
    content: `Content of document ${documentId}. `.repeat(Math.floor(Math.random() * 100) + 10),
    wordCount: 0,
  };
}

export async function analyzeDocument(doc: Document): Promise<Document> {
  log.info('Analyzing document', { documentId: doc.id });
  // Simulate analysis
  const wordCount = doc.content.split(/\s+/).length;
  return { ...doc, wordCount };
}

export async function storeResult(result: ProcessingResult): Promise<void> {
  log.info('Storing result', {
    documentId: result.documentId,
    wordCount: result.wordCount,
    status: result.status,
  });
}

export async function checkPipelineHealth(): Promise<string> {
  log.info('Checking pipeline health');
  // Simulate health check
  const healthy = Math.random() > 0.1;
  return healthy ? 'healthy' : 'degraded';
}

export async function sendHealthAlert(status: string, checksPerformed: number): Promise<void> {
  log.info('Sending health alert', { status, checksPerformed });
}

export async function generateBatchReport(
  batchId: string,
  totalDocs: number,
  succeeded: number,
  failed: number
): Promise<string> {
  log.info('Generating batch report', { batchId, totalDocs, succeeded, failed });
  return `Batch ${batchId}: ${succeeded}/${totalDocs} succeeded, ${failed} failed`;
}
```

### Child Workflow: Process Single Document

```typescript
// src/workflows/process-document.ts
import { proxyActivities, log } from '@temporalio/workflow';
import type * as docActivities from '../activities/document';
import type { ProcessingResult } from '../activities/document';

const { fetchDocument, analyzeDocument, storeResult } =
  proxyActivities<typeof docActivities>({
    startToCloseTimeout: '2 minutes',
    retry: { maximumAttempts: 3 },
  });

export async function processDocumentWorkflow(documentId: string): Promise<ProcessingResult> {
  log.info('Processing document', { documentId });

  try {
    // Fetch
    const doc = await fetchDocument(documentId);

    // Analyze
    const analyzed = await analyzeDocument(doc);

    // Store
    const result: ProcessingResult = {
      documentId,
      wordCount: analyzed.wordCount,
      status: 'processed',
    };
    await storeResult(result);

    log.info('Document processed', { documentId, wordCount: analyzed.wordCount });
    return result;

  } catch (err) {
    const result: ProcessingResult = {
      documentId,
      wordCount: 0,
      status: 'failed',
      error: err instanceof Error ? err.message : 'Unknown error',
    };
    await storeResult(result);
    return result;
  }
}
```

### Parent Workflow: Batch Processing with Fan-Out/Fan-In

```typescript
// src/workflows/batch-processing.ts
import { executeChild, proxyActivities, workflowInfo, log } from '@temporalio/workflow';
import { processDocumentWorkflow } from './process-document';
import type * as docActivities from '../activities/document';
import type { ProcessingResult } from '../activities/document';

const { generateBatchReport } = proxyActivities<typeof docActivities>({
  startToCloseTimeout: '1 minute',
});

export interface BatchResult {
  batchId: string;
  total: number;
  succeeded: number;
  failed: number;
  results: ProcessingResult[];
  report: string;
}

export async function batchProcessingWorkflow(
  batchId: string,
  documentIds: string[]
): Promise<BatchResult> {
  log.info('Starting batch processing', {
    batchId,
    documentCount: documentIds.length,
  });

  const parentId = workflowInfo().workflowId;

  // Fan-out: Process all documents in parallel via child workflows
  const results = await Promise.all(
    documentIds.map((docId) =>
      executeChild(processDocumentWorkflow, {
        args: [docId],
        workflowId: `${parentId}-doc-${docId}`,
      })
    )
  );

  // Fan-in: Aggregate results
  const succeeded = results.filter((r) => r.status === 'processed').length;
  const failed = results.filter((r) => r.status === 'failed').length;

  log.info('Batch processing complete', { batchId, succeeded, failed });

  // Generate report
  const report = await generateBatchReport(batchId, documentIds.length, succeeded, failed);

  return {
    batchId,
    total: documentIds.length,
    succeeded,
    failed,
    results,
    report,
  };
}
```

### Monitoring Workflow with Continue-As-New

```typescript
// src/workflows/pipeline-monitor.ts
import { continueAsNew, workflowInfo, sleep, log, proxyActivities } from '@temporalio/workflow';
import type * as docActivities from '../activities/document';

const { checkPipelineHealth, sendHealthAlert } = proxyActivities<typeof docActivities>({
  startToCloseTimeout: '30 seconds',
});

export interface MonitorState {
  checksPerformed: number;
  lastStatus: string;
  alertsSent: number;
  startedAt: string;
}

export async function pipelineMonitorWorkflow(
  state?: MonitorState
): Promise<void> {
  // Initialize state on first run
  const monitorState: MonitorState = state ?? {
    checksPerformed: 0,
    lastStatus: 'unknown',
    alertsSent: 0,
    startedAt: new Date().toISOString(),
  };

  // Run monitoring loop
  for (let i = 0; i < 50; i++) {
    const status = await checkPipelineHealth();
    monitorState.checksPerformed++;
    monitorState.lastStatus = status;

    if (status !== 'healthy') {
      await sendHealthAlert(status, monitorState.checksPerformed);
      monitorState.alertsSent++;
    }

    log.info('Health check', {
      check: monitorState.checksPerformed,
      status,
      alertsSent: monitorState.alertsSent,
    });

    await sleep('10 seconds');
  }

  // After 50 iterations, continue-as-new to reset history
  const info = workflowInfo();
  log.info('Continuing as new', {
    historyLength: info.historyLength,
    checksPerformed: monitorState.checksPerformed,
  });

  await continueAsNew<typeof pipelineMonitorWorkflow>(monitorState);
}
```

### Worker

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as docActivities from './activities/document';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows'),
    activities: docActivities,
    taskQueue: 'document-queue',
  });

  console.log('Worker started on task queue: document-queue');
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
import { Client, ScheduleOverlapPolicy } from '@temporalio/client';
import { batchProcessingWorkflow } from './workflows/batch-processing';
import { pipelineMonitorWorkflow } from './workflows/pipeline-monitor';

async function run() {
  const client = new Client();

  // 1. Start a batch processing workflow (fan-out/fan-in)
  console.log('=== Starting Batch Processing ===');
  const batchResult = await client.workflow.execute(batchProcessingWorkflow, {
    workflowId: 'batch-2026-04-20',
    taskQueue: 'document-queue',
    args: ['batch-001', ['doc-1', 'doc-2', 'doc-3', 'doc-4', 'doc-5']],
  });
  console.log('Batch result:', JSON.stringify(batchResult, null, 2));

  // 2. Start the monitoring workflow (continue-as-new)
  console.log('\n=== Starting Pipeline Monitor ===');
  const monitorHandle = await client.workflow.start(pipelineMonitorWorkflow, {
    workflowId: 'pipeline-monitor',
    taskQueue: 'document-queue',
    args: [undefined],  // No initial state
  });
  console.log(`Monitor started: ${monitorHandle.workflowId}`);
  console.log('(Monitor runs indefinitely via continue-as-new)');

  // 3. Create a schedule for daily batch processing
  console.log('\n=== Creating Daily Schedule ===');
  try {
    await client.schedule.create({
      scheduleId: 'daily-batch',
      spec: {
        intervals: [{ every: '1 day' }],
      },
      action: {
        type: 'startWorkflow',
        workflowType: 'batchProcessingWorkflow',
        taskQueue: 'document-queue',
        args: ['daily-batch', ['doc-a', 'doc-b', 'doc-c']],
      },
      policies: {
        overlap: ScheduleOverlapPolicy.SKIP,
      },
    });
    console.log('Schedule created: daily-batch');
  } catch (err) {
    console.log('Schedule may already exist:', (err as Error).message);
  }
}

run().catch(console.error);
```

### Run It

1. **Start the Temporal dev server:** `temporal server start-dev`
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`

**What to observe in the Web UI:**

- **`batch-2026-04-20`** — Parent workflow with 5 child workflows (`batch-2026-04-20-doc-1` through `doc-5`). Each child has its own Event History.
- **`pipeline-monitor`** — Runs continuously. After ~50 checks, it continues-as-new. The Workflow ID stays the same, but the Run ID changes.
- **Schedules tab** — The `daily-batch` schedule appears, showing next execution time.

---

## Part 7: Experiments

### Experiment 1: Observe Child Workflow Isolation

In the Web UI, click on the parent workflow `batch-2026-04-20`. You'll see `ChildWorkflowExecutionStarted` and `ChildWorkflowExecutionCompleted` events — but not the child's internal activities. Click on a child workflow to see its separate Event History.

### Experiment 2: Watch Continue-As-New

Open `pipeline-monitor` in the Web UI. After ~50 health checks, you'll see the workflow complete with `ContinueAsNewInitiated`. Click "Continue" to see the next execution — same Workflow ID, new Run ID, fresh history, but the state (checksPerformed, alertsSent) carries over.

### Experiment 3: Kill Worker During Fan-Out

1. Start a batch with 20 documents
2. Kill the worker while child workflows are running
3. Restart the worker
4. Observe: completed children are not re-run, only pending ones resume

### Experiment 4: Trigger Schedule Manually

```bash
temporal schedule trigger --schedule-id daily-batch
```

This starts a batch processing workflow immediately, outside the normal schedule.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapters 10–11** teach that batch processing (MapReduce) uses fan-out/fan-in on bounded data, while stream processing handles unbounded data with checkpointing. Both need exactly-once or idempotent processing.

    2. **Child Workflows** implement the fan-out/fan-in pattern — each child has its own Event History, failure domain, and retry policy. Use them when you need isolation, not just code organization.

    3. **Continue-As-New** prevents unbounded history growth for long-running workflows. It "restarts" with fresh history while preserving state — analogous to stream processing checkpoints. Check `workflowInfo().continueAsNewSuggested` or `historyLength > 10000`.

    4. **Schedules** provide cron-like recurring execution with overlap policies. Use `ScheduleOverlapPolicy.SKIP` to prevent overlapping runs.

    5. **Parent Close Policies** control child lifecycle:
        - `TERMINATE` (default) — Kill child when parent closes
        - `ABANDON` — Child continues independently
        - `REQUEST_CANCEL` — Graceful cancellation request

---

## Next Module

[:octicons-arrow-right-24: Module 8 — Observability & Versioning](module-08-observability.md)
