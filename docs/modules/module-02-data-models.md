# Module 2 — Data Models & Activity Design

!!! abstract "Module Overview"
    **DDIA Reference:** Chapters 2–3 — Data Models, Storage & Retrieval  
    **Temporal Concepts:** Activity design, data handling, idempotency, error classification, payload limits  
    **Time:** ~3 hours

---

## Part 1: Theory — Data Models & Encoding (DDIA Chapters 2–3)

### Data Models Shape How We Think

Kleppmann's Chapter 2 makes a profound observation: **data models are the most important part of developing software**, because they affect not only how the software is written, but also how we *think about the problem*.

Most applications are built by layering one data model on top of another:

1. You model the real world in terms of objects, data structures, and APIs
2. You store those data structures in a general-purpose data model (JSON, tables, graphs)
3. The database represents that data in terms of bytes on disk
4. The hardware represents bytes as electrical currents, magnetic fields, etc.

Each layer hides the complexity of the layer below by providing a clean abstraction.

### Relational vs Document vs Graph

DDIA Chapter 2 compares three major data models:

| Model | Strengths | Weaknesses | Example |
|-------|-----------|------------|---------|
| **Relational** | Joins, many-to-many relationships, schema enforcement | Rigid schema, impedance mismatch with objects | PostgreSQL, MySQL |
| **Document** | Schema flexibility, data locality, natural for self-contained records | Poor for many-to-many, joins are weak | MongoDB, CouchDB |
| **Graph** | Excellent for highly connected data, flexible schema | Complex queries, less mature tooling | Neo4j, DGraph |

The key insight: **there is no one-size-fits-all data model**. The best choice depends on the relationships in your data.

### Schema-on-Write vs Schema-on-Read

| Approach | When Schema Is Enforced | Analogy | Use When |
|----------|------------------------|---------|----------|
| **Schema-on-write** | At write time (database rejects invalid data) | Static typing | Structure is known and stable |
| **Schema-on-read** | At read time (application interprets data) | Dynamic typing | Structure varies or evolves frequently |

### Storage & Retrieval (Chapter 3)

Chapter 3 dives into how databases actually store and retrieve data:

**Log-structured storage (LSM-Trees / SSTables):**

- Writes are appended to an in-memory buffer (memtable), then flushed to sorted files on disk
- Reads may need to check multiple files, aided by bloom filters
- Compaction merges files in the background
- Optimized for **write-heavy** workloads

**Page-oriented storage (B-Trees):**

- Data is organized in fixed-size pages (typically 4KB)
- Updates modify pages in place
- A write-ahead log (WAL) ensures crash recovery
- Optimized for **read-heavy** workloads

### Encoding and Evolution

Data needs to be encoded (serialized) for storage or network transmission. DDIA covers several formats:

| Format | Schema? | Human-readable? | Compact? | Evolution Support |
|--------|---------|-----------------|----------|-------------------|
| JSON | No (implicit) | Yes | No | Field addition/removal OK |
| XML | Optional (XSD) | Yes | No | Verbose but flexible |
| Protocol Buffers | Yes (.proto) | No | Yes | Field tags enable evolution |
| Avro | Yes (.avsc) | No | Yes | Schema resolution at read time |

The critical concept: **forward and backward compatibility**.

- **Backward compatibility** — New code can read data written by old code
- **Forward compatibility** — Old code can read data written by new code

Both are essential in distributed systems where different services may be running different versions of code simultaneously.

---

## Part 2: Mental Model — Activities as the Data Boundary

### Why This Matters for Temporal

In a Temporal application, data flows through several boundaries:

```
Client → Temporal Service → Worker → Workflow → Activity → External System
                                                    ↓
                                              Database / API
```

At each boundary, data is **serialized and deserialized**. Understanding data models and encoding is critical because:

1. **Workflow inputs/outputs** are serialized and stored in the Event History
2. **Activity inputs/outputs** are serialized and stored in the Event History
3. **Payload size limits** exist (max 2MB per payload, 4MB per gRPC message)
4. **Schema evolution** matters when you change workflow or activity signatures while workflows are running

### Activities: The Boundary Between Deterministic and Non-Deterministic

In Module 1, we learned the golden rule: *Workflows orchestrate, Activities execute*. Now let's understand Activities more deeply.

An Activity is where your workflow interacts with the outside world:

```
┌─────────────────────────────────────────────────┐
│  Workflow (Deterministic Sandbox)                │
│                                                  │
│  ┌──────────────┐    ┌──────────────┐           │
│  │ Orchestration │───▶│ proxyActivities │        │
│  │ Logic         │    │ (type-only)     │        │
│  └──────────────┘    └───────┬──────┘           │
└──────────────────────────────┼──────────────────┘
                               │ Command: ScheduleActivityTask
                               ▼
┌──────────────────────────────────────────────────┐
│  Temporal Service                                 │
│  Persists: input args, result, errors             │
└──────────────────────────────┬───────────────────┘
                               │ Task dispatched to Worker
                               ▼
┌──────────────────────────────────────────────────┐
│  Activity (No Sandbox — Full Node.js)             │
│                                                   │
│  ✅ Network calls    ✅ File I/O                  │
│  ✅ Database queries ✅ Math.random()             │
│  ✅ Current time     ✅ External APIs             │
└──────────────────────────────────────────────────┘
```

Key insight: **Activity inputs and outputs are persisted in the Event History**. This means:

- Keep payloads small (references, not raw data)
- Ensure data is JSON-serializable (the default converter)
- Think about schema evolution when changing activity signatures

### The Temporal Data Converter

The Temporal TypeScript SDK uses a **Data Converter** to serialize and deserialize all data that crosses the workflow/activity boundary. The default converter handles:

- `undefined` and `null`
- `Uint8Array` (as binary)
- Any JSON-serializable type

This means your activity inputs and outputs must be JSON-serializable by default. Classes with methods, functions, `Map`, `Set`, and circular references will **not** work without a custom converter.

---

## Part 3: Designing Good Activities

### Principle 1: Keep Activities Focused

Each activity should do **one thing**. This gives you:

- **Granular retries** — If sending an email fails, you don't re-run the database write
- **Clear error handling** — Each activity has its own timeout and retry policy
- **Better visibility** — The Event History shows exactly which step failed

```typescript
// ❌ BAD — One giant activity that does everything
export async function processOrder(order: Order): Promise<void> {
  await db.insert(order);           // Step 1
  await paymentApi.charge(order);   // Step 2
  await emailService.send(order);   // Step 3
}

// ✅ GOOD — Separate activities for each concern
export async function saveOrder(order: Order): Promise<string> {
  const result = await db.insert(order);
  return result.id;
}

export async function chargePayment(orderId: string, amount: number): Promise<string> {
  const result = await paymentApi.charge({ orderId, amount });
  return result.transactionId;
}

export async function sendConfirmation(email: string, orderId: string): Promise<void> {
  await emailService.send({ to: email, template: 'order-confirmation', data: { orderId } });
}
```

### Principle 2: Make Activities Idempotent

Temporal may re-execute activities during **retries** (on failure) or after **worker restarts**. Without idempotency, this causes duplicate side effects: double charges, duplicate emails, duplicate database entries.

#### Strategy 1: Idempotency Keys

Pass a unique identifier to external services so they can deduplicate:

```typescript
import { activityInfo } from '@temporalio/activity';

export async function chargePayment(orderId: string, amount: number): Promise<string> {
  const info = activityInfo();

  // Use workflowId + activityId as idempotency key
  const idempotencyKey = `${info.workflowExecution.workflowId}-charge-${orderId}`;

  const result = await paymentApi.charge({
    amount,
    idempotencyKey,  // Payment provider deduplicates based on this
  });

  return result.transactionId;
}
```

Good idempotency key sources:

- **Workflow ID** — unique per workflow execution
- **Business identifier** — order ID, transaction ID
- **Workflow ID + activity name** — unique per activity within a workflow

#### Strategy 2: Check-Before-Act

Query the current state before making changes:

```typescript
export async function sendWelcomeEmail(userId: string, email: string): Promise<void> {
  // Check if already sent
  const alreadySent = await db.query(
    'SELECT sent FROM welcome_emails WHERE user_id = $1',
    [userId]
  );

  if (alreadySent) {
    return; // Already done, skip
  }

  await emailService.send({ to: email, template: 'welcome' });
  await db.insert('welcome_emails', { user_id: userId, sent: true });
}
```

### Principle 3: Classify Errors Correctly

Not all errors are the same. Temporal needs to know which errors are **retryable** (transient) and which are **non-retryable** (permanent):

| Error Type | Retryable? | Examples |
|-----------|-----------|---------|
| **Transient** | ✅ Yes | Network timeout, rate limit (429), service unavailable (503) |
| **Permanent** | ❌ No | Invalid input (400), not found (404), authentication failure (401) |

```typescript
import { ApplicationFailure } from '@temporalio/activity';

export async function lookupUser(userId: string): Promise<User> {
  try {
    const response = await fetch(`https://api.example.com/users/${userId}`);

    if (response.status === 404) {
      // Permanent error — don't retry, this user doesn't exist
      throw ApplicationFailure.nonRetryable(`User ${userId} not found`);
    }

    if (response.status === 429) {
      // Transient error — Temporal will retry automatically
      throw new Error('Rate limited, will retry');
    }

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    return await response.json();
  } catch (err) {
    if (err instanceof ApplicationFailure) throw err; // Re-throw non-retryable
    throw err; // All other errors are retryable by default
  }
}
```

### Principle 4: Handle Large Data with References

Temporal has payload size limits:

- **Max 2MB** per individual payload
- **Max 4MB** per gRPC message
- **Max 50MB** for complete workflow history (aim for <10MB)

**Never pass large data through the workflow**. Instead, pass references:

```typescript
// ❌ BAD — Large data flows through workflow history
export async function downloadFile(url: string): Promise<Buffer> {
  const response = await fetch(url);
  return Buffer.from(await response.arrayBuffer()); // Enters history!
}

// ✅ GOOD — Activity handles large data internally, returns a reference
export async function processFile(inputUrl: string): Promise<string> {
  // Download inside the activity
  const data = await fetch(inputUrl).then(r => r.buffer());

  // Process inside the activity
  const result = transform(data);

  // Upload inside the activity
  const outputKey = await s3.upload('results', result);

  // Return only a small reference
  return outputKey;
}
```

The workflow only sees small strings (references), never the large data itself.

---

## Part 4: File Organization

The TypeScript SDK **bundles workflow files** and runs them in an isolated V8 sandbox. This has important implications for how you organize your code.

### Recommended Structure

```
src/
├── workflows/
│   ├── order.ts          # Only workflow functions
│   └── signup.ts         # Only workflow functions
├── activities/
│   ├── payment.ts        # Only activity functions
│   ├── email.ts          # Only activity functions
│   └── database.ts       # Only activity functions
├── shared/
│   └── types.ts          # Shared interfaces/types (no I/O!)
├── worker.ts             # Worker setup — imports both
└── client.ts             # Client code to start workflows
```

### Rules

1. **Workflow files** — Only workflow functions and imports from `@temporalio/workflow`. Use `import type` for activity types.
2. **Activity files** — Regular Node.js code. Can import any library.
3. **Shared types** — Interfaces and type definitions used by both. Must not contain any I/O or side effects.
4. **Worker** — Imports activities directly (not via proxy) and points to workflow files.

```typescript
// workflows/order.ts — CORRECT
import { proxyActivities, log } from '@temporalio/workflow';
import type * as paymentActivities from '../activities/payment';  // type-only!
import type * as emailActivities from '../activities/email';      // type-only!
import type { Order } from '../shared/types';                     // type-only!

const { chargePayment, refundPayment } = proxyActivities<typeof paymentActivities>({
  startToCloseTimeout: '5 minutes',
});

const { sendConfirmation } = proxyActivities<typeof emailActivities>({
  startToCloseTimeout: '1 minute',
});

export async function orderWorkflow(order: Order): Promise<string> {
  const txnId = await chargePayment(order.id, order.total);
  await sendConfirmation(order.email, order.id);
  return txnId;
}
```

!!! warning "Why `import type` matters"
    Using `import * as activities` (without `type`) brings the actual Node.js code into the workflow sandbox bundle. This causes bundling errors or runtime failures because Node.js modules like `fs`, `http`, and database drivers are **blocked** in the V8 sandbox.

---

## Part 5: Retry Policies & Timeout Configuration

### Timeout Types

When you create activity proxies with `proxyActivities`, you configure timeouts:

```typescript
const { myActivity } = proxyActivities<typeof activities>({
  startToCloseTimeout: '5 minutes',      // Max time for a single attempt
  scheduleToCloseTimeout: '30 minutes',  // Max time including all retries
  heartbeatTimeout: '30 seconds',        // Max time between heartbeats
});
```

| Timeout | What It Limits | When to Set |
|---------|---------------|-------------|
| `startToCloseTimeout` | Duration of a **single attempt** | Always set this — it's required |
| `scheduleToCloseTimeout` | Total duration **including retries** | Set when you have an overall deadline |
| `heartbeatTimeout` | Time between **heartbeat calls** | Set for long-running activities that heartbeat |

### Retry Policy

Temporal retries failed activities automatically. The default policy is usually sufficient, but you can customize it:

```typescript
const { chargePayment } = proxyActivities<typeof paymentActivities>({
  startToCloseTimeout: '30 seconds',
  retry: {
    initialInterval: '1s',           // First retry after 1 second
    backoffCoefficient: 2,           // Double the interval each retry
    maximumInterval: '1m',           // Cap at 1 minute between retries
    maximumAttempts: 5,              // Give up after 5 attempts
    nonRetryableErrorTypes: ['ValidationError', 'NotFoundError'],
  },
});
```

!!! tip "When to customize retries"
    Only set retry options if you have a **domain-specific reason**. The defaults (infinite retries with exponential backoff) are suitable for most use cases. The most common customization is setting `nonRetryableErrorTypes` to prevent retrying permanent errors.

---

## Part 6: Build It — Product Catalog Service

Let's build a product catalog workflow that demonstrates proper activity design, data handling, idempotency, and error classification.

### The Scenario

An e-commerce product import workflow that:

1. Fetches product data from an external API
2. Validates the data
3. Saves to a database
4. Indexes for search

### Shared Types

```typescript
// src/shared/types.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  description: string;
}

export interface ImportResult {
  productId: string;
  status: 'created' | 'updated' | 'skipped';
}
```

### Activity Implementations

```typescript
// src/activities/product.ts
import { ApplicationFailure, log } from '@temporalio/activity';
import type { Product, ImportResult } from '../shared/types';

// Simulated external API and database
const externalApi = new Map<string, Product>();
const database = new Map<string, Product>();
const searchIndex = new Map<string, Product>();

// Seed some test data
externalApi.set('prod-1', {
  id: 'prod-1', name: 'Mechanical Keyboard', price: 149.99,
  category: 'electronics', description: 'RGB mechanical keyboard',
});
externalApi.set('prod-2', {
  id: 'prod-2', name: '', price: -10,  // Invalid data for testing
  category: 'electronics', description: 'Bad product',
});

export async function fetchProduct(productId: string): Promise<Product> {
  log.info('Fetching product from external API', { productId });

  // Simulate API call
  const product = externalApi.get(productId);

  if (!product) {
    // Non-retryable: product doesn't exist in source system
    throw ApplicationFailure.nonRetryable(
      `Product ${productId} not found in source system`
    );
  }

  return product;
}

export async function validateProduct(product: Product): Promise<Product> {
  log.info('Validating product', { productId: product.id });

  const errors: string[] = [];

  if (!product.name || product.name.trim().length === 0) {
    errors.push('Product name is required');
  }
  if (product.price <= 0) {
    errors.push('Product price must be positive');
  }
  if (!product.category) {
    errors.push('Product category is required');
  }

  if (errors.length > 0) {
    // Non-retryable: invalid data won't become valid on retry
    throw ApplicationFailure.nonRetryable(
      `Validation failed: ${errors.join(', ')}`
    );
  }

  return product;
}

export async function saveProduct(product: Product): Promise<ImportResult> {
  log.info('Saving product to database', { productId: product.id });

  // Check-before-act pattern for idempotency
  const existing = database.get(product.id);

  if (existing && JSON.stringify(existing) === JSON.stringify(product)) {
    log.info('Product unchanged, skipping', { productId: product.id });
    return { productId: product.id, status: 'skipped' };
  }

  const status = existing ? 'updated' : 'created';
  database.set(product.id, product);

  return { productId: product.id, status };
}

export async function indexProduct(product: Product): Promise<void> {
  log.info('Indexing product for search', { productId: product.id });

  // Simulate search indexing
  searchIndex.set(product.id, product);
}
```

Notice the patterns in action:

- **`fetchProduct`** — Throws `ApplicationFailure.nonRetryable()` for missing products (permanent error)
- **`validateProduct`** — Throws `ApplicationFailure.nonRetryable()` for invalid data (won't fix itself on retry)
- **`saveProduct`** — Uses check-before-act for idempotency (safe to retry)
- **`indexProduct`** — Simple operation, default retry is fine

### Workflow Definition

```typescript
// src/workflows/product-import.ts
import { proxyActivities, log, ApplicationFailure } from '@temporalio/workflow';
import type * as productActivities from '../activities/product';
import type { ImportResult } from '../shared/types';

const {
  fetchProduct,
  validateProduct,
  saveProduct,
  indexProduct,
} = proxyActivities<typeof productActivities>({
  startToCloseTimeout: '30 seconds',
  retry: {
    maximumAttempts: 3,
    nonRetryableErrorTypes: ['ValidationError'],
  },
});

export async function productImportWorkflow(productId: string): Promise<ImportResult> {
  log.info('Starting product import', { productId });

  // Step 1: Fetch from external source
  const rawProduct = await fetchProduct(productId);
  log.info('Product fetched', { productId });

  // Step 2: Validate
  const validProduct = await validateProduct(rawProduct);
  log.info('Product validated', { productId });

  // Step 3: Save to database
  const result = await saveProduct(validProduct);
  log.info('Product saved', { productId, status: result.status });

  // Step 4: Index for search (only if created or updated)
  if (result.status !== 'skipped') {
    await indexProduct(validProduct);
    log.info('Product indexed', { productId });
  }

  return result;
}
```

### Batch Import with Parallel Execution

Now let's add a workflow that imports multiple products in parallel:

```typescript
// src/workflows/batch-import.ts
import { proxyActivities, log } from '@temporalio/workflow';
import type * as productActivities from '../activities/product';
import type { ImportResult } from '../shared/types';

const {
  fetchProduct,
  validateProduct,
  saveProduct,
  indexProduct,
} = proxyActivities<typeof productActivities>({
  startToCloseTimeout: '30 seconds',
  retry: { maximumAttempts: 3 },
});

interface BatchResult {
  succeeded: ImportResult[];
  failed: Array<{ productId: string; error: string }>;
}

export async function batchImportWorkflow(productIds: string[]): Promise<BatchResult> {
  log.info('Starting batch import', { count: productIds.length });

  const results = await Promise.allSettled(
    productIds.map(async (productId) => {
      const raw = await fetchProduct(productId);
      const valid = await validateProduct(raw);
      const result = await saveProduct(valid);
      if (result.status !== 'skipped') {
        await indexProduct(valid);
      }
      return result;
    })
  );

  const succeeded: ImportResult[] = [];
  const failed: Array<{ productId: string; error: string }> = [];

  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      succeeded.push(result.value);
    } else {
      failed.push({
        productId: productIds[index],
        error: result.reason?.message ?? 'Unknown error',
      });
    }
  });

  log.info('Batch import complete', {
    succeeded: succeeded.length,
    failed: failed.length,
  });

  return { succeeded, failed };
}
```

!!! note "Promise.allSettled vs Promise.all"
    We use `Promise.allSettled` instead of `Promise.all` so that one product's failure doesn't abort the entire batch. Each product is processed independently — failures are collected and reported.

### Worker and Client

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as productActivities from './activities/product';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/product-import'),
    activities: productActivities,
    taskQueue: 'product-queue',
  });

  console.log('Worker started on task queue: product-queue');
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
import { productImportWorkflow } from './workflows/product-import';

async function run() {
  const client = new Client();

  // Import a valid product
  console.log('Importing prod-1...');
  const result = await client.workflow.execute(productImportWorkflow, {
    workflowId: 'import-prod-1',
    taskQueue: 'product-queue',
    args: ['prod-1'],
  });
  console.log('Result:', result);

  // Try importing an invalid product (will fail at validation)
  console.log('\nImporting prod-2 (invalid)...');
  try {
    await client.workflow.execute(productImportWorkflow, {
      workflowId: 'import-prod-2',
      taskQueue: 'product-queue',
      args: ['prod-2'],
    });
  } catch (err) {
    console.log('Expected failure:', (err as Error).message);
  }

  // Try importing a non-existent product
  console.log('\nImporting prod-999 (not found)...');
  try {
    await client.workflow.execute(productImportWorkflow, {
      workflowId: 'import-prod-999',
      taskQueue: 'product-queue',
      args: ['prod-999'],
    });
  } catch (err) {
    console.log('Expected failure:', (err as Error).message);
  }
}

run().catch(console.error);
```

### Run It

1. **Ensure the Temporal dev server is running**
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`
4. **Check the Web UI** at [http://localhost:8233](http://localhost:8233) — observe:
    - `import-prod-1` → Completed successfully
    - `import-prod-2` → Failed at validation (non-retryable)
    - `import-prod-999` → Failed at fetch (non-retryable)

---

## Part 7: Experiments

### Experiment 1: Watch Non-Retryable vs Retryable Errors

In the Web UI, compare the Event History of `import-prod-2` (non-retryable validation error) vs a workflow with a transient error. Non-retryable errors fail immediately — no retry attempts in the history.

### Experiment 2: Modify an Activity and Observe

Change `saveProduct` to throw a transient error on the first attempt:

```typescript
let attemptCount = 0;
export async function saveProduct(product: Product): Promise<ImportResult> {
  attemptCount++;
  if (attemptCount === 1) {
    throw new Error('Database temporarily unavailable');  // Retryable!
  }
  // ... rest of the function
}
```

Run the workflow and check the Event History — you'll see `ActivityTaskFailed` followed by a successful `ActivityTaskCompleted` on the retry.

### Experiment 3: Payload Size Awareness

Try passing a very large string as a product description and observe what happens. This builds intuition for why the "pass references, not data" pattern matters.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapters 2–3** teach that data models shape how we think about problems, and encoding/schema evolution is critical for systems that evolve over time

    2. **Activities are the data boundary** in Temporal — all I/O, API calls, and database operations happen here. Activity inputs and outputs are serialized and persisted in the Event History

    3. **Four principles of good activity design:**
        - **Focused** — One activity, one responsibility
        - **Idempotent** — Safe to retry (use idempotency keys or check-before-act)
        - **Error-classified** — Use `ApplicationFailure.nonRetryable()` for permanent errors
        - **Small payloads** — Pass references, not large data

    4. **File organization matters** — Keep workflows and activities in separate files. Use `import type` for activity imports in workflow files

    5. **Retry policies** — The defaults are usually fine. Customize `nonRetryableErrorTypes` to prevent retrying permanent errors. Set `startToCloseTimeout` on every activity

---

## Next Module

[:octicons-arrow-right-24: Module 3 — Replication & Reliability](module-03-replication.md)
