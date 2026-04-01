# Event Hub Pull-Mode Consumer: Library API Design

## Overview

This document designs a library that wraps the pull-mode Event Hub consumer internals into a clean, user-facing API. The library handles all infrastructure concerns (partition ownership, ETag-based load balancing, rebalancing, AMQP connections) and exposes two simple operations to the caller:

1. **`FetchAsync`** — fetch one event per owned partition on demand, optionally limited to a random subset of partitions
2. **`UpdateCheckpointAsync`** — advance the checkpoint for a partition after the caller has finished processing

The caller is responsible for the work that happens between these two calls. The library never auto-checkpoints.

---

## Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Explicit checkpointing** | Checkpoint timing is a business decision — the library must not advance it automatically |
| **Caller owns the processing window** | The gap between `FetchAsync` and `UpdateCheckpointAsync` is where user work happens; the library must not interfere |
| **At-least-once by default** | Checkpoint only after successful processing; a crash before `UpdateCheckpointAsync` causes the event to be re-delivered on next startup |
| **Partition context is opaque** | The caller receives a `FetchedEvent` handle — not a raw partition ID string — to prevent misuse (e.g. checkpointing the wrong partition) |
| **Non-concurrent fetch** | The library serializes concurrent `FetchAsync` calls internally; callers do not need to worry about it |
| **Lifecycle is explicit** | `StartAsync` / `DisposeAsync` are explicit; the library does nothing until started |

---

## Core Types

### `FetchedEvent`

The unit returned by `FetchAsync`. Carries the event data and the opaque context needed to checkpoint it. The caller must not inspect or store partition internals — only pass this back to `UpdateCheckpointAsync`.

```csharp
/// <summary>
/// An event fetched from a single Event Hub partition, along with the
/// context required to checkpoint it after processing.
/// </summary>
public sealed class FetchedEvent
{
    /// <summary>The partition this event came from.</summary>
    public string PartitionId { get; }

    /// <summary>The event payload and metadata.</summary>
    public EventData Data { get; }

    /// <summary>Sequence number of this event within its partition.</summary>
    public long SequenceNumber { get; }

    /// <summary>Offset of this event within its partition.</summary>
    public string Offset { get; }

    /// <summary>UTC time the event was enqueued in Event Hubs.</summary>
    public DateTimeOffset EnqueuedTime { get; }

    // Internal: opaque checkpoint token; not exposed to caller
    internal CheckpointToken CheckpointToken { get; }
}
```

`FetchedEvent` is **immutable** and **not tied to any live resource** — it is safe to hold across awaits, store in a queue, or pass across threads.

---

### `FetchResult`

The return value of `FetchAsync`. Contains one `FetchedEvent` per partition that had an available event. Partitions with no events are omitted.

```csharp
/// <summary>
/// The result of a single FetchAsync call.
/// Contains at most one event per queried partition.
/// </summary>
public sealed class FetchResult
{
    /// <summary>
    /// Events received, keyed by partition ID.
    /// Only partitions that had an available event are included.
    /// </summary>
    public IReadOnlyDictionary<string, FetchedEvent> Events { get; }

    /// <summary>
    /// Partitions that were queried but had no available event.
    /// When using FetchAsync(maxPartitions), only the selected partitions
    /// appear here — not all owned partitions.
    /// </summary>
    public IReadOnlyCollection<string> EmptyPartitions { get; }

    /// <summary>
    /// Partitions that were queried in this fetch (selected subset or all owned).
    /// QueriedPartitions = Events.Keys ∪ EmptyPartitions.
    /// </summary>
    public IReadOnlyCollection<string> QueriedPartitions { get; }

    /// <summary>True if at least one event was received.</summary>
    public bool HasEvents => Events.Count > 0;
}
```

---

### `IEventHubPullConsumer`

The primary library interface. Designed to be easily mockable in unit tests.

```csharp
/// <summary>
/// Pull-mode Event Hub consumer. Manages partition ownership, ownership renewal,
/// and rebalancing automatically. The caller controls when events are fetched
/// and when checkpoints are advanced.
/// </summary>
public interface IEventHubPullConsumer : IAsyncDisposable
{
    /// <summary>
    /// Initialize the consumer: discover partitions, claim ownership,
    /// restore checkpoints, open AMQP receivers.
    /// Must be called before FetchAsync.
    /// </summary>
    Task StartAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Fetch one event from each owned partition concurrently.
    /// Returns immediately with whatever is available; does not block
    /// until all partitions have events.
    ///
    /// Only partitions that had an available event within FetchWaitTime
    /// are included in FetchResult.Events.
    ///
    /// Does NOT advance any checkpoint — the caller must call
    /// UpdateCheckpointAsync after processing.
    /// </summary>
    Task<FetchResult> FetchAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Fetch one event from up to <paramref name="maxPartitions"/> randomly
    /// selected owned partitions concurrently.
    ///
    /// If <paramref name="maxPartitions"/> >= owned partition count, this
    /// behaves identically to the parameterless overload (fetches from all).
    ///
    /// Partition selection is randomized on each call to distribute load
    /// evenly across partitions over time, avoiding starvation.
    ///
    /// Does NOT advance any checkpoint — the caller must call
    /// UpdateCheckpointAsync after processing.
    /// </summary>
    /// <param name="maxPartitions">
    /// Maximum number of owned partitions to fetch from. Partitions are
    /// selected randomly from the current owned set.
    /// </param>
    Task<FetchResult> FetchAsync(
        int maxPartitions,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Advance the checkpoint for the partition that produced this event.
    /// Call this after all processing on the event is complete and durable.
    ///
    /// If this instance no longer owns the partition (ownership lost), the call
    /// is a no-op and returns false.
    /// </summary>
    /// <param name="fetchedEvent">The event returned by a previous FetchAsync call.</param>
    /// <returns>True if the checkpoint was written; false if the partition is no longer owned.</returns>
    Task<bool> UpdateCheckpointAsync(
        FetchedEvent fetchedEvent,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// The set of partition IDs currently owned by this instance.
    /// This set changes over time as rebalancing occurs.
    /// </summary>
    IReadOnlyCollection<string> OwnedPartitions { get; }

    /// <summary>
    /// Raised when this instance acquires ownership of a new partition.
    /// </summary>
    event EventHandler<PartitionOwnershipChangedEventArgs> PartitionAcquired;

    /// <summary>
    /// Raised when this instance loses ownership of a partition
    /// (stolen by another instance, ownership expiry, or graceful release).
    /// </summary>
    event EventHandler<PartitionOwnershipChangedEventArgs> PartitionLost;
}
```

---

### Supporting Types

```csharp
public sealed class PartitionOwnershipChangedEventArgs : EventArgs
{
    public string PartitionId { get; }
    public PartitionOwnershipChangeReason Reason { get; }
}

public enum PartitionOwnershipChangeReason
{
    Acquired,              // acquired on startup or rebalance
    Stolen,                // another instance stole via ETag race (412 on renewal)
    OwnershipExpired,      // ownership record expired (instance failed to renew in time)
    Rebalanced,            // actively given up to balance across instances
    Shutdown               // released during graceful DisposeAsync
}
```

---

## Usage Pattern

The consumer is registered through dependency injection and injected into a `BackgroundService`. The library does not dictate the outer loop structure:

```csharp
// In Program.cs / Startup.cs — register the consumer
services.Configure<EventHubPullConsumerOptions>(config.GetSection("EventHubPullConsumer"));
services.AddSingleton<IEventHubPullConsumer, EventHubPullConsumer>();
```

```csharp
// In the hosted service — consume via constructor injection
public class EventProcessingWorker : BackgroundService
{
    private readonly IEventHubPullConsumer _consumer;
    private readonly IMyProcessor _processor;
    private readonly ILogger<EventProcessingWorker> _logger;

    public EventProcessingWorker(
        IEventHubPullConsumer consumer,
        IMyProcessor processor,
        ILogger<EventProcessingWorker> logger)
    {
        _consumer  = consumer;
        _processor = processor;
        _logger    = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _consumer.StartAsync(stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            await FetchAndProcessAsync(stoppingToken);
        }
    }

    private async Task FetchAndProcessAsync(CancellationToken stoppingToken)
    {
        // Fetch from up to 3 randomly selected partitions per cycle
        // Use parameterless FetchAsync() to fetch from all owned partitions
        FetchResult result = await _consumer.FetchAsync(maxPartitions: 3, stoppingToken);

        if (!result.HasEvents)
        {
            // No events available on any owned partition right now
            await Task.Delay(TimeSpan.FromMilliseconds(200), stoppingToken);
            return;
        }

        await Task.WhenAll(result.Events.Values.Select(async fetchedEvent =>
        {
            // ── User does work here ──────────────────────────────────
            await _processor.ProcessAsync(fetchedEvent.Data, stoppingToken);

            // Only checkpoint after work is durably complete
            bool checkpointed = await _consumer.UpdateCheckpointAsync(fetchedEvent, stoppingToken);

            if (!checkpointed)
            {
                // Partition was lost between fetch and checkpoint — event may
                // be re-processed by another instance. Handle idempotency.
                _logger.LogWarning("Lost partition {Id} before checkpoint", fetchedEvent.PartitionId);
            }
        }));
    }
}


```

---

## Sequence Diagram: Fetch → Process → Checkpoint

```
Caller                    Library (IEventHubPullConsumer)           Azure
  │                                │                                   │
  │  StartAsync()                  │                                   │
  │ ─────────────────────────────► │  GetPartitionIds()  ──────────► Event Hubs
  │                                │  ClaimOwnership()   ──────────► Blob Storage (conditional PUT)
  │                                │  RestoreCheckpoints()─────────► Blob Storage
  │                                │  OpenAmqpReceivers()──────────► Event Hubs (AMQP)
  │ ◄───────────────────────────── │
  │
  │  FetchAsync(maxPartitions?)      │
  │ ─────────────────────────────► │
  │                                │  Select partitions (all, or random k from owned set)
  │                                │  ReceiveBatchAsync() (per selected partition, Task.WhenAll)
  │                                │ ──────────────────────────────► AMQP prefetch buffer
  │                                │ ◄──────────────────────────────
  │ ◄── FetchResult ────────────── │  (checkpoint NOT written)
  │
  │  [caller does work on events]  │
  │
  │  UpdateCheckpointAsync(event)  │
  │ ─────────────────────────────► │
  │                                │  Verify still own partition (ownership check)
  │                                │  SetMetadataAsync(offset, seqNo) ─► Blob Storage
  │ ◄── bool (true/false) ──────── │
```

---

## Checkpoint Semantics: At-Least-Once

The library enforces **at-least-once delivery** by design:

```
Timeline A — happy path:
  FetchAsync() → [work] → UpdateCheckpointAsync() → ✅ processed exactly once

Timeline B — crash before checkpoint:
  FetchAsync() → [work] → CRASH
  (next startup) → RestoreCheckpoint() → returns offset BEFORE this event
  → FetchAsync() re-delivers the same event → caller must be idempotent

Timeline C — ownership lost before checkpoint:
  FetchAsync() → [work] → UpdateCheckpointAsync() → returns false (ownership gone)
  Another instance has already taken this partition from its last checkpoint
  → event will be re-processed by the other instance → caller must be idempotent
```

> **The caller is responsible for idempotency.** The library guarantees at-least-once; exactly-once requires idempotent processing logic in the caller.

---

## Internal Implementation Sketch

```
EventHubPullConsumer
  │
  ├── _partitionManager: PartitionOwnershipManager
  │       ├── StartAsync()               — scan, claim ownership, restore
  │       ├── RenewAllAsync()            — background timer, every 30s (conditional PUT)
  │       ├── RebalanceAsync()           — runs within renewal cycle (steal/relinquish)
  │       ├── IsOwned(partitionId)       — ownership health check before checkpoint
  │       └── RelinquishAllAsync()       — graceful shutdown (clear OwnerIdentifier)
  │
  ├── _receivers: ConcurrentDictionary<string, PartitionReceiver>
  │       — one PartitionReceiver (AMQP link) per owned partition
  │       — added on acquire, removed on release/loss
  │
  ├── _checkpointStore: BlobCheckpointStore
  │       ├── GetCheckpointAsync(partitionId)
  │       └── UpdateCheckpointAsync(partitionId, offset, sequenceNumber)
  │
  ├── _fetchGuard: SemaphoreSlim(1, 1)
  │       — serializes concurrent FetchAsync calls
  │
  └── FetchAsync(maxPartitions?)
          → _fetchGuard.WaitAsync()
          → partitions = maxPartitions == null
              ? _receivers.Keys                                    // all owned
              : _receivers.Keys.OrderBy(_ => Random.Shared.Next()) // random subset
                               .Take(maxPartitions.Value)
          → Task.WhenAll(partitions.Select(p => _receivers[p].ReceiveBatchAsync(1, FetchWaitTime)))
          → build FetchResult (attach CheckpointToken per event, track QueriedPartitions)
          → _fetchGuard.Release()

  UpdateCheckpointAsync(fetchedEvent)
          → _partitionManager.IsOwned(fetchedEvent.PartitionId)  → false: return false
          → _checkpointStore.UpdateCheckpointAsync(...)           → write blob metadata
          → return true
```

---

## `CheckpointToken` (Internal)

The `FetchedEvent.CheckpointToken` is an internal type that carries everything the library needs to write the checkpoint, without exposing blob storage details to the caller:

```csharp
internal sealed class CheckpointToken
{
    internal string PartitionId     { get; }
    internal string Offset          { get; }
    internal long   SequenceNumber  { get; }
    // No BlobClient reference — the store is resolved at checkpoint time
    // via the library's own BlobCheckpointStore, not from this token
}
```

This keeps `FetchedEvent` a simple, serializable, dependency-free DTO.

---

## Configuration

```csharp
public sealed class EventHubPullConsumerOptions
{
    // ── Event Hub ──────────────────────────────────────────────────────
    public string            FullyQualifiedNamespace   { get; set; }  // "<ns>.servicebus.windows.net"
    public string            EventHubName              { get; set; }
    public string            ConsumerGroup             { get; set; } = "$Default";
    public TokenCredential   Credential                { get; set; }  // DefaultAzureCredential recommended

    // ── Blob Storage ───────────────────────────────────────────────────
    public string            BlobConnectionString      { get; set; }
    public string            BlobContainerName         { get; set; } = "eventhub-pullmode";

    // ── Ownership management ────────────────────────────────────────
    public TimeSpan          OwnershipExpirationInterval { get; set; } = TimeSpan.FromMinutes(2);
    public TimeSpan          LoadBalancingUpdateInterval { get; set; } = TimeSpan.FromSeconds(30);
    public LoadBalancingStrategy LoadBalancingStrategy    { get; set; } = LoadBalancingStrategy.Greedy;

    // ── Fetch behaviour ────────────────────────────────────────────────
    public int               MaxEventsPerFetch         { get; set; } = 1;
    public TimeSpan          FetchWaitTime             { get; set; } = TimeSpan.FromMilliseconds(500);
    public int               PrefetchCount             { get; set; } = 1;
    public TimeSpan          IdleDelay                 { get; set; } = TimeSpan.FromMilliseconds(200);

    // ── Startup ────────────────────────────────────────────────────────
    public EventPosition     DefaultStartingPosition   { get; set; } = EventPosition.Earliest;
    public string            InstanceId                { get; set; } = $"{Environment.MachineName}-{Guid.NewGuid():N}";
}
```

---

## Dependency Injection Registration

The registration shown in the usage pattern above binds options from configuration. For inline options or custom credential setup:

```csharp
// Inline options (no IConfiguration binding)
services.AddSingleton<IEventHubPullConsumer>(_ =>
    new EventHubPullConsumer(new EventHubPullConsumerOptions
    {
        FullyQualifiedNamespace = "myns.servicebus.windows.net",
        EventHubName            = "my-hub",
        Credential              = new DefaultAzureCredential(),
        BlobConnectionString    = "DefaultEndpointsProtocol=https;...",
        BlobContainerName       = "eventhub-pullmode"
    }));
```

The `BackgroundService` pattern shown in the usage pattern above is the recommended hosting approach. The host manages the service lifetime — `DisposeAsync` on `IEventHubPullConsumer` is called when the host shuts down, relinquishing all ownership gracefully.

---

## Related Documents

- [EventHub-PullMode-Internals.md](./EventHub-PullMode-Internals.md) — Internal architecture: AMQP, ETag-based ownership, rebalancing, checkpoint storage