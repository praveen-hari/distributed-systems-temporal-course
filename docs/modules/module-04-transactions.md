# Module 4 — Transactions & the Saga Pattern

!!! abstract "Module Overview"
    **DDIA Reference:** Chapter 7 — Transactions  
    **Temporal Concepts:** Saga pattern, compensation logic, `CancellationScope.nonCancellable`, error handling in multi-step workflows  
    **Time:** ~3 hours

---

## Part 1: Theory — Transactions (DDIA Chapter 7)

### Why Transactions Exist

Kleppmann opens Chapter 7 with a simple observation: many things can go wrong in data systems:

- The database software or hardware may fail at any time (including in the middle of a write)
- The application may crash at any time (including halfway through a series of operations)
- Network interruptions can cut off the application from the database, or one database node from another
- Several clients may write to the database at the same time, overwriting each other's changes
- A client may read data that doesn't make sense because it has only been partially updated

**Transactions** are the mechanism databases use to simplify these problems. A transaction groups several reads and writes into a logical unit — either the entire transaction succeeds (commit), or it fails and is rolled back (abort).

### ACID Properties

The safety guarantees provided by transactions are described by the acronym **ACID**:

| Property | Meaning | What It Guarantees |
|----------|---------|-------------------|
| **Atomicity** | All-or-nothing | If a transaction fails partway, all changes are rolled back. No partial state. |
| **Consistency** | Valid state transitions | The database moves from one valid state to another. Application-level invariants are preserved. |
| **Isolation** | Concurrent transactions don't interfere | Each transaction behaves as if it's the only one running. |
| **Durability** | Committed data survives crashes | Once a transaction commits, the data is safely stored. |

!!! note "Atomicity is the key concept for this module"
    In the context of distributed systems, **atomicity** is the most relevant property. It means: if a multi-step operation fails partway through, all completed steps are undone. No partial state is left behind.

### The Problem with Distributed Transactions

ACID transactions work well within a single database. But modern applications span **multiple services**, each with its own database:

```
Order Service          Payment Service         Inventory Service
┌──────────┐          ┌──────────┐            ┌──────────┐
│ Orders DB │          │ Payments │            │ Inventory │
│           │          │    DB    │            │    DB     │
└──────────┘          └──────────┘            └──────────┘
```

A single order involves writes to all three databases. How do you ensure atomicity across them?

#### Two-Phase Commit (2PC)

The traditional answer is **Two-Phase Commit**:

```
Phase 1 (Prepare):
  Coordinator → Order DB:     "Can you commit?"  → "Yes"
  Coordinator → Payment DB:   "Can you commit?"  → "Yes"
  Coordinator → Inventory DB: "Can you commit?"  → "Yes"

Phase 2 (Commit):
  Coordinator → Order DB:     "Commit!"
  Coordinator → Payment DB:   "Commit!"
  Coordinator → Inventory DB: "Commit!"
```

**The problems with 2PC** (as Kleppmann explains):

1. **Blocking** — If the coordinator crashes after Phase 1 but before Phase 2, all participants are stuck holding locks, waiting for a decision that may never come
2. **Single point of failure** — The coordinator is a critical component
3. **Performance** — Holding locks across multiple databases is slow
4. **Heterogeneous systems** — 2PC requires all participants to support the same protocol. You can't do 2PC across a REST API, a message queue, and a database

In practice, **2PC is rarely used across microservices**. It's too slow, too fragile, and most external services don't support it.

### The Saga Pattern: An Alternative

The **Saga pattern** is the practical alternative to distributed transactions. Instead of trying to make multiple services commit atomically, a Saga:

1. Executes a sequence of **local transactions** (one per service)
2. If any step fails, executes **compensating transactions** in reverse order to undo the completed steps

```
Forward Path (happy path):
  Step 1: Reserve inventory     ✅
  Step 2: Charge payment        ✅
  Step 3: Ship order            ❌ Failed!

Compensation Path (rollback):
  Compensate Step 2: Refund payment    ✅
  Compensate Step 1: Release inventory ✅
```

Key characteristics of Sagas:

- **No locks** — Each step commits independently
- **Eventually consistent** — The system may be in an intermediate state during execution
- **Compensation, not rollback** — You can't "undo" a sent email or a shipped package. Compensations are *semantic* undos (refund, cancel, release)
- **Compensation must be idempotent** — Compensations may be retried, so they must be safe to execute multiple times

---

## Part 2: Mental Model — Temporal as a Saga Orchestrator

### Why Temporal Is Perfect for Sagas

Implementing the Saga pattern without Temporal requires:

- A message queue to coordinate steps
- A state machine to track which steps completed
- Persistent storage for compensation state
- Retry logic for each step and each compensation
- Error handling for compensation failures

Temporal provides **all of this out of the box**:

| Saga Requirement | Temporal Feature |
|---|---|
| Coordinate sequential steps | Workflow orchestration |
| Track which steps completed | Event History |
| Persist compensation state | Workflow local state (durable) |
| Retry failed steps | Activity retry policies |
| Handle compensation failures | try/catch in workflow code |
| Survive coordinator crashes | Durable execution |

### The Saga Pattern in Temporal

The pattern in Temporal is elegant:

1. Maintain a **list of compensation functions**
2. Before each activity call, **push its compensation** onto the list
3. If any step fails, **execute compensations in reverse order**
4. Wrap compensations in `CancellationScope.nonCancellable` to ensure they run even if the workflow is cancelled

```
┌─────────────────────────────────────────────────────┐
│  Saga Workflow                                       │
│                                                      │
│  compensations = []                                  │
│                                                      │
│  compensations.push(releaseInventory)                │
│  await reserveInventory()          ✅                │
│                                                      │
│  compensations.push(refundPayment)                   │
│  await chargePayment()             ✅                │
│                                                      │
│  await shipOrder()                 ❌ FAILS          │
│                                                      │
│  catch:                                              │
│    for comp of compensations.reverse():              │
│      await comp()   // refundPayment, releaseInventory│
│                                                      │
└─────────────────────────────────────────────────────┘
```

!!! warning "Register compensation BEFORE the activity call"
    Always push the compensation **before** calling the activity. If the activity succeeds on the remote service but the Worker crashes before recording the result, Temporal will retry the activity. The compensation must already be registered to handle this edge case.

### CancellationScope.nonCancellable

When a workflow is cancelled (e.g., by an operator or another workflow), Temporal propagates the cancellation to all running activities. But during compensation, you **must** complete the cleanup — you can't cancel a refund halfway through.

`CancellationScope.nonCancellable` ensures that the code inside it runs to completion, even if the workflow is cancelled:

```typescript
import { CancellationScope } from '@temporalio/workflow';

// This code runs even if the workflow is cancelled
await CancellationScope.nonCancellable(async () => {
  await refundPayment(order);
  await releaseInventory(order);
});
```

---

## Part 3: Build It — Travel Booking Saga

Let's build a classic Saga example: a travel booking system that reserves a flight, hotel, and car. If any step fails, all previous reservations are cancelled.

### Shared Types

```typescript
// src/shared/types.ts
export interface TripBooking {
  tripId: string;
  userId: string;
  flight: { from: string; to: string; date: string };
  hotel: { city: string; checkIn: string; checkOut: string };
  car: { city: string; pickUp: string; dropOff: string };
}

export interface BookingResult {
  tripId: string;
  flightConfirmation: string;
  hotelConfirmation: string;
  carConfirmation: string;
}
```

### Activity Implementations

```typescript
// src/activities/booking.ts
import { ApplicationFailure, log } from '@temporalio/activity';
import type { TripBooking } from '../shared/types';

// --- Flight Activities ---

export async function bookFlight(
  tripId: string,
  from: string,
  to: string,
  date: string
): Promise<string> {
  log.info('Booking flight', { tripId, from, to, date });
  // Simulate API call to airline
  const confirmation = `FL-${tripId}-${Date.now()}`;
  log.info('Flight booked', { tripId, confirmation });
  return confirmation;
}

export async function cancelFlight(tripId: string, confirmation: string): Promise<void> {
  log.info('Cancelling flight', { tripId, confirmation });
  // Simulate cancellation API call
  // Idempotent: cancelling an already-cancelled flight is a no-op
  log.info('Flight cancelled', { tripId, confirmation });
}

// --- Hotel Activities ---

export async function bookHotel(
  tripId: string,
  city: string,
  checkIn: string,
  checkOut: string
): Promise<string> {
  log.info('Booking hotel', { tripId, city, checkIn, checkOut });
  // Simulate API call to hotel service
  const confirmation = `HT-${tripId}-${Date.now()}`;
  log.info('Hotel booked', { tripId, confirmation });
  return confirmation;
}

export async function cancelHotel(tripId: string, confirmation: string): Promise<void> {
  log.info('Cancelling hotel', { tripId, confirmation });
  // Idempotent cancellation
  log.info('Hotel cancelled', { tripId, confirmation });
}

// --- Car Activities ---

export async function bookCar(
  tripId: string,
  city: string,
  pickUp: string,
  dropOff: string
): Promise<string> {
  log.info('Booking car rental', { tripId, city, pickUp, dropOff });

  // Simulate failure: no cars available in certain cities
  if (city.toLowerCase() === 'tokyo') {
    throw ApplicationFailure.nonRetryable(
      `No cars available in ${city}`
    );
  }

  const confirmation = `CR-${tripId}-${Date.now()}`;
  log.info('Car booked', { tripId, confirmation });
  return confirmation;
}

export async function cancelCar(tripId: string, confirmation: string): Promise<void> {
  log.info('Cancelling car rental', { tripId, confirmation });
  // Idempotent cancellation
  log.info('Car cancelled', { tripId, confirmation });
}

// --- Notification ---

export async function sendBookingConfirmation(
  userId: string,
  tripId: string
): Promise<void> {
  log.info('Sending booking confirmation', { userId, tripId });
  log.info('Confirmation sent', { userId, tripId });
}

export async function sendCancellationNotice(
  userId: string,
  tripId: string,
  reason: string
): Promise<void> {
  log.info('Sending cancellation notice', { userId, tripId, reason });
  log.info('Cancellation notice sent', { userId, tripId });
}
```

Notice:

- Every booking activity has a corresponding **cancel** activity (the compensation)
- Cancel activities are **idempotent** — cancelling twice is safe
- `bookCar` throws `ApplicationFailure.nonRetryable()` for Tokyo — simulating a permanent failure that triggers compensation

### The Saga Workflow

```typescript
// src/workflows/trip-booking.ts
import { proxyActivities, log, CancellationScope } from '@temporalio/workflow';
import type * as bookingActivities from '../activities/booking';
import type { TripBooking, BookingResult } from '../shared/types';

const {
  bookFlight, cancelFlight,
  bookHotel, cancelHotel,
  bookCar, cancelCar,
  sendBookingConfirmation,
  sendCancellationNotice,
} = proxyActivities<typeof bookingActivities>({
  startToCloseTimeout: '1 minute',
  retry: {
    maximumAttempts: 3,
  },
});

export async function tripBookingWorkflow(booking: TripBooking): Promise<BookingResult> {
  log.info('Starting trip booking saga', { tripId: booking.tripId });

  // Compensation stack — functions to undo completed steps
  const compensations: Array<() => Promise<void>> = [];

  try {
    // === Step 1: Book Flight ===
    // Register compensation BEFORE the activity call
    compensations.push(async () => {
      // flightConfirmation is captured in closure when available
      if (flightConfirmation) {
        await cancelFlight(booking.tripId, flightConfirmation);
      }
    });
    var flightConfirmation = await bookFlight(
      booking.tripId,
      booking.flight.from,
      booking.flight.to,
      booking.flight.date
    );
    log.info('Flight booked', { confirmation: flightConfirmation });

    // === Step 2: Book Hotel ===
    compensations.push(async () => {
      if (hotelConfirmation) {
        await cancelHotel(booking.tripId, hotelConfirmation);
      }
    });
    var hotelConfirmation = await bookHotel(
      booking.tripId,
      booking.hotel.city,
      booking.hotel.checkIn,
      booking.hotel.checkOut
    );
    log.info('Hotel booked', { confirmation: hotelConfirmation });

    // === Step 3: Book Car ===
    // No compensation needed for car if it fails — nothing to undo
    compensations.push(async () => {
      if (carConfirmation) {
        await cancelCar(booking.tripId, carConfirmation);
      }
    });
    var carConfirmation = await bookCar(
      booking.tripId,
      booking.car.city,
      booking.car.pickUp,
      booking.car.dropOff
    );
    log.info('Car booked', { confirmation: carConfirmation });

    // === All steps succeeded ===
    await sendBookingConfirmation(booking.userId, booking.tripId);

    return {
      tripId: booking.tripId,
      flightConfirmation,
      hotelConfirmation,
      carConfirmation,
    };

  } catch (err) {
    log.warn('Booking failed, running compensations', {
      tripId: booking.tripId,
      error: String(err),
    });

    // Run compensations in reverse order
    // nonCancellable ensures compensations complete even if workflow is cancelled
    await CancellationScope.nonCancellable(async () => {
      for (const compensate of compensations.reverse()) {
        try {
          await compensate();
        } catch (compErr) {
          // Log but don't fail — best effort compensation
          log.warn('Compensation failed', {
            tripId: booking.tripId,
            error: String(compErr),
          });
        }
      }
    });

    // Notify user of cancellation
    await CancellationScope.nonCancellable(async () => {
      await sendCancellationNotice(
        booking.userId,
        booking.tripId,
        err instanceof Error ? err.message : 'Unknown error'
      );
    });

    throw err; // Re-throw to fail the workflow
  }
}
```

Key patterns in this workflow:

1. **Compensation registered before activity call** — `compensations.push(...)` comes before `await bookFlight(...)`
2. **Reverse order execution** — `compensations.reverse()` ensures the last completed step is undone first
3. **`CancellationScope.nonCancellable`** — Compensations run even if the workflow is cancelled
4. **Best-effort compensation** — Each compensation is wrapped in try/catch. If a compensation fails, we log it and continue with the remaining compensations
5. **Closure captures** — The compensation functions capture confirmation IDs via closures, checking if they exist before attempting cancellation

### Worker

```typescript
// src/worker.ts
import { Worker } from '@temporalio/worker';
import * as bookingActivities from './activities/booking';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows/trip-booking'),
    activities: bookingActivities,
    taskQueue: 'booking-queue',
  });

  console.log('Worker started on task queue: booking-queue');
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
import { tripBookingWorkflow } from './workflows/trip-booking';
import type { TripBooking } from './shared/types';

async function run() {
  const client = new Client();

  // Test 1: Successful booking (all steps succeed)
  console.log('=== Test 1: Successful Trip Booking ===');
  const trip1: TripBooking = {
    tripId: 'trip-001',
    userId: 'user-alice',
    flight: { from: 'SFO', to: 'LHR', date: '2026-06-15' },
    hotel: { city: 'London', checkIn: '2026-06-15', checkOut: '2026-06-20' },
    car: { city: 'London', pickUp: '2026-06-15', dropOff: '2026-06-20' },
  };

  const result1 = await client.workflow.execute(tripBookingWorkflow, {
    workflowId: `booking-${trip1.tripId}`,
    taskQueue: 'booking-queue',
    args: [trip1],
  });
  console.log('Booking result:', JSON.stringify(result1, null, 2));

  // Test 2: Failed booking — car not available in Tokyo
  // Flight and hotel succeed, car fails → compensations run
  console.log('\n=== Test 2: Failed Trip (No Cars in Tokyo) ===');
  const trip2: TripBooking = {
    tripId: 'trip-002',
    userId: 'user-bob',
    flight: { from: 'LAX', to: 'NRT', date: '2026-07-01' },
    hotel: { city: 'Tokyo', checkIn: '2026-07-01', checkOut: '2026-07-05' },
    car: { city: 'Tokyo', pickUp: '2026-07-01', dropOff: '2026-07-05' },
  };

  try {
    await client.workflow.execute(tripBookingWorkflow, {
      workflowId: `booking-${trip2.tripId}`,
      taskQueue: 'booking-queue',
      args: [trip2],
    });
  } catch (err) {
    console.log('Expected failure:', (err as Error).message);
    console.log('(Flight and hotel were compensated — check Web UI)');
  }
}

run().catch(console.error);
```

### Run It and Observe

1. **Start the Temporal dev server:** `temporal server start-dev`
2. **Start the worker:** `npx ts-node src/worker.ts`
3. **Run the client:** `npx ts-node src/client.ts`

**What to observe in the Web UI ([http://localhost:8233](http://localhost:8233)):**

**`booking-trip-001`** (Success):
```
WorkflowExecutionStarted
ActivityTaskScheduled    → bookFlight
ActivityTaskCompleted    → FL-trip-001-...
ActivityTaskScheduled    → bookHotel
ActivityTaskCompleted    → HT-trip-001-...
ActivityTaskScheduled    → bookCar
ActivityTaskCompleted    → CR-trip-001-...
ActivityTaskScheduled    → sendBookingConfirmation
ActivityTaskCompleted
WorkflowExecutionCompleted
```

**`booking-trip-002`** (Saga Compensation):
```
WorkflowExecutionStarted
ActivityTaskScheduled    → bookFlight
ActivityTaskCompleted    → FL-trip-002-...     ✅ Succeeded
ActivityTaskScheduled    → bookHotel
ActivityTaskCompleted    → HT-trip-002-...     ✅ Succeeded
ActivityTaskScheduled    → bookCar
ActivityTaskFailed       → "No cars in Tokyo"  ❌ Failed!
ActivityTaskScheduled    → cancelCar           ↩️ Compensate (no-op)
ActivityTaskCompleted
ActivityTaskScheduled    → cancelHotel         ↩️ Compensate
ActivityTaskCompleted
ActivityTaskScheduled    → cancelFlight        ↩️ Compensate
ActivityTaskCompleted
ActivityTaskScheduled    → sendCancellationNotice
ActivityTaskCompleted
WorkflowExecutionFailed
```

The compensation activities run in **reverse order**: car → hotel → flight. This is the Saga pattern in action.

---

## Part 4: Saga Design Considerations

### Forward Recovery vs Backward Recovery

There are two strategies when a step fails:

| Strategy | Approach | When to Use |
|----------|----------|-------------|
| **Backward recovery** | Undo completed steps (compensate) | When steps are reversible (cancel reservation, refund payment) |
| **Forward recovery** | Retry or use alternatives to complete | When steps are not easily reversible (email sent, physical shipment) |

The travel booking example uses **backward recovery**. In practice, you often combine both:

```typescript
try {
  await shipOrder(order);
} catch (err) {
  // Forward recovery: try alternative shipping provider
  try {
    await shipOrderAlternative(order);
  } catch (altErr) {
    // Backward recovery: can't ship, compensate everything
    await runCompensations();
    throw altErr;
  }
}
```

### Compensation Failures

What happens if a compensation itself fails? This is a real concern:

```typescript
// Best-effort: log and continue
for (const compensate of compensations.reverse()) {
  try {
    await compensate();
  } catch (compErr) {
    log.warn('Compensation failed', { error: compErr });
    // Continue with remaining compensations
    // May need manual intervention later
  }
}
```

In production, you might:

- **Alert operations** when a compensation fails
- **Record the failure** for manual resolution
- **Retry compensations** with their own retry policy

### Parallel Sagas

For independent steps, you can run them in parallel and still compensate correctly:

```typescript
export async function parallelSagaWorkflow(order: Order): Promise<string> {
  const compensations: Array<() => Promise<void>> = [];

  try {
    // Run independent steps in parallel
    compensations.push(() => cancelFlight(order));
    compensations.push(() => cancelHotel(order));

    const [flightId, hotelId] = await Promise.all([
      bookFlight(order),
      bookHotel(order),
    ]);

    // Sequential step that depends on both
    compensations.push(() => cancelCar(order));
    const carId = await bookCar(order);

    return `Booked: ${flightId}, ${hotelId}, ${carId}`;
  } catch (err) {
    await CancellationScope.nonCancellable(async () => {
      for (const compensate of compensations.reverse()) {
        try { await compensate(); } catch (e) { log.warn('Compensation failed', { error: e }); }
      }
    });
    throw err;
  }
}
```

!!! note "Parallel compensation caveat"
    When running steps in parallel with `Promise.all`, if one fails, the others may still be in progress. Temporal handles this correctly — the failed step triggers the catch block, and compensations run for any steps that completed.

---

## Part 5: Experiments

### Experiment 1: Observe Compensation Order

Run the Tokyo trip (Test 2) and carefully examine the Event History in the Web UI. Verify that compensations run in reverse order: car (no-op) → hotel → flight.

### Experiment 2: Kill Worker During Compensation

1. Start a booking for Tokyo (will fail at car step)
2. Kill the Worker immediately when you see "Cancelling hotel" in the logs
3. Restart the Worker
4. Observe that the remaining compensations (cancel flight) still execute

This proves that **compensation is durable** — even if the Worker crashes mid-compensation, Temporal resumes from the Event History.

### Experiment 3: Cancel a Running Workflow

Start a successful booking, but cancel it via CLI before it completes:

```bash
# In one terminal, start a workflow
npx ts-node src/client.ts

# In another terminal, cancel it quickly
temporal workflow cancel --workflow-id booking-trip-001
```

Observe that `CancellationScope.nonCancellable` ensures compensations still run even though the workflow was cancelled.

---

## Key Takeaways

!!! success "What You Learned"

    1. **DDIA Chapter 7** teaches that ACID transactions provide atomicity within a single database, but distributed transactions (2PC) are impractical across microservices — they're slow, fragile, and most services don't support them

    2. **The Saga pattern** is the practical alternative: execute local transactions sequentially, and run compensating transactions in reverse if any step fails

    3. **Temporal is a natural Saga orchestrator** — it provides durable state tracking, automatic retries, and crash recovery. No message queues or state machines needed

    4. **The Saga implementation pattern in Temporal:**
        - Maintain a `compensations` array
        - Push compensation **before** each activity call
        - On failure, execute compensations in **reverse order**
        - Wrap compensations in `CancellationScope.nonCancellable`
        - Catch compensation failures individually (best-effort)

    5. **Compensation activities must be idempotent** — they may be retried due to Worker crashes or network issues

    6. **Forward vs backward recovery** — Use backward recovery (compensate) when steps are reversible. Use forward recovery (retry/alternative) when they're not. Combine both in practice.

---

## Next Module

[:octicons-arrow-right-24: Module 5 — Fault Tolerance](module-05-fault-tolerance.md)
