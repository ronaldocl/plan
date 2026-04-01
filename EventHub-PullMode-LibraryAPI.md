# Event Hub Pull-Mode Consumer: Library API Design

## Overview

This document defines the **public contract** for a pull-mode Event Hub consumer library built on top of `EventHubConsumerClient` and `PartitionReceiver`.

The library owns the infrastructure concerns:
- Partition ownership
- ETag-based load balancing
- Ownership renewal and rebalance
- AMQP receiver lifecycle
- Checkpoint persistence

The caller owns the processing window between fetch and checkpoint.

The v1 surface area is intentionally small:

1. **`StartAsync`** initializes ownership, restores checkpoints, and opens receivers.
2. **`FetchAsync`** fetches at most one event per queried partition on demand.
3. **`UpdateCheckpointAsync`** advances the checkpoint only after caller processing is complete.

The library **never auto-checkpoints**.

---

## Canonical Contract

These rules are the source of truth for the implementation and for the internals document:

- Checkpointing is always explicit.
- `FetchAsync` is serialized internally; callers do not need to coordinate concurrent fetch calls.
- Each fetch returns at most one event per queried partition.
- A fetched event is safe to hold across awaits or queue boundaries, but its checkpoint token can become stale after rebalance.
- `UpdateCheckpointAsync` only moves checkpoints forward; it never rewinds them.
- Rebalancing is transparent to fetching, but duplicate delivery remains possible and callers must be idempotent.

---

## Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Explicit checkpointing** | Checkpoint timing is a business decision; the library must not advance it automatically |
| **Caller owns the processing window** | The gap between `FetchAsync` and `UpdateCheckpointAsync` is where user work happens |
| **At-least-once by default** | A crash before checkpoint causes re-delivery from the prior checkpoint |
| **Opaque partition context** | The caller receives a `FetchedEvent` handle instead of composing checkpoint coordinates manually |
| **Non-concurrent fetch** | The library serializes `FetchAsync` to avoid concurrent reads on the same `PartitionReceiver` |
| **Monotonic checkpoints** | A later checkpoint must never be replaced by an older one |
| **Explicit lifecycle** | `StartAsync` / `DisposeAsync` are required; the library does nothing before startup |

---

## Core Types

### `FetchedEvent`

The unit returned by `FetchAsync`. It carries the event data plus the opaque token needed for a later checkpoint attempt.

```csharp
/// <summary>
/// An event fetched from a single Event Hub partition, together with the
/// opaque token required to attempt a checkpoint later.
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

    // Internal: includes partition ID, sequence/offset, and ownership epoch.
    internal CheckpointToken CheckpointToken { get; }
}
```

`FetchedEvent` is **immutable** and **not tied to any live resource**. It is safe to hold across awaits, place in a queue, or pass across threads.

What is *not* guaranteed is that its checkpoint token remains valid forever. A later call to `UpdateCheckpointAsync` can return `false` if:
- the partition was rebalanced away,
- the token belongs to an older ownership epoch, or
- the checkpoint has already advanced to this event or beyond it.

---

### `FetchResult`

The return value of `FetchAsync`. It contains one `FetchedEvent` per queried partition that had an event available within the fetch wait window.

```csharp
/// <summary>
/// The result of a single FetchAsync call.
/// Contains at most one event per queried partition.
/// </summary>
public sealed class FetchResult
{
    /// <summary>
    /// Events received, keyed by partition ID.
    /// Only partitions that produced an event are included.
    /// </summary>
    public IReadOnlyDictionary<string, FetchedEvent> Events { get; }

    /// <summary>
    /// Partitions that were queried but produced no event within FetchWaitTime.
    /// </summary>
    public IReadOnlyCollection<string> EmptyPartitions { get; }

    /// <summary>
    /// Partitions included in this fetch attempt.
    /// QueriedPartitions = Events.Keys ∪ EmptyPartitions.
    /// </summary>
    public IReadOnlyCollection<string> QueriedPartitions { get; }

    /// <summary>True if at least one event was received.</summary>
    public bool HasEvents => Events.Count > 0;
}
```

---

### `IEventHubPullConsumer`

The primary library interface. It is designed to be mockable in unit tests and small enough to use comfortably from a hosted service.

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
    /// restore checkpoints, and open receivers.
    /// Must be called before FetchAsync.
    /// </summary>
    Task StartAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Fetch at most one event from each currently owned partition.
    ///
    /// The library serializes concurrent FetchAsync calls internally.
    /// No checkpoint is advanced automatically.
    /// </summary>
    Task<FetchResult> FetchAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Fetch at most one event from up to <paramref name="maxPartitions"/>
    /// currently owned partitions.
    ///
    /// Partition selection is implementation-defined but must be fair over time
    /// (for example, round-robin or shuffled rotation). Callers must not depend
    /// on a specific selection order.
    ///
    /// If <paramref name="maxPartitions"/> is greater than or equal to the
    /// number of owned partitions, this behaves like the parameterless overload.
    /// No checkpoint is advanced automatically.
    /// </summary>
    Task<FetchResult> FetchAsync(int maxPartitions, CancellationToken cancellationToken = default);

    /// <summary>
    /// Attempt to advance the checkpoint for the partition that produced this event.
    ///
    /// Returns true only when the stored checkpoint moved forward.
    /// Returns false when the token is stale, the partition is no longer owned,
    /// or the checkpoint is already at or beyond this event.
    /// </summary>
    Task<bool> UpdateCheckpointAsync(FetchedEvent fetchedEvent, CancellationToken cancellationToken = default);

    /// <summary>
    /// The partitions currently owned by this instance.
    /// This set changes over time as rebalancing occurs.
    /// </summary>
    IReadOnlyCollection<string> OwnedPartitions { get; }

    /// <summary>
    /// Raised when this instance acquires a partition.
    /// </summary>
    event EventHandler<PartitionOwnershipChangedEventArgs> PartitionAcquired;

    /// <summary>
    /// Raised when this instance loses a partition.
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
    Acquired,
    Stolen,
    Expired,
    Rebalanced,
    Shutdown
}
```

---

## Fetch Semantics

### Parameterless `FetchAsync`

`FetchAsync()` queries every partition currently owned by this instance and returns at most one event per partition.

If a partition has no event available within `FetchWaitTime`, that partition appears in `EmptyPartitions` rather than `Events`.

### `FetchAsync(maxPartitions)`

`FetchAsync(maxPartitions)` queries only a subset of the currently owned partitions.

The library should use a **fair** selection strategy over time. The contract intentionally does not require pure randomness because fairness is the real goal, not a specific RNG choice. A round-robin cursor or shuffled ring is preferred over re-sorting the full owned set on every call.

### Serialization

The library serializes concurrent `FetchAsync` calls internally. This keeps the public API simple and avoids concurrent reads on the same `PartitionReceiver`.

### Prefetch Caveat

Limiting `maxPartitions` limits which receivers are *drained* during a fetch call. It does **not** guarantee that no events are already buffered for non-selected partitions when receivers are long-lived and prefetch is enabled.

For stricter pull fidelity:
- keep `PrefetchCount` low,
- prefer `PrefetchCount = 0` or `1`, and
- treat `maxPartitions` as a scheduling limit, not a hard upstream isolation boundary.

---

## Checkpoint Semantics

The library enforces **at-least-once delivery** by default:

```text
FetchAsync() -> caller processes -> UpdateCheckpointAsync() -> checkpoint advances
```

If the process crashes before checkpoint, the next owner resumes from the prior checkpoint and the event may be delivered again.

### Monotonic Advance

Checkpoints only move forward. `UpdateCheckpointAsync` returns `false` when:
- the stored checkpoint is already at this sequence number or beyond it,
- the `FetchedEvent` belongs to an older ownership epoch, or
- the library can tell the partition is no longer owned by this instance.

This protects the common case where:
- an older queued event is checkpointed after a newer one, or
- a rebalance makes a previously fetched event stale.

### Ownership Validation

The internal checkpoint token includes the partition's **ownership epoch** at fetch time. `UpdateCheckpointAsync` validates that epoch before attempting the checkpoint write.

This blocks obviously stale checkpoint attempts after rebalance, but it is still **not a distributed transaction** across the ownership blob and the checkpoint blob. Idempotent processing is still required.

---

## Usage Pattern

The consumer is typically registered through dependency injection and used from a `BackgroundService`. The library does not dictate the outer polling loop.

```csharp
// In Program.cs / Startup.cs
services.Configure<EventHubPullConsumerOptions>(config.GetSection("EventHubPullConsumer"));
services.AddSingleton<IEventHubPullConsumer, EventHubPullConsumer>();
```

```csharp
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
        _consumer = consumer;
        _processor = processor;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _consumer.StartAsync(stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            FetchResult result = await _consumer.FetchAsync(maxPartitions: 3, stoppingToken);

            if (!result.HasEvents)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(200), stoppingToken);
                continue;
            }

            await Task.WhenAll(result.Events.Values.Select(async fetchedEvent =>
            {
                await _processor.ProcessAsync(fetchedEvent.Data, stoppingToken);

                bool checkpointed = await _consumer.UpdateCheckpointAsync(fetchedEvent, stoppingToken);
                if (!checkpointed)
                {
                    _logger.LogWarning(
                        "Checkpoint skipped for partition {PartitionId}; token was stale or checkpoint had already advanced.",
                        fetchedEvent.PartitionId);
                }
            }));
        }
    }
}
```

---

## Sequence Diagram: Fetch -> Process -> Checkpoint

```text
Caller                    Library                              Azure
  |                          |                                   |
  | StartAsync()             |                                   |
  | -----------------------> | GetPartitionIds() -------------> Event Hubs
  |                          | ClaimOwnership() --------------> Blob Storage
  |                          | RestoreCheckpoints() ----------> Blob Storage
  |                          | OpenReceivers() ---------------> Event Hubs
  | <----------------------- |
  |
  | FetchAsync(maxPartitions?)                                  |
  | -----------------------> | Select owned partitions          |
  |                          | ReceiveBatchAsync() -----------> Event Hubs
  | <----------------------- | FetchResult (no checkpoint)      |
  |
  | [caller processes event] |
  |
  | UpdateCheckpointAsync()  |
  | -----------------------> | Validate ownership epoch         |
  |                          | TryAdvanceCheckpoint() --------> Blob Storage
  | <----------------------- | bool                             |
```

---

## Internal Implementation Sketch

```text
EventHubPullConsumer
  |
  |-- _partitionManager: PartitionOwnershipManager
  |     |-- StartAsync()              -- discover, claim, restore
  |     |-- RenewAllAsync()           -- background renewal with If-Match on ownership ETag
  |     |-- RebalanceAsync()          -- claim unowned / steal from overloaded owners
  |     |-- TryGetOwnership(partitionId)
  |     |-- RelinquishAllAsync()
  |
  |-- _receivers: ConcurrentDictionary<string, PartitionReceiver>
  |     -- one long-lived receiver per owned partition
  |
  |-- _checkpointStore: BlobCheckpointStore
  |     |-- GetCheckpointAsync(partitionId)
  |     |-- TryAdvanceAsync(partitionId, token)
  |
  |-- _fetchGuard: SemaphoreSlim(1, 1)
  |     -- serializes FetchAsync
  |
  |-- FetchAsync(maxPartitions?)
  |     -- snapshot owned partitions
  |     -- choose fair subset if maxPartitions is specified
  |     -- Task.WhenAll(ReceiveBatchAsync(1, FetchWaitTime))
  |     -- build FetchResult + embed ownership epoch into CheckpointToken
  |
  |-- UpdateCheckpointAsync(fetchedEvent)
        -- validate token epoch against current ownership state
        -- reject stale or non-monotonic checkpoint attempts
        -- write checkpoint metadata only if the checkpoint advances
```

---

## `CheckpointToken` (Internal)

The checkpoint token is intentionally opaque to callers.

```csharp
internal sealed class CheckpointToken
{
    internal string PartitionId { get; }
    internal string Offset { get; }
    internal long SequenceNumber { get; }
    internal string OwnershipEpoch { get; }
}
```

`OwnershipEpoch` changes each time a partition changes owners. That gives the library a cheap way to reject checkpoint attempts from a prior ownership incarnation of the same partition.

---

## Configuration

```csharp
public sealed class EventHubPullConsumerOptions
{
    // Event Hub
    public string FullyQualifiedNamespace { get; set; }   // "<ns>.servicebus.windows.net"
    public string EventHubName            { get; set; }
    public string ConsumerGroup           { get; set; } = "$Default";
    public TokenCredential Credential     { get; set; }   // DefaultAzureCredential recommended

    // Blob Storage
    public string BlobConnectionString    { get; set; }
    public string BlobContainerName       { get; set; } = "eventhub-pullmode";

    // Ownership management
    public TimeSpan OwnershipExpirationInterval { get; set; } = TimeSpan.FromMinutes(2);
    public TimeSpan LoadBalancingUpdateInterval { get; set; } = TimeSpan.FromSeconds(30);
    public LoadBalancingStrategy LoadBalancingStrategy { get; set; } = LoadBalancingStrategy.Greedy;

    // Fetch behavior
    public TimeSpan FetchWaitTime         { get; set; } = TimeSpan.FromMilliseconds(500);
    public int PrefetchCount              { get; set; } = 1;

    // Startup
    public EventPosition DefaultStartingPosition { get; set; } = EventPosition.Earliest;
    public string InstanceId              { get; set; } = $"{Environment.MachineName}-{Guid.NewGuid():N}";
}
```

Notes:
- `PrefetchCount` should stay low for pull-style semantics.
- `MaxEventsPerFetch` is intentionally omitted from v1 because the public API returns at most one event per queried partition.
- `IdleDelay` is caller policy and stays outside the library contract.

---

## Related Documents

- [EventHub-PullMode-Internals.md](./EventHub-PullMode-Internals.md) — Implementation details: AMQP links, ownership renewal, rebalance, checkpoint persistence
