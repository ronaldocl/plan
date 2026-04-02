# Event Hub Pull-Mode: Internal Architecture & Design

## Overview

Pull mode inverts the usual Event Hubs control flow: **your code decides when to fetch**, rather than the SDK continuously invoking a handler.

The low-level SDK primitive is `EventHubConsumerClient` from `Azure.Messaging.EventHubs`. Unlike `EventProcessorClient`, it does **not** provide:
- partition ownership,
- distributed load balancing,
- automatic checkpointing, or
- a background processing loop.

Those concerns are implemented by the library described in [EventHub-PullMode-LibraryAPI.md](./EventHub-PullMode-LibraryAPI.md). That API document is the **public contract**. This document explains the internal model behind it.

This design assumes:
- explicit checkpointing,
- serialized `FetchAsync` (if multiple callers invoke it at once, one runs and the others wait),
- at most one event per queried partition per fetch call, and
- ETag-based partition ownership backed by Blob Storage.

---

## SDK Packages

| Package | Purpose |
|---------|---------|
| `Azure.Messaging.EventHubs` | `EventHubConsumerClient`, `PartitionReceiver`, `PartitionProperties`, `EventData`, `EventPosition` |
| `Azure.Storage.Blobs` | Ownership and checkpoint persistence |
| `Azure.Identity` | `DefaultAzureCredential` for authentication |

No `Azure.Messaging.EventHubs.Processor` package is needed because pull mode deliberately avoids `EventProcessorClient`.

---

## Pull Mode vs. Push Mode

| Aspect | Push Mode (`EventProcessorClient`) | Pull Mode (library on top of `EventHubConsumerClient`) |
|--------|------------------------------------|--------------------------------------------------------|
| **Trigger model** | SDK drives handler invocation | Caller invokes `FetchAsync` on demand |
| **Partition ownership** | Built in | Library-managed |
| **Checkpointing** | Built in | Explicit `UpdateCheckpointAsync(FetchedEvent)` |
| **Load balancing** | Built in | Library-managed with Blob Storage ETags |
| **Fetch granularity** | Continuous stream | At most one event per queried partition |
| **Backpressure** | Natural in handler loop | Explicit; caller controls polling cadence |
| **Thread model** | Background loops per owned partition | One receiver per owned partition, fetch loop caller-driven |

---

## Reader Primitives

`EventHubConsumerClient` exposes two useful receive styles:

### `ReadEventsFromPartitionAsync`

```csharp
await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsFromPartitionAsync(
    partitionId,
    startingPosition,
    cancellationToken))
{
    EventData data = partitionEvent.Data;
}
```

This is a long-lived async iterator. Internally it keeps an AMQP receive link open and yields events as they become available.

### `PartitionReceiver.ReceiveBatchAsync`

The pull-mode library uses a dedicated `PartitionReceiver` per owned partition:

```csharp
var receiver = new PartitionReceiver(
    consumerGroup,
    partitionId,
    startingPosition,
    fullyQualifiedNamespace,
    eventHubName,
    credential,
    new PartitionReceiverOptions { PrefetchCount = 1 });

IEnumerable<EventData> batch = await receiver.ReceiveBatchAsync(
    maxMessages: 1,
    maxWaitTime: TimeSpan.FromMilliseconds(500),
    cancellationToken);
```

`ReceiveBatchAsync` is a better fit for the library contract because it maps naturally to:
- one fetch request,
- one partition-local receive operation,
- a bounded wait time, and
- at most one returned event per partition.

---

## AMQP Mechanics

### Long-Lived Receive Links

Each `PartitionReceiver` opens its own AMQP receive link for a single partition. The library keeps that link open while the partition is owned.

The internal shape is:

```text
Pull consumer instance
  -> one AMQP connection to Event Hubs
  -> one PartitionReceiver per owned partition
  -> one receive link per PartitionReceiver
```

This keeps partition state local and avoids reopening links on every fetch.

### Credit and Prefetch

Under the hood, Event Hubs still uses credit-based AMQP push. Events can be pushed into a local prefetch buffer while credits are available.

At a high level:

```text
ReceiveBatchAsync(maxMessages: 1, maxWaitTime: 500ms)
  -> if local buffer already has an event, return immediately
  -> otherwise wait up to maxWaitTime for broker delivery
  -> return empty if nothing arrives
```

This has an important consequence:

> Even in a "pull" API, open receivers plus prefetch mean the broker may already have delivered events into local buffers before the caller asks for them.

For that reason:
- `PrefetchCount` should stay low for pull-like behavior,
- `PrefetchCount = 0` or `1` is the most faithful configuration, and
- `FetchAsync(maxPartitions)` should be treated as a **receiver-drain scheduling limit**, not proof that non-selected partitions have no buffered events.

### Why `FetchAsync` Is Serialized

`PartitionReceiver` is not intended to be read concurrently for the same partition. The library therefore guards `FetchAsync` with a `SemaphoreSlim(1, 1)` and performs one batch receive per selected partition inside that critical section.

In simple terms: only one `FetchAsync` call is actively draining receivers at a time.

That keeps the caller contract simple:
- concurrent fetch calls are allowed,
- the library serializes them, and
- each receiver sees one in-flight read at a time.

---

## Distributed Partition Ownership

`EventHubConsumerClient` has no notion of multi-instance ownership. In a distributed deployment, the library must coordinate who owns each partition before reading from it.

The chosen design uses **Blob Storage ETag-based optimistic concurrency**, matching the same general approach used by `EventProcessorClient`.

### Why ETag Ownership

This design is chosen over Blob leases because it better fits:
- fast scale-up convergence,
- region-by-region rolling deploys,
- cross-region Blob Storage latency tolerance, and
- a load-balancer that can actively steal from overloaded owners.

The trade-off is that a partition can briefly be processed by both the old and new owner during a steal. Callers must therefore be idempotent.

---

## Ownership Record Model

Each partition has an ownership blob under a stable path:

```text
{container}/{namespace}/{hub}/{consumer-group}/ownership/{partitionId}
```

The blob body can be empty. Ownership state lives in metadata plus Blob service properties:

| Field | Source | Purpose |
|-------|--------|---------|
| `OwnerIdentifier` | Metadata | Current owner instance ID |
| `OwnershipEpoch` | Metadata | Changes each time ownership transfers to a new instance |
| `PartitionId` | Metadata | Human-readable debugging aid |
| `ConsumerGroup` | Metadata | Human-readable debugging aid |
| `ETag` | Blob property | Optimistic concurrency token |
| `LastModified` | Blob property | Authoritative ownership freshness timestamp |

### Why `LastModified` Instead of Custom Timestamp Metadata

Blob Storage already updates `LastModified` whenever metadata changes. Using the service timestamp is simpler and avoids cross-region clock-skew issues that come with storing a custom `LastModifiedTime` string in metadata.

### Ownership Epoch

`OwnershipEpoch` is a library-generated identifier, typically a GUID.

It changes when:
- this instance first claims an unowned partition,
- this instance steals a partition from another owner,
- any other instance steals a partition from this instance and later a new owner claims it.

It stays the same during renewal.

That gives the library a lightweight way to distinguish:
- "same partition, same owner incarnation"
from
- "same partition, but a different ownership generation".

---

## Correct Conditional Write Semantics

All ownership transitions are implemented with conditional metadata updates on the ownership blob.

| Operation | Condition |
|-----------|-----------|
| Ensure blob exists | `If-None-Match: *` on create |
| Claim existing unowned/expired partition | `If-Match: <etag read during list>` |
| Steal from another owner | `If-Match: <etag read during list>` |
| Renew owned partition | `If-Match: <current owned etag>` |
| Relinquish on shutdown | `If-Match: <current owned etag>` |

The key refinement is this:

> Do **not** use `If-Match: *` for normal claims. It only verifies that the blob exists; it does not protect against another writer changing the record after you read it.

Using the exact ETag from the last ownership scan preserves the optimistic-concurrency guarantee:
- two stealers racing on the same ETag cannot both win,
- a renewal that loses the race gets HTTP 412,
- ownership changes are linearized by Blob Storage versioning.

---

## Ownership Expiry

Ownership expiry is enforced by application logic, not by the storage service:

```text
expired = UtcNow - blob.Properties.LastModified > OwnershipExpirationInterval
```

If a record is expired, any instance may try to claim it by writing metadata with `If-Match` against the last observed ETag.

This is more tolerant than Blob leases in a cross-region deployment because:
- renewals happen every `LoadBalancingUpdateInterval`, and
- expiry is typically much larger than one missed renewal.

With defaults:
- renewal interval: 30 seconds
- expiration interval: 2 minutes

that leaves buffer for GC pauses, transient latency spikes, or brief regional instability.

---

## Startup Flow

Startup is deterministic and ownership-first:

```text
StartAsync()
  1. GetPartitionIdsAsync()
  2. EnsureOwnershipBlobsExistAsync()
  3. ListOwnershipAsync()
  4. Calculate target ownership counts
  5. Claim up to minimumOwned partitions
  6. Restore checkpoints for newly owned partitions
  7. Open PartitionReceiver instances
```

### 1. Discover Partitions

The library asks Event Hubs for the partition IDs:

```csharp
string[] partitionIds = await consumerClient.GetPartitionIdsAsync(cancellationToken);
```

### 2. Ensure Ownership Blobs Exist

Each partition should have a stable ownership blob. Instances may race to create them:

```text
Create blob with If-None-Match: *
  -> first creator succeeds
  -> concurrent creators get 409 / precondition failure and move on
```

### 3. Scan Ownership State

The library lists ownership blobs under the prefix for the namespace / hub / consumer group and reads:
- metadata,
- ETag,
- `LastModified`.

Each partition is categorized as:
- owned by self and not expired,
- owned by another instance and not expired,
- unowned, or
- expired.

### 4. Compute Fair Share

Using the non-expired owners discovered in the scan:

```text
activeProcessors = distinct non-expired OwnerIdentifier values plus self if absent
minimumOwned     = floor(totalPartitions / activeProcessors)
maximumOwned     = minimumOwned + 1
```

### 5. Claim Ownership

For each unowned or expired partition the instance wants to claim:

```text
Set metadata:
  OwnerIdentifier = this.InstanceId
  OwnershipEpoch  = new GUID
  PartitionId     = partitionId
  ConsumerGroup   = consumerGroup

Write with If-Match: observed ETag
```

Outcomes:
- success -> this instance owns the partition and records the new ETag plus epoch locally
- `412 Precondition Failed` -> another writer won the race; retry on the next load-balancing cycle

### 6. Restore Checkpoints

For each newly owned partition:

```text
read checkpoint blob metadata
  -> if present: EventPosition.FromOffset(offset, isInclusive: false)
  -> else:       DefaultStartingPosition
```

### 7. Open Receivers

The library creates one `PartitionReceiver` per owned partition and stores it in a concurrent dictionary keyed by partition ID.

---

## On-Demand Fetch

The public API is `FetchAsync`, not direct access to `PartitionReceiver`.

Internally the flow is:

```text
FetchAsync(maxPartitions?)
  -> wait on _fetchGuard
  -> snapshot currently owned receivers
  -> choose all partitions, or a fair subset
  -> Task.WhenAll(ReceiveBatchAsync(1, FetchWaitTime)) across selected partitions
  -> build FetchResult
  -> release _fetchGuard
```

### Fair Partition Selection

When `maxPartitions` is specified, the library should use a fairness-oriented selector such as:
- round-robin cursor over the owned set, or
- shuffled rotation refreshed occasionally.

The important property is **eventual coverage**, not pure randomness. Re-sorting every call with `OrderBy(_ => Random.Shared.Next())` is needlessly expensive and makes behavior harder to reason about.

### Fetch Output Shape

For each selected partition:
- if one event is available within `FetchWaitTime`, create a `FetchedEvent`
- otherwise add the partition to `EmptyPartitions`

Each `FetchedEvent` includes an internal `CheckpointToken` containing:
- partition ID,
- sequence number,
- offset,
- ownership epoch captured at fetch time

The library does **not** auto-checkpoint here.

### Example Sketch

```csharp
public async Task<FetchResult> FetchAsync(
    int? maxPartitions = null,
    CancellationToken cancellationToken = default)
{
    await _fetchGuard.WaitAsync(cancellationToken);
    try
    {
        var selected = SelectPartitionsFairly(_receivers.Keys, maxPartitions);

        var tasks = selected.Select(async partitionId =>
        {
            PartitionReceiver receiver = _receivers[partitionId];
            IEnumerable<EventData> batch = await receiver.ReceiveBatchAsync(
                maxMessages: 1,
                maxWaitTime: _options.FetchWaitTime,
                cancellationToken);

            EventData? eventData = batch.FirstOrDefault();
            return (partitionId, eventData);
        });

        var results = await Task.WhenAll(tasks);
        return BuildFetchResult(results, _ownershipStateSnapshot);
    }
    finally
    {
        _fetchGuard.Release();
    }
}
```

---

## Checkpoint Management

Checkpoints are stored separately from ownership records:

```text
{container}/{namespace}/{hub}/{consumer-group}/checkpoint/{partitionId}
```

Metadata typically includes:

| Field | Purpose |
|-------|---------|
| `Offset` | Resume position |
| `SequenceNumber` | Monotonic progress marker |
| `UpdatedAt` | Human-readable diagnostics |

### Checkpoint Invariants

The library should enforce these invariants:

1. A checkpoint only advances; it never moves backward.
2. A checkpoint token from an older ownership epoch is rejected.
3. The library attempts an ownership validation before writing.

### Update Flow

```text
UpdateCheckpointAsync(fetchedEvent)
  1. Read token from fetchedEvent
  2. Validate current ownership state for that partition
       -> owner must still be self
       -> ownership epoch must still match token
  3. Read current checkpoint
       -> if current sequence number >= token sequence number: return false
  4. Write checkpoint metadata
       -> use checkpoint blob ETag to protect against concurrent writers on the checkpoint record
  5. Return true if checkpoint advanced
```

### Important Limitation

Ownership and checkpoints live in **different blobs**, so this is not a cross-blob transaction.

That means the library can reject:
- obviously stale tokens,
- backward checkpoint attempts,
- partitions already known to be lost,

but it cannot make ownership transfer and checkpoint update fully atomic across storage records.

This is why the public contract still requires:
- at-least-once expectations, and
- idempotent processing by the caller.

### Example Sketch

```csharp
public async Task<bool> UpdateCheckpointAsync(
    FetchedEvent fetchedEvent,
    CancellationToken cancellationToken = default)
{
    CheckpointToken token = fetchedEvent.CheckpointToken;

    OwnershipRecord? ownership = await _partitionManager.GetCurrentOwnershipAsync(
        token.PartitionId,
        cancellationToken);

    if (ownership == null ||
        ownership.OwnerIdentifier != _options.InstanceId ||
        ownership.OwnershipEpoch != token.OwnershipEpoch)
    {
        return false;
    }

    return await _checkpointStore.TryAdvanceAsync(token, cancellationToken);
}
```

---

## Ownership Renewal

Owned partitions must be renewed before they are considered expired.

Renewal runs every `LoadBalancingUpdateInterval` and uses the current known ownership ETag:

```text
foreach owned partition:
  write metadata with If-Match: currentOwnedEtag
    -> keep OwnerIdentifier the same
    -> keep OwnershipEpoch the same
    -> Blob Storage updates ETag and LastModified
```

Possible outcomes:
- success -> update local ETag and keep reading
- `412 Precondition Failed` -> another instance changed the record first; stop reading that partition

On renewal loss the library should:
- close the receiver,
- remove the partition from the owned set,
- invalidate the local ownership epoch snapshot,
- raise `PartitionLost`.

Stopping promptly minimizes the duplicate-processing window after a steal.

---

## Periodic Rebalance

Rebalancing runs inside the same background ownership loop.

### Why It Exists

Without rebalance, ownership drifts when:
- a process crashes,
- a new instance joins,
- a region is rolled during deployment,
- an instance shuts down gracefully,
- partition count changes.

### High-Level Algorithm

```text
LoadBalancingLoop()
  1. List ownership records
  2. Filter expired records using LastModified
  3. Compute active owner count and fair-share targets
  4. If below minimumOwned:
       claim unowned / expired partitions
  5. If no claimable partitions and another owner is above maximumOwned:
       attempt one steal
  6. Renew partitions already owned
```

### Greedy vs. Balanced

The two useful strategies are:

| Strategy | Behavior |
|----------|----------|
| `Balanced` | One claim / steal opportunity per normal interval |
| `Greedy` | Same logic, but reruns rapidly while unbalanced to converge faster |

The core rule stays the same in both modes:

> At most one writer wins a given ownership ETag race.

`Greedy` just reduces the time between attempts while the cluster is clearly unbalanced.

---

## Rolling Deployments

ETag ownership behaves well in region-by-region rolling deploys:

1. Old pods stop renewing.
2. Their ownership records age out after `OwnershipExpirationInterval`.
3. Surviving instances claim the expired partitions.
4. New pods come up and steal back a fair share if they start under-loaded.

This is slower than a perfect graceful handoff because expiry is still time-based, but it is more tolerant than short Blob lease durations in a cross-region system.

If shutdown is graceful, the old owner can relinquish immediately:

```text
OwnerIdentifier = ""
OwnershipEpoch  = ""
write with If-Match: currentOwnedEtag
```

That makes the partition claimable on the next rebalance cycle without waiting for expiry.

---

## Duplicate Processing Window

The main trade-off of ETag ownership is the brief overlap after a steal:

1. Instance A steals a partition and updates the ownership blob.
2. Instance B, the old owner, has not observed the loss yet.
3. Both instances may read from the same partition briefly.
4. B eventually fails renewal with 412 and stops.

Mitigations:
- stop reading immediately on renewal loss,
- keep checkpoints explicit and monotonic,
- keep processing idempotent.

This is the same general trade-off made by `EventProcessorClient`.

---

## Graceful Shutdown

Shutdown should:

1. stop the load-balancing / renewal loop,
2. close all owned receivers,
3. relinquish ownership blobs with `If-Match` on the last known ETag,
4. close the shared Event Hubs client / connection.

Relinquishing ownership is better than waiting for expiry because it shortens the window before another instance can claim the partition.

---

## End-to-End Flow

```text
                         +--------------------------------------+
                         |          Azure Blob Storage          |
                         |  ownership/{partitionId}            |
                         |  checkpoint/{partitionId}           |
                         +-----------------+--------------------+
                                           ^
                                           |
+------------------------------------------+----------------------------------+
| Pull-mode consumer instance                                                 |
|                                                                             |
| StartAsync                                                                  |
|   -> GetPartitionIds()                                                      |
|   -> Ensure ownership blobs exist                                           |
|   -> Claim partitions with If-Match on ownership ETag                       |
|   -> Restore checkpoint offsets                                             |
|   -> Open one PartitionReceiver per owned partition                         |
|                                                                             |
| Background loop                                                             |
|   -> List ownership records                                                 |
|   -> Rebalance                                                              |
|   -> Renew owned records with If-Match on current ETag                      |
|                                                                             |
| FetchAsync                                                                  |
|   -> serialize through _fetchGuard                                          |
|   -> ReceiveBatchAsync(1, FetchWaitTime) across selected partitions         |
|   -> return FetchResult (no checkpoint)                                     |
|                                                                             |
| UpdateCheckpointAsync                                                       |
|   -> validate ownership epoch                                               |
|   -> advance checkpoint only if monotonic                                   |
+------------------------------------------+----------------------------------+
                                           |
                                           v
                         +--------------------------------------+
                         |          Event Hubs Broker           |
                         |     AMQP connection and links        |
                         +--------------------------------------+
```

---

## Key Design Properties

| Property | Detail |
|----------|--------|
| **Ownership mechanism** | Blob Storage metadata with ETag-based optimistic concurrency |
| **Freshness source** | Blob service `LastModified`, not custom timestamp metadata |
| **Ownership transfer safety** | Exact `If-Match` on observed ownership ETag |
| **Claim / steal behavior** | First successful writer for a given ETag wins |
| **Reader model** | One `PartitionReceiver` per owned partition |
| **Fetch concurrency** | Parallel across selected partitions, serialized across fetch calls |
| **Fetch granularity** | At most one event per queried partition |
| **Checkpoint policy** | Explicit and monotonic |
| **Checkpoint staleness detection** | Ownership epoch captured at fetch time |
| **Delivery semantics** | At-least-once; caller idempotency required |

---

## Configuration Reference

```csharp
public sealed class EventHubPullConsumerOptions
{
    // Event Hub
    public string FullyQualifiedNamespace { get; set; }
    public string EventHubName            { get; set; }
    public string ConsumerGroup           { get; set; } = "$Default";
    public TokenCredential Credential     { get; set; }

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

---

## Edge Cases and Failure Modes

| Scenario | Behavior |
|----------|----------|
| **Instance crash** | Ownership expires after `OwnershipExpirationInterval`; another instance claims on a later rebalance cycle |
| **Graceful shutdown** | Ownership is relinquished immediately and can be claimed on the next cycle |
| **Two stealers race** | One wins the observed ETag; the other gets 412 |
| **Renewal loses race** | Partition is removed locally and `PartitionLost` fires |
| **Queued stale event is checkpointed later** | Rejected if checkpoint already advanced or ownership epoch changed |
| **Concurrent fetch calls** | Serialized by the library |
| **No event on a partition** | Partition lands in `EmptyPartitions` after `FetchWaitTime` |
| **Prefetch buffers non-selected partitions** | Possible; `maxPartitions` is a drain limit, not a hard broker isolation boundary |
| **Partition count increases** | Detected on a later partition discovery / rebalance pass |
| **Cross-region latency spike** | Expiration window is larger than one renewal interval, so brief misses do not immediately orphan ownership |

---

## Blob Lease Alternative

Blob leases are still a viable alternative, but they change the trade-offs:

| Aspect | ETag Ownership | Blob Leases |
|--------|----------------|-------------|
| **Stealing active owner** | Supported | Not supported |
| **Crash detection** | Application TTL | Lease TTL |
| **Scale-up convergence** | Fast with greedy rebalance | Slower |
| **Duplicate risk during rebalance** | Brief overlap possible | Lower |
| **Cross-region tolerance** | Better with larger expiry window | More sensitive to renewal latency |

Blob leases are worth preferring when strict single-owner enforcement matters more than fast rebalance and duplicate tolerance.

---

## Related Documents

- [EventHub-PullMode-LibraryAPI.md](./EventHub-PullMode-LibraryAPI.md) — Public API contract
- [EventHub-PushMode-Internals.md](./EventHub-PushMode-Internals.md) — Push-mode internals and background processor behavior
