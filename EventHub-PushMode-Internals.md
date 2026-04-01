# Event Hub Push-Mode: Internal Architecture

## Overview

The push-mode consumer in the Azure Event Hubs C# SDK is implemented via `EventProcessorClient` (`Azure.Messaging.EventHubs.Processor`). From the user's perspective, events are *pushed* to registered async handlers. Internally, this is built on **AMQP credit-based flow control** over a persistent TCP connection, layered on top of `async`/`await` task loops and Azure Blob Storage-backed distributed coordination.

---

## SDK Packages

| Package | Purpose |
|---------|---------|
| `Azure.Messaging.EventHubs` | Core types: `EventData`, `EventHubConsumerClient`, `EventPosition` |
| `Azure.Messaging.EventHubs.Processor` | `EventProcessorClient` — the push-mode processor |
| `Azure.Storage.Blobs` | Checkpoint and partition ownership storage (required) |
| `Azure.Identity` | `DefaultAzureCredential` for authentication |

---

## User-Facing API (How Push Mode Looks from Outside)

```csharp
var processor = new EventProcessorClient(
    storageClient,                                       // BlobContainerClient — for checkpoints/ownership
    EventHubConsumerClient.DefaultConsumerGroupName,
    "<NAMESPACE>.servicebus.windows.net",
    "<HUB_NAME>",
    new DefaultAzureCredential());

// Register handlers — the SDK calls these when events arrive
processor.ProcessEventAsync           += ProcessEventHandler;
processor.ProcessErrorAsync           += ProcessErrorHandler;
processor.PartitionInitializingAsync  += PartitionInitializingHandler;
processor.PartitionClosingAsync       += PartitionClosingHandler;

await processor.StartProcessingAsync();   // returns immediately; processing runs in background
// ...
await processor.StopProcessingAsync();
```

`StartProcessingAsync()` returns as soon as background tasks are launched. From that point on, the SDK owns the loop and calls your handlers whenever events arrive.

---

## Thread Architecture

`StartProcessingAsync()` internally calls:

```csharp
_runningProcessorTask = Task.Factory.StartNew(
    () => RunProcessingAsync(token),
    token,
    TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default
).Unwrap();
```

- `LongRunning` — hints the TPL to allocate a **dedicated OS thread** (not a thread-pool thread), appropriate for indefinitely running loops.
- One additional `LongRunning` thread is created **per owned partition**.

```
StartProcessingAsync()
  └─► LongRunning Thread: RunProcessingAsync()           ← 1 thread (load-balancing loop)
            └─► LongRunning Thread: performProcessing()  ← 1 thread per owned partition
```

For a processor owning 4 partitions → **5 dedicated threads** total.

---

## Layer 1: The Load-Balancing Loop (`RunProcessingAsync`)

The root background task is a `while (!cancelled)` async loop running every **30 seconds** (default `LoadBalancingUpdateInterval`):

```csharp
while (!cancellationToken.IsCancellationRequested)
{
    partitionIds = await ListPartitionIdsAsync(connection, cancellationToken);
    remainingTime = await PerformLoadBalancingAsync(cycleDuration, partitionIds, cancellationToken);
    await Task.Delay(remainingTime, cancellationToken);   // releases thread during sleep
}
```

Each cycle:

1. **Renew** existing ownership records in Blob Storage (before the 2-minute expiry)
2. **List** all ownership records from Blob Storage
3. **Claim or steal** at most one partition per cycle
4. **Start** a new `performProcessing` task for newly claimed partitions
5. **Stop** processors for partitions no longer owned

> **At most one partition is claimed per cycle** — this prevents thundering-herd at scale and enables gradual rebalancing.

### Load-Balancing Algorithm (`PartitionLoadBalancer`)

```
partitionCount     = total partitions (e.g. 8)
activeProcessors   = distinct owners with non-expired ownership (e.g. 3)
minimumOwned       = floor(8 / 3) = 2
maximumOwned       = minimumOwned + 1 = 3

If this instance owns < minimumOwned:
  → Claim an unowned partition (random), OR
  → Steal from a processor that owns > maximumOwned, OR
  → Steal from a processor that owns == maximumOwned (to equalize)
```

**Ownership records** are stored in Azure Blob Storage as ETag-versioned metadata. Blob Storage has a **flat namespace** — there are no real folders. Blob names like `ownership/3` are just strings that contain `/`, so the SDK fetches all records efficiently using a **prefix query**:

```csharp
// BlobCheckpointStoreInternal.ListOwnershipAsync() does essentially this:
string prefix = $"{fullyQualifiedNamespace}/{eventHubName}/{consumerGroup}/ownership";

await foreach (BlobItem blob in containerClient.GetBlobsAsync(
    traits: BlobTraits.Metadata,   // include OwnerIdentifier, LastModifiedTime, etc.
    prefix: prefix))
{
    // each blob is one partition's ownership record
    // blob.Properties.ETag gives the version for optimistic concurrency
}
```

This is a **server-side filter** — only matching blobs are returned over the wire. Results are paginated (default 5000 per page) via `AsyncPageable<BlobItem>`, so listing all ownership records is typically a single `GET /?restype=container&comp=list&prefix=...` HTTP call, not one call per partition. The same prefix pattern is used for checkpoint blobs (under `.../checkpoint/`).

A "steal" is a `ClaimOwnershipAsync()` call that overwrites another processor's ETag — it only succeeds if the **optimistic concurrency check** passes. Whichever instance wins the ETag race gets the partition.

### ETag-Based Optimistic Concurrency (Deep Dive)

Each partition's ownership is a small blob in the checkpoint container at a path like:

```
<container>/<namespace>/<hub>/<consumer-group>/ownership/<partition-id>
```

The blob metadata contains:

| Field | Example Value |
|-------|---------------|
| `OwnerIdentifier` | `"MyProcessor-1"` |
| `PartitionId` | `"3"` |
| `ConsumerGroup` | `"$Default"` |
| `LastModifiedTime` | `2026-04-01T10:00:00Z` |
| **ETag** (auto-managed by Blob Storage) | `"0x8DC4A2B3C4D5E6F"` |

**Example — Stealing a partition:**

Imagine 3 instances (A, B, C) sharing 8 partitions. B currently owns partition 3.

```
Blob: ownership/3
  Metadata:  OwnerIdentifier = "B"
  ETag:      "0x8DC4A2B3C4D5E6F"    ← set when B last renewed
```

1. **A detects imbalance:** A owns 1, B owns 4, C owns 3. `minimumOwned = floor(8/3) = 2`, `maximumOwned = 3`. A is below minimum, B exceeds maximum → A decides to steal partition 3 and remembers the ETag it read.

2. **A writes a conditional update** via `ClaimOwnershipAsync()`:
   ```http
   PUT /ownership/3?comp=metadata
   x-ms-meta-owneridentifier: A          ← "I'm the new owner"
   If-Match: "0x8DC4A2B3C4D5E6F"         ← "Only if nobody changed it since I read it"
   ```

3. **Two possible outcomes:**

   | Scenario | What happened | Result |
   |----------|--------------|--------|
   | **Nobody touched the blob** | ETag still matches → Blob Storage accepts the write, assigns new ETag | ✓ A now owns partition 3 |
   | **B renewed ownership first** | B's renewal changed the ETag → A's `If-Match` is stale | ✗ HTTP 412 Precondition Failed; A retries next cycle |

**Timeline of a failed steal:**

```
t=0s    B renews partition 3             ETag → "...E6F"
t=15s   A reads ownership, sees "...E6F", decides to steal
t=28s   B renews partition 3 again       ETag → "...5F0"  ← changed
t=29s   A writes with If-Match "...E6F"                   ← stale!
        → 412 Precondition Failed                          ← A loses the race
t=30s   A's next LB cycle: re-reads, re-evaluates, maybe retries
```

This is "optimistic" concurrency because there is **no distributed lock** — the protocol assumes conflicts are rare and proceeds without coordination. If two writers race, one wins and the other gets a harmless 412. The worst case is a wasted write attempt, never split-brain where two processors think they own the same partition.

**Greedy mode (default):** With `LoadBalancingStrategy.Greedy` (the default), when unbalanced the cycle delay drops to ~15 ms for rapid claiming until balanced. `Balanced` mode always waits the full 30s.

---

## Layer 2: Per-Partition Processing (`performProcessing`)

When a partition is claimed, `TryStartProcessingPartition()` launches a new `LongRunning` task:

```csharp
async Task performProcessing()
{
    // Phase 1: Initialize — load checkpoint, determine starting position
    var startingPosition = await initializeStartingPositionAsync(partition, ...);

    // Phase 2: Open a dedicated AMQP connection for this partition
    connection = CreateConnection();                                      // one connection per partition
    consumer   = CreateConsumer(ConsumerGroup, partitionId, ...);        // opens ReceivingAmqpLink (prefetch=300)

    // Register cancellation: closing AMQP link unblocks pending ReceiveAsync
    using var reg = cancellationSource.Token.Register(
        state => ((TransportConsumer)state).CloseAsync(CancellationToken.None), consumer);

    while (!cancellationSource.IsCancellationRequested)
    {
        // Blocks until events arrive (drain prefetch buffer) or MaximumWaitTime elapses
        eventBatch = await consumer.ReceiveAsync(maxCount, MaximumWaitTime, token);

        // Sequential dispatch — next ReceiveAsync is NOT called until your handler returns
        await ProcessEventBatchAsync(partition, eventBatch, ...);

        // Track position for consumer restart recovery
        lastEvent = eventBatch.LastOrDefault();
        if (lastEvent != null)
            startingPosition = EventPosition.FromOffset(lastEvent.OffsetString, isInclusive: false);
    }
}
```

Key design decisions:

| Decision | Rationale |
|----------|-----------|
| **Own AMQP connection per partition** | Partition failures are fully isolated |
| **Sequential dispatch within a partition** | Natural backpressure — slow handler stalls the loop, not memory |
| **Up to 1 consumer restart on failure** | Non-cancellation errors restart from last checkpoint; second failure faults the task |

---

## Layer 3: AMQP — The Actual Push Mechanism

`ReceiveAsync` is not polling. The underlying AMQP link uses **credit-based flow control**:

```csharp
var linkSettings = new AmqpLinkSettings
{
    Role             = true,                    // receiver
    TotalLinkCredit  = prefetchCount,           // 300 by default
    AutoSendFlow     = prefetchCount > 0,       // true — auto-replenishes credits
    SettleType       = SettleMode.SettleOnSend, // pre-settled; no ACK round-trip per message
    Source           = new Source {
        Address   = endpoint.AbsolutePath,
        FilterSet = filters                     // stream position filter
    }
};
```

How it works:

1. On link open, the SDK sends an AMQP `FLOW` frame granting **300 credits** to the broker.
2. The broker immediately **streams up to 300 messages** down the persistent TCP connection without waiting for any further requests.
3. As messages are consumed, `AutoSendFlow` automatically sends new `FLOW` frames to replenish credits.
4. This is **true broker-to-client push** — no polling, no request/response per batch.

### `ReceiveAsync` Behavior

```
ReceiveAsync(maxCount, waitTime)
  ├── Prefetch buffer non-empty → return immediately (no network I/O)
  └── Prefetch buffer empty     → await broker push (suspend up to waitTime)
                                    └── First message arrives
                                          → wait up to 20ms for more (batch-building window)
                                          → return whatever arrived
```

If `MaximumWaitTime` is set and nothing arrives within that window, `ReceiveAsync` returns an **empty batch**. The loop then calls `ProcessEventBatchAsync` with no events — this is how "no events" is signaled (useful for heartbeat logic). If `MaximumWaitTime` is `null` (the default), `ReceiveAsync` waits indefinitely and empty batches are never dispatched.

---

## Event Dispatching Pipeline (End to End)

```
[Event Hubs Broker]
   │  AMQP TRANSFER frame (broker pushes on available credit)
   ▼
[ReceivingAmqpLink — local prefetch buffer, 300 events]
   │  consumer.ReceiveAsync(maxCount, 60s)
   │  → drains buffer; suspends if empty
   ▼
[AmqpConsumer]
   │  AMQP messages → List<EventData>  (via AmqpMessageConverter)
   │  DisposeDelivery(msg, AcceptedOutcome)  ← pre-settle each message
   ▼
[EventProcessor<TPartition>.performProcessing inner loop]
   │  await ProcessEventBatchAsync(partition, eventBatch, ...)
   ▼
[EventProcessor<TPartition>.OnProcessingEventBatchAsync]
   │  foreach (EventData eventData in batch)          ← sequential, one at a time
   │  {
   │      args = new ProcessEventArgs(context, eventData, checkpointFunc, token)
   │      await _processEventAsync(args)              ← YOUR ProcessEventAsync handler
   │  }
   ▼
[Your ProcessEventAsync handler]
   │  (optional) await args.UpdateCheckpointAsync()
   ▼
[BlobCheckpointStore.UpdateCheckpointAsync()]
   ▼
[Azure Blob Storage]
```

Additional dispatch notes:

- Your handler runs **inline on the partition's LongRunning thread** — not dispatched to a thread-pool thread.
- Exceptions per event are **caught individually**. If a single exception occurs, it is rethrown directly; if multiple occur, they are rethrown as `AggregateException` after the full batch.
- The error handler (`ProcessErrorAsync`) is invoked via `Task.Run(...)` — **fire-and-forget**, does not block the processing loop.

---

## Stopping (`StopProcessingAsync`)

```
StopProcessingAsync()
  → Cancel CancellationTokenSource
       │
       ├── Cancellation callback fires → CloseAsync(AMQP link)
       │     → unblocks pending ReceiveAsync immediately (TaskCanceledException)
       │     → performProcessing loop exits cleanly
       │
       ├── await Task.WhenAll(all partition stop tasks)
       │     → each: OnPartitionProcessingStoppedAsync()  ← your closing handler
       │
       └── LoadBalancer.RelinquishOwnershipAsync()
             → clears OwnerIdentifier on all ownership blobs (sets to empty string)
             → records remain in Blob Storage as unowned; other processors can claim them
```

The cancellation cascade is: **one `Cancel()` call → AMQP link closes → receive unblocks → loop exits → ownership relinquished** (blobs updated, not deleted).

---

## Configuration Reference

```csharp
var options = new EventProcessorClientOptions
{
    MaximumWaitTime                    = null,                        // default: null (wait indefinitely; no empty-batch heartbeat)
    LoadBalancingUpdateInterval        = TimeSpan.FromSeconds(30),  // LB loop cycle frequency
    PartitionOwnershipExpirationInterval = TimeSpan.FromMinutes(2), // ownership TTL in Blob Storage
    LoadBalancingStrategy              = LoadBalancingStrategy.Greedy, // default; Balanced waits full 30s per cycle
    PrefetchCount                      = 300,                       // AMQP link credit
    Identifier                         = "MyProcessor-1"            // instance name in ownership records
};
```

---

## Architecture Summary

```
StartProcessingAsync()
  │
  └─► [LongRunning Thread] RunProcessingAsync()                 ← load-balancing loop
            │
            │  while(!cancelled) — every 30s
            │  ├── ListPartitionIds()
            │  ├── PerformLoadBalancingAsync()
            │  │       ├── RenewOwnership    → Blob Storage
            │  │       ├── ListOwnership     → Blob Storage
            │  │       ├── FindAndClaimOwnershipAsync()
            │  │       │       └── ClaimOwnership → Blob Storage (ETag race)
            │  │       ├── TryStartProcessingPartition(partitionId)
            │  │       │       └─► [LongRunning Thread] performProcessing()  ← 1 per partition
            │  │       │               ├── own EventHubConnection (AMQP)
            │  │       │               ├── own ReceivingAmqpLink (credit=300, AutoSendFlow)
            │  │       │               └── while(!cancelled)
            │  │       │                     ├── consumer.ReceiveAsync(N, MaximumWaitTime)
            │  │       │                     │     ├── drain prefetch buffer (instant)
            │  │       │                     │     └── await broker AMQP push (≤MaximumWaitTime or indefinite)
            │  │       │                     └── ProcessEventBatchAsync()
            │  │       │                           └── foreach event:
            │  │       │                                 await _processEventAsync(args)  ← YOUR CODE
            │  │       └── StopProcessingPartitionAsync(lost partitions)
            │  └── Task.Delay(30s)
```

### Key Properties

| Property | Detail |
|----------|--------|
| **Push mechanism** | AMQP credit-based flow (`TotalLinkCredit=300`, `AutoSendFlow=true`) — broker streams events |
| **Thread model** | 1 `LongRunning` thread for LB loop + 1 `LongRunning` thread per owned partition |
| **Receive behavior** | Drains local prefetch buffer (instant); if empty, suspends until events arrive (or `MaximumWaitTime` if set) |
| **Handler concurrency** | Sequential per partition; max N concurrent handlers = N owned partitions |
| **Load balancing store** | Azure Blob Storage with ETag-based optimistic concurrency |
| **Ownership TTL** | 2 minutes — must be renewed every 30s cycle or partition can be stolen |
| **Partition stealing** | Win the ETag write race in Blob Storage |
| **Backpressure** | Natural — next `ReceiveAsync` blocked until your handler returns |
| **Error isolation** | Handler exceptions caught per-event; single rethrown directly, multiple as `AggregateException`; infra errors are fire-and-forget to error handler |
| **Cancellation** | Token cancel → AMQP link close → unblocks receive → clean exit |

---

## Comparison: Push Mode vs. Pull Mode

| Aspect | Push Mode (`EventProcessorClient`) | Pull Mode (`EventHubConsumerClient`) |
|--------|------------------------------------|--------------------------------------|
| **Trigger model** | SDK calls your handler | You call `ReceiveAsync()` / `ReadEventsFromPartitionAsync()` |
| **Partition management** | Automatic (Blob Storage ETag coordination) | Manual |
| **Checkpointing** | Built-in `UpdateCheckpointAsync()` | Manual |
| **Load balancing** | Automatic across instances | Developer responsibility |
| **Blob Storage required** | Yes (checkpoints + ownership) | Optional (checkpoints only) |
| **Thread model** | 1 LongRunning thread per partition | Developer-managed |
| **Production suitability** | Recommended | Prototyping / special use cases |
| **AMQP mechanism** | Credit-push, persistent link | Same underneath |
| **Fetch granularity** | Continuous streaming | On-demand, controlled by caller |
