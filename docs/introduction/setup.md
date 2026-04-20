# Environment Setup

This guide walks you through setting up everything you need for the course.

---

## 1. Install Node.js

You need **Node.js 20+** and npm.

=== "macOS (Homebrew)"

    ```bash
    brew install node
    ```

=== "Using nvm"

    ```bash
    nvm install 20
    nvm use 20
    ```

Verify your installation:

```bash
node --version   # Should be v20.x or higher
npm --version    # Should be v10.x or higher
```

---

## 2. Install Temporal CLI

The Temporal CLI includes a local development server that you'll use throughout the course.

=== "macOS (Homebrew)"

    ```bash
    brew install temporal
    ```

=== "Linux (amd64)"

    Download from [temporal.download](https://temporal.download/cli/archive/latest?platform=linux&arch=amd64), extract the archive, and add the `temporal` binary to your PATH:

    ```bash
    # After downloading and extracting:
    sudo cp temporal /usr/local/bin/
    ```

=== "Linux (arm64)"

    Download from [temporal.download](https://temporal.download/cli/archive/latest?platform=linux&arch=arm64), extract the archive, and add the `temporal` binary to your PATH:

    ```bash
    # After downloading and extracting:
    sudo cp temporal /usr/local/bin/
    ```

Verify the installation:

```bash
temporal --version
```

---

## 3. Start the Temporal Dev Server

The Temporal dev server runs locally and provides everything you need: a Temporal cluster, a Web UI, and persistence.

```bash
temporal server start-dev
```

!!! info "Keep this running"
    The dev server needs to stay running in a terminal while you work. It's perfectly fine to share this single server across all modules and exercises.

Once started, open the **Temporal Web UI** at:

**[http://localhost:8233](http://localhost:8233)**

You should see an empty dashboard — this is where you'll monitor your workflows throughout the course.

---

## 4. Scaffold Your First Project

Create a new directory for the course exercises:

```bash
mkdir temporal-course && cd temporal-course
npm init -y
```

Install the Temporal TypeScript SDK packages:

```bash
npm install @temporalio/client @temporalio/worker @temporalio/workflow @temporalio/activity
```

!!! warning "Version consistency"
    All `@temporalio/*` packages **must** have the same version number. You can verify this with:
    ```bash
    npm ls | grep @temporalio
    ```

Install TypeScript and development dependencies:

```bash
npm install -D typescript ts-node @types/node
```

Create a `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "commonjs",
    "lib": ["ES2021"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"]
}
```

---

## 5. Project Structure

Create the following directory structure:

```
temporal-course/
├── src/
│   ├── workflows/       # Workflow definitions (deterministic)
│   ├── activities/      # Activity implementations (I/O, side effects)
│   ├── worker.ts        # Worker setup
│   └── client.ts        # Client code to start workflows
├── tsconfig.json
└── package.json
```

```bash
mkdir -p src/workflows src/activities
```

!!! tip "Why separate workflows and activities?"
    The Temporal TypeScript SDK **bundles workflow files** and runs them in an isolated V8 sandbox. Keeping workflow and activity code in separate files:

    - Ensures the workflow sandbox doesn't accidentally import Node.js modules
    - Improves Worker startup time by minimizing what gets bundled
    - Makes the determinism boundary clear: workflows = deterministic, activities = anything goes

---

## 6. Verify Everything Works

Let's create a minimal "hello world" to verify the setup.

**`src/activities/greet.ts`** — A simple activity:

```typescript
export async function greet(name: string): Promise<string> {
  return `Hello, ${name}!`;
}
```

**`src/workflows/greeting.ts`** — A workflow that calls the activity:

```typescript
import { proxyActivities } from '@temporalio/workflow';
// Use type-only import for activities — this is critical!
import type * as activities from '../activities/greet';

const { greet } = proxyActivities<typeof activities>({
  startToCloseTimeout: '1 minute',
});

export async function greetingWorkflow(name: string): Promise<string> {
  return await greet(name);
}
```

!!! warning "Type-only imports"
    Always use `import type * as activities` (not `import * as activities`) in workflow files. This ensures only the **type information** is imported into the workflow sandbox, not the actual Node.js code.

**`src/worker.ts`** — The worker that executes workflows and activities:

```typescript
import { Worker } from '@temporalio/worker';
import * as activities from './activities/greet';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/greeting'),
    activities,
    taskQueue: 'greeting-queue',
  });

  console.log('Worker started, polling on task queue: greeting-queue');
  await worker.run();
}

run().catch((err) => {
  console.error('Worker failed:', err);
  process.exit(1);
});
```

**`src/client.ts`** — Start a workflow execution:

```typescript
import { Client } from '@temporalio/client';
import { greetingWorkflow } from './workflows/greeting';

async function run() {
  const client = new Client();

  const result = await client.workflow.execute(greetingWorkflow, {
    workflowId: 'greeting-workflow-1',
    taskQueue: 'greeting-queue',
    args: ['Distributed Systems Learner'],
  });

  console.log(`Result: ${result}`);
}

run().catch((err) => {
  console.error('Client failed:', err);
  process.exit(1);
});
```

### Run It

1. **Ensure the Temporal dev server is running** (from step 3)

2. **Start the worker** (in a new terminal):
    ```bash
    npx ts-node src/worker.ts
    ```

3. **Execute the workflow** (in another terminal):
    ```bash
    npx ts-node src/client.ts
    ```

You should see:

```
Result: Hello, Distributed Systems Learner!
```

4. **Check the Web UI** at [http://localhost:8233](http://localhost:8233) — you should see your completed workflow with its full event history.

---

## What Just Happened?

Let's break down what Temporal did behind the scenes:

```
┌──────────┐     ┌──────────────────┐     ┌──────────┐
│  Client   │────▶│  Temporal Server  │◀────│  Worker   │
│ client.ts │     │  (Dev Server)     │     │ worker.ts │
└──────────┘     └──────────────────┘     └──────────┘
     │                    │                      │
     │  1. Start workflow │                      │
     │───────────────────▶│                      │
     │                    │  2. Schedule task     │
     │                    │─────────────────────▶│
     │                    │                      │ 3. Execute workflow
     │                    │                      │    → Call greet()
     │                    │  4. Report result     │
     │                    │◀─────────────────────│
     │  5. Return result  │                      │
     │◀───────────────────│                      │
```

1. The **Client** sent a "start workflow" request to the Temporal Server
2. The Temporal Server placed a task on the **task queue** (`greeting-queue`)
3. The **Worker** polled the task queue, picked up the task, and executed the workflow
4. The workflow called the `greet` activity, which returned a result
5. The result was stored in the **event history** and returned to the client

The key insight: **the workflow's execution is durable**. If the worker crashed mid-execution, another worker could pick up the task and resume from where it left off, using the event history.

---

## Troubleshooting

??? question "Worker won't start — connection error"
    Make sure the Temporal dev server is running:
    ```bash
    temporal server start-dev
    ```

??? question "Package version mismatch errors"
    All `@temporalio/*` packages must have the same version:
    ```bash
    npm ls | grep @temporalio
    ```
    If versions differ, reinstall:
    ```bash
    npm install @temporalio/client@latest @temporalio/worker@latest @temporalio/workflow@latest @temporalio/activity@latest
    ```

??? question "Workflow never completes"
    Check that:

    1. The worker is running and connected to the correct task queue
    2. The task queue name matches between client and worker
    3. The Temporal dev server is running

---

## Next Step

You're all set! Let's dive into the first module.

[:octicons-arrow-right-24: Module 1: Foundations of Distributed Systems](../modules/module-01-foundations.md)
