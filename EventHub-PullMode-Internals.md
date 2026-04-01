# Event Hub Pull-Mode: Internal Architecture & Design

## Overview

Pull mode inverts the control flow relative to push mode: **your code decides when to fetch events**, rather than the SDK calling your handler when they arrive. The SDK primitive is `EventHubConsumerClient` (`Azure.Messaging.EventHubs`). Unlike `EventProcessorClient`, it provides no built-in partition ownership, load balancing, or checkpointing -- all of that becomes the application's responsibility.

This document covers:
- How `EventHubConsumerClient` works internally at the AMQP level
- How to build distributed partition ownership on top of it using ETag-based ownership (chosen design) or blob leases (alternative)
- How `ReadEventsFromPartitionAsync` and `ReceiveAsync` behave under the hood
- Design decisions and trade-offs for a production pull-mode consumer

---

## SDK Packages

| Package | Purpose |
|---------|---------|
| `Azure.Messaging.EventHubs` | `EventHubConsumerClient`, `PartitionProperties`, `EventData`, `EventPosition` |
| `Azure.Storage.Blobs` | ETag-based partition ownership; checkpoint persistence |
| `Azure.Identity` | `DefaultAzureCredential` for authentication |

No `Azure.Messaging.EventHubs.Processor` package is needed -- pull mode deliberately avoids `EventProcessorClient`.

---

## Pull Mode vs. Push Mode at a Glance

| Aspect | Push Mode (`EventProcessorClient`) | Pull Mode (`EventHubConsumerClient`) |
|--------|------------------------------------|--------------------------------------|
| **Trigger model** | SDK calls your handler continuously | Your code calls fetch on demand |
| **Partition ownership** | Automatic (Blob Storage ETag coordination) | Manual (ETag-based ownership -- same mechanism as push mode) |
| **Checkpointing** | Built-in `UpdateCheckpointAsync()` | Manual (write blob metadata) |
| **Load balancing** | Automatic, gradual, ETag-raced | Developer-implemented (same ETag algorithm) |
| **Fetch granularity** | Continuous stream | Exactly N events per call |
| **Backpressure** | Natural -- handler blocks the loop | Explicit -- you control when to call |
| **Thread model** | 1 `LongRunning` thread per partition | Developer-managed |
| **Blob Storage required** | Yes (ownership + checkpoints) | Yes (ownership + checkpoints) |
| **AMQP mechanism** | Credit-push, persistent link, prefetch=300 | Iterator-based; prefetch configurable |

---

## `EventHubConsumerClient`: What It Is Internally

`EventHubConsumerClient` is a **thin wrapper over a single AMQP connection** to the Event Hubs namespace. It does not manage partitions, ownership, or background loops. It exposes two main reading APIs:

### `ReadEventsFromPartitionAsync` (iterator/streaming)

```csharp
await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsFromPartitionAsync(
    partitionId,
    startingPosition,
    cancellationToken))
{
    EventData data = partitionEvent.Data;
    // process...
}
```

Internally this opens a **`ReceivingAmqpLink`** on the AMQP connection and returns an `IAsyncEnumerable<PartitionEvent>` backed by a prefetch buffer. Each `await foreach` iteration:
1. Checks the local prefetch buffer (filled by broker push on the AMQP link)
2. If empty, suspends until the broker pushes the next message or `MaximumWaitTime` elapses
3. Yields one `PartitionEvent` at a time

This is a **long-lived, stateful iterator** -- the AMQP link stays open across iterations. Disposing or cancelling the iterator closes the link.

### `ReceiveAsync` (batch, on-demand)

Not directly on `EventHubConsumerClient` -- the lower-level batch receive is accessed via `EventHubConsumerClient` creating a `PartitionReceiver`:

```csharp
var receiver = new PartitionReceiver(
    consumerGroup,
    partitionId,
    startingPosition,
    fullyQualifiedNamespace,
    eventHubName,
    credential,
    new PartitionReceiverOptions { PrefetchCount = 10 });

IEnumerable<EventData> batch = await receiver.ReceiveBatchAsync(
    maxMessages: 1,
    maxWaitTime: TimeSpan.FromSeconds(5),
    cancellationToken);
```

`PartitionReceiver` opens its own AMQP link. `ReceiveBatchAsync` drains up to `maxMessages` from the prefetch buffer, waiting up to `maxWaitTime` if the buffer is empty.

---

## AMQP Internals: What Happens Under the Hood

Both APIs share the same AMQP machinery described in the push-mode doc -- the key difference is **prefetch count** and **who drives the loop**.

### AMQP Link Setup

```csharp
var linkSettings = new AmqpLinkSettings
{
    Role             = true,                      // receiver
    TotalLinkCredit  = prefetchCount,             // default 300 for ReadEventsFromPartitionAsync
                                                  // configurable for PartitionReceiver
    AutoSendFlow     = prefetchCount > 0,         // auto-replenish credits
    SettleType       = SettleMode.SettleOnSend,   // pre-settled; no ACK per message
    Source           = new Source {
        Address   = partitionEndpoint,
        FilterSet = new FilterSet {               // stream position filter
            { AmqpFilter.ConsumerFilterName, offsetFilter }
        }
    }
};
```

The broker streams events into the **local prefetch buffer** as long as credits are available. This is the same credit-based push at the AMQP layer as in push mode. The difference is that in pull mode, **your application loop controls when the buffer is drained** -- not the SDK's background thread.

### Prefetch Buffer Behavior

```
ReceiveBatchAsync(maxMessages=1, maxWaitTime=5s)
  ├── Prefetch buffer non-empty → dequeue up to maxMessages, return immediately
  └── Prefetch buffer empty     → wait up to maxWaitTime for broker to push
                                    ├── Event arrives   → return it
                                    └── Timeout elapsed → return empty collection
```

> **Key implication for pull mode:** Even though you call fetch "on demand", the broker is continuously pushing into the prefetch buffer in the background as long as the AMQP link is open. Setting `PrefetchCount = 1` (or `0` to disable) ensures the broker only sends what you explicitly ask for -- critical if you want strict on-demand semantics.

---

## Distributed Partition Ownership

`EventHubConsumerClient` has no concept of distributed ownership. In a multi-instance deployment, you must coordinate which instance owns which partition. This document covers two mechanisms:

1. **ETag-based ownership** (chosen design) -- the same mechanism used by `EventProcessorClient` in push mode
2. **Blob leases** (alternative) -- Azure Blob Storage exclusive leases

### Chosen Design: ETag-Based Ownership

ETag-based ownership uses **Azure Blob Storage ETag-versioned metadata** for optimistic concurrency control. This is the same mechanism that `EventProcessorClient` uses internally, adapted for pull-mode use.

#### How ETag Ownership Works

Each partition's ownership is a small blob with metadata fields:

| Field | Example Value |
|-------|---------------|
| `OwnerIdentifier` | `"MyProcessor-1"` |
| `PartitionId` | `"3"` |
| `ConsumerGroup` | `"$Default"` |
| `LastModifiedTime` | `2026-04-01T10:00:00Z` |
| **ETag** (auto-managed by Blob Storage) | `"0x8DC4A2B3C4D5E6F"` |

There is no distributed lock. All coordination happens through **conditional PUTs** with `If-Match` headers:

| Operation | How It Works |
|-----------|-------------|
| **Claim unowned** | Conditional PUT with `If-Match: *` (or the existing ETag) -- sets `OwnerIdentifier` to this instance |
| **Steal from another** | Same conditional PUT -- overwrites existing owner if ETag matches what was read |
| **Renew ownership** | Same conditional PUT -- updates `LastModifiedTime` by writing metadata; refreshes ETag |
| **Relinquish** | Set `OwnerIdentifier = string.Empty` -- blob remains; other instances detect it as unowned |

A "steal" succeeds only if the ETag hasn't changed since the instance last read it. If another instance renewed or stole the partition first, the ETag will be different and the PUT returns **HTTP 412 Precondition Failed** -- the stealing instance retries on the next load-balancing cycle.

#### Ownership Blob Layout

```
{container}/
├── {namespace}/{hub}/{consumer-group}/
│   ├── ownership/
│   │   ├── 0    ← ownership blob (metadata: OwnerIdentifier, etc.)
│   │   ├── 1
│   │   └── ...
│   └── checkpoint/
│       ├── 0    ← checkpoint blob (metadata: Offset, SequenceNumber)
│       ├── 1
│       └── ...
```

Blob Storage has a flat namespace -- the path separators are just part of the blob name. All ownership records are fetched efficiently with a single **prefix query**:

```csharp
string prefix = $"{fullyQualifiedNamespace}/{eventHubName}/{consumerGroup}/ownership";

await foreach (BlobItem blob in containerClient.GetBlobsAsync(
    traits: BlobTraits.Metadata,   // include OwnerIdentifier, LastModifiedTime, etc.
    prefix: prefix))
{
    // blob.Properties.ETag gives the version for optimistic concurrency
    // blob.Metadata["OwnerIdentifier"] gives current owner
}
```

This is a server-side filter -- only matching blobs are returned over the wire. Typically a single HTTP call for all ownership records.

#### Ownership Expiry

Unlike blob leases (which the Azure Storage service enforces), ETag ownership expiry is **enforced by application logic**:

- Each load-balancing cycle reads all ownership blobs and checks `LastModifiedTime`
- If `DateTimeOffset.UtcNow - LastModifiedTime > OwnershipExpirationInterval` (default: 2 minutes), the partition is considered **expired/unowned**
- Any instance can then claim it via conditional PUT

This application-level TTL is more tolerant of network latency than service-enforced lease TTLs. A 2-minute expiry window provides ample margin for cross-region blob storage operations, where a 30-second lease renewal would be fragile.

#### Example: ETag Race During Steal

```
Blob: ownership/3
  Metadata:  OwnerIdentifier = "B"
  ETag:      "0x8DC4A2B3C4D5E6F"    ← set when B last renewed

1. A reads ownership/3 → sees OwnerIdentifier="B", ETag="...E6F"
2. A determines B owns too many partitions → decides to steal partition 3

3. A writes conditional PUT:
     x-ms-meta-owneridentifier: A
     If-Match: "0x8DC4A2B3C4D5E6F"

4. Two outcomes:

   ETag unchanged → 200 OK → A now owns partition 3
   B renewed (ETag changed) → 412 Precondition Failed → A retries next cycle
```

**Timeline of a failed steal:**

```
t=0s    B renews partition 3             ETag → "...E6F"
t=15s   A reads ownership, sees "...E6F", decides to steal
t=28s   B renews partition 3 again       ETag → "...5F0"  ← changed
t=29s   A writes with If-Match "...E6F"                   ← stale!
        → 412 Precondition Failed                          ← A loses the race
t=30s   A's next LB cycle: re-reads, re-evaluates, maybe retries
```

---

## Why ETag Ownership (Design Rationale)

The ETag-based approach was chosen over blob leases for several reasons specific to this deployment:

### Rolling Deployments and Instance ID Churn

With 6 regions x 20 instances = **120 total instances**, instance IDs change on every redeployment (pod restarts, rolling deploys in Kubernetes). Deployments are **region-by-region** -- one region's 20 instances restart while the other 5 regions (100 instances) remain active.

| Mechanism | Behavior on Region-by-Region Rolling Deploy |
|-----------|---------------------------------------------|
| **Blob leases** | As each pod restarts, its lease is orphaned for up to `LeaseDuration` (30-60s). The 100 surviving instances across other regions cannot expedite this -- the Azure Storage service enforces the lock. With 20 pods restarting sequentially, there is a rolling cascade of orphaned partitions throughout the deploy window. |
| **ETag ownership** | As each pod restarts, its ownership records stop being renewed. The 100 surviving instances detect stale ownership on their next LB cycle and **redistribute the partitions among themselves**. When the new pods come up, they steal back a fair share from the temporarily overloaded survivors. The 100 active instances act as a buffer -- partitions are never unprocessed. |

In a region-by-region deploy, the 100 surviving instances absorb the deploying region's partitions — but only **after `OwnershipExpirationInterval` (2 minutes)** elapses for each departing pod's ownership records. Once expired, greedy mode redistributes the freed partitions in sub-seconds. The newly started instances then steal back a fair share from temporarily overloaded survivors on their first LB cycle. The total dark window per partition is bounded by the 2-minute expiry — longer than blob lease TTL (30-60s), but more tolerant of cross-region latency and GC pauses.

### Faster Convergence and Recovery

| Aspect | Blob Leases | ETag Ownership |
|--------|-------------|----------------|
| **Crash detection** | Wait for lease expiry (30-60s) | Detected on next LB cycle via `LastModifiedTime` check |
| **Partition claiming speed** | One unowned partition per rebalance cycle (60s) | Greedy mode: cycle delay drops to ~15ms when unbalanced |
| **Stealing from overloaded** | Not possible -- active leases cannot be broken | Conditional PUT wins if ETag matches; overloaded owners can lose partitions |
| **Scale-up convergence** | Multiple rebalance cycles (minutes) | Greedy mode converges in seconds |

With blob leases, when a region's instances restart, the surviving 100 instances cannot claim the orphaned partitions until the leases expire -- then must wait through multiple 60-second rebalance cycles to redistribute. With ETag ownership and greedy mode, survivors detect stale ownership and steal within one LB cycle, converging in seconds.

### Cross-Region Blob Storage Latency

The deployment spans 6 regions accessing a shared blob storage account.

| Mechanism | Latency Sensitivity |
|-----------|---------------------|
| **Blob leases** | Lease renewal must succeed within `LeaseDuration` (30s). Cross-region blob storage latency (50-200ms typical, spikes to seconds) makes a 15s renewal interval fragile. A single missed renewal = lost partition. |
| **ETag ownership** | Ownership TTL is 2 minutes. A renewal cycle runs every 30s. Even with 2-3 consecutive slow renewals, the 2-minute window provides ample buffer. |

### Trade-Off: Duplicate Processing Window

ETag ownership has one downside compared to blob leases: when a partition is stolen, there is a brief window where **both the old and new owner may process events simultaneously**.

- The old owner doesn't know it lost ownership until it tries to renew (or until its next load-balancing cycle)
- During this window, both owners may fetch and process the same events
- **Mitigation:** Idempotent processing is required. The checkpoint mechanism ensures the new owner resumes from the last committed offset, so duplicates are bounded by the checkpoint interval.

This is the same trade-off that `EventProcessorClient` makes in push mode. In practice, the duplicate window is short (seconds) and idempotent processing handles it safely.

---

## Instance Startup Sequence

On startup, each instance must discover partitions and claim ownership:

```
StartAsync()
  │
  ├── 1. GetPartitionIdsAsync()
  │       → Query Event Hub for partition list (e.g. ["0","1","2","3","4","5","6","7"])
  │
  ├── 2. EnsureOwnershipBlobsExistAsync()
  │       → For each partitionId: create blob if not exists (PUT blob, no overwrite)
  │       → All instances race; only first PUT succeeds; others get 409 Conflict → OK
  │
  ├── 3. ListOwnershipAsync()
  │       → Prefix query for all ownership blobs under
  │         {namespace}/{hub}/{consumer-group}/ownership/
  │       → Read metadata: OwnerIdentifier, LastModifiedTime, ETag
  │       → Categorize each partition:
  │           - Owned by self         (OwnerIdentifier == this.Id, not expired)
  │           - Owned by other        (OwnerIdentifier != this.Id, not expired)
  │           - Unowned / expired     (OwnerIdentifier empty, or LastModifiedTime
  │                                    older than OwnershipExpirationInterval)
  │
  ├── 4. CalculateTargetCount()
  │       → activeProcessors = count of distinct non-expired OwnerIdentifiers + self
  │       → minimumOwned = floor(totalPartitions / activeProcessors)
  │       → maximumOwned = minimumOwned + 1
  │
  ├── 5. ClaimOwnershipAsync(up to minimumOwned)
  │       → For each unowned/expired partition, attempt conditional PUT:
  │           x-ms-meta-owneridentifier: {this.Id}
  │           If-Match: {existing ETag}   (or If-None-Match: * for new blobs)
  │       → Stop when owned count reaches minimumOwned or no claimable partitions
  │       → In greedy mode: cycle delay drops to ~15ms; rapidly claim until balanced
  │
  ├── 6. RestoreCheckpointsAsync()
  │       → For each owned partition: read checkpoint blob metadata
  │       → If exists: startingPosition = EventPosition.FromOffset(offset, exclusive: false)
  │       → If not:    startingPosition = options.DefaultStartingPosition (e.g. Earliest)
  │
  └── 7. OpenPartitionReadersAsync()
          → For each owned partition: create PartitionReceiver with restored position
          → Store in _readers: Dictionary<string, PartitionReceiver>
```

---

## On-Demand Fetch: `FetchEventsAsync`

The core pull-mode operation -- called by your application whenever it needs events:

```csharp
public async Task<IReadOnlyDictionary<string, EventData?>> FetchEventsAsync(
    CancellationToken cancellationToken = default)
{
    // Concurrent read across all owned partitions
    var tasks = _readers.Select(async kvp =>
    {
        string partitionId = kvp.Key;
        PartitionReceiver receiver = kvp.Value;

        IEnumerable<EventData> batch = await receiver.ReceiveBatchAsync(
            maxMessages: 1,
            maxWaitTime: _options.FetchWaitTime,   // e.g. 500ms
            cancellationToken);

        EventData? eventData = batch.FirstOrDefault();
        return (partitionId, eventData);
    });

    var results = await Task.WhenAll(tasks);

    // Checkpoint partitions that yielded an event
    foreach (var (partitionId, eventData) in results.Where(r => r.eventData != null))
        await UpdateCheckpointAsync(partitionId, eventData!, cancellationToken);

    return results.ToDictionary(r => r.partitionId, r => r.eventData);
}
```

Key design decisions:

| Decision | Rationale |
|----------|-----------|
| **Concurrent reads across partitions** | `Task.WhenAll` -- all partitions fetched in parallel; total latency = slowest partition |
| **`maxMessages: 1`** | Strict on-demand semantics -- exactly one event per partition per call |
| **Short `maxWaitTime`** | e.g. 500ms -- avoids blocking the caller when a partition has no events |
| **Checkpoint after fetch** | At-least-once: checkpoint after caller receives data. Swap order for at-most-once. |
| **Null for empty partitions** | Partitions with no available events return `null` -- caller decides whether to retry |

---

## Ownership Renewal: Background Timer

Ownership must be renewed before it expires. A background timer handles this by updating ownership blob metadata every cycle to refresh `LastModifiedTime`:

```
OwnershipRenewalTimer (fires every LoadBalancingUpdateInterval, e.g. every 30s)
  │
  └── foreach owned partition:
        try
          → ClaimOwnershipAsync(partitionId)   ← conditional PUT with If-Match: {currentETag}
          → Updates LastModifiedTime + gets new ETag
        catch RequestFailedException (412 Precondition Failed)
          → ownership was stolen by another instance
          → ClosePartitionReader(partitionId)
          → _readers.TryRemove(partitionId)
          → _ownedPartitions.TryRemove(partitionId)
          → Log warning: partition {id} lost (stolen)
```

**What "losing ownership" means:**
- Another instance called `ClaimOwnershipAsync` on the same blob and won the ETag race (because this instance was overloaded, or the load balancer determined rebalancing was needed)
- The renewal conditional PUT returns HTTP 412 Precondition Failed (ETag mismatch)
- This instance must stop reading that partition immediately to minimize **duplicate processing**

> **Difference from blob leases:** With leases, renewal failure means the lease expired and someone else acquired it. With ETag ownership, failure means someone **actively stole** the partition -- this can happen even when the ownership isn't expired, enabling faster rebalancing.

---

## Periodic Rebalance

### Why Rebalancing Exists

An Event Hub has a fixed number of partitions (e.g., 32). In production, you typically run multiple consumer instances (e.g., 120 pods across 6 regions, 20 per region) for high availability and throughput. Rebalancing is the process of **redistributing partition ownership across instances** so that work is spread evenly.

Without rebalancing, partition assignments become unbalanced when:
- An instance **crashes** -- its partitions become unrenewed, detected as expired on the next LB cycle
- An instance **scales up** -- the new instance starts with 0 partitions while existing instances are overloaded
- An instance **scales down gracefully** -- its partitions are relinquished but no one picks them up until claimed
- A **region-by-region rolling deploy** restarts one region's instances -- old ownership records go stale, surviving regions must absorb the load

Rebalancing runs on a background timer (default every 30s via `LoadBalancingUpdateInterval`) and corrects these imbalances automatically.

### Rebalance Algorithm

The algorithm mirrors `EventProcessorClient`'s load-balancing logic:

```
LoadBalancingTimer (every LoadBalancingUpdateInterval, e.g. 30s)
  │
  ├── 1. ListOwnershipAsync()
  │       → Prefix query for all ownership blobs
  │       → Read metadata + ETag for each
  │       → Filter expired: LastModifiedTime > OwnershipExpirationInterval ago
  │       → Build: activeProcessors, ownedByMe, ownedByOther, unowned/expired
  │
  │       activeProcessors is determined by counting distinct non-empty
  │       OwnerIdentifier values from non-expired ownership records:
  │
  │       activeProcessors = ownership
  │           .Where(o => !string.IsNullOrEmpty(o.OwnerIdentifier))
  │           .Where(o => UtcNow - o.LastModifiedTime <= OwnershipExpirationInterval)
  │           .Select(o => o.OwnerIdentifier)
  │           .Distinct()
  │           .Count()
  │
  ├── 2. CalculateTargetCount()
  │       → minimumOwned = floor(totalPartitions / activeProcessors)
  │       → maximumOwned = minimumOwned + 1
  │
  ├── 3. If ownedByMe.Count < minimumOwned:
  │       → Attempt to claim ONE partition:
  │           a. Claim an unowned/expired partition (conditional PUT), OR
  │           b. Steal from a processor that owns > maximumOwned, OR
  │           c. Steal from a processor that owns == maximumOwned (to equalize)
  │       → On successful claim:
  │           → RestoreCheckpoint(partitionId)
  │           → OpenPartitionReader(partitionId)
  │       → In greedy mode: if still below minimum, next cycle fires in ~15ms
  │
  └── 4. If ownedByMe.Count > maximumOwned:
          → Relinquish excess partitions:
              → Set OwnerIdentifier = string.Empty (conditional PUT)
              → ClosePartitionReader(partitionId)
```

**Key differences from the blob lease approach:**
- **Stealing is possible** -- the algorithm can take partitions from overloaded owners, not just claim expired/unowned ones

### Active Processors vs. Standby Instances

`activeProcessors` is inferred from ownership blobs -- there is no service registry. Only instances that **currently own at least one partition** appear in ownership records. Instances that have never successfully claimed a partition are invisible to the algorithm.

At scale, this means most instances are standby:

```
120 instances, 32 partitions:
  Instances with ownership records:  32  (each owns 1 partition)
  Standby instances (no records):    88  (invisible to load balancer)

  activeProcessors = 32  (not 120)
  minimumOwned = floor(32 / 32) = 1
  maximumOwned = 1 + 1 = 2
```

This is correct behavior -- you can't have more concurrent readers than partitions on a single consumer group. The algorithm naturally limits active readers to roughly match partition count. Standby instances provide **high availability**: when an active instance crashes or deploys, a standby instance claims the freed partition on its next LB cycle.

> **Same as `EventProcessorClient`:** Push mode uses the same logic. Standby instances are invisible until they successfully claim a partition, at which point they appear in subsequent ownership listings.
- **Greedy mode** -- when unbalanced, the cycle delay drops to ~15ms for rapid convergence; once balanced, the delay returns to the full `LoadBalancingUpdateInterval` (30s)
- **At most one partition per cycle** -- applies to both balanced and greedy mode. The difference is cycle frequency: balanced mode waits 30s between cycles (gradual); greedy mode waits ~15ms (rapid). Greedy mode doesn't claim multiple partitions per cycle — it just runs many more cycles per second.

### Convergence: How Balance Is Reached

Rebalancing is **gradual** in balanced mode (one partition per 30s cycle) but **rapid** in greedy mode (one partition per ~15ms until balanced). This means greedy mode can redistribute dozens of partitions in seconds.

**Example: 32 partitions, 6 regions x 20 instances, Region B deploys**

Rolling deployment starts new pods **before** terminating old ones (rolling update strategy). During the deploy, both old and new pods coexist briefly.

```
Steady state: 120 instances, 32 partitions — balanced
  Each instance owns 0 or 1 partition (floor(32/120) = 0, max = 1)
  32 instances each own 1 partition; 88 are standby

── Region B starts rolling deploy (20 instances, one pod at a time) ──

Phase 1: New pod starts alongside old pod
  Pod B-1' (new ID) starts while Pod B-1 (old) is still running
  → B-1' runs LB cycle: minimumOwned = floor(32/121) = 0, max = 1
  → B-1' owns 0, which is >= minimumOwned (0) → no action needed
  → B-1' joins as standby

Phase 2: Old pod terminates
  Pod B-1 receives SIGTERM → graceful shutdown:
    → Relinquishes ownership (sets OwnerIdentifier = empty)
    → Closes AMQP links
  → If graceful shutdown succeeds: partition is immediately unowned
  → If ungraceful (crash/OOM): ownership expires after 2 minutes

Phase 3: Freed partition claimed
  On next LB cycle, any standby instance (including B-1') detects
  the unowned partition
  → Greedy mode: claims it within ~15ms
  → Partition downtime ≈ 0 (graceful) or ≈ 2 minutes (crash)

Phase 4: Repeat for each pod in Region B
  Pod B-2' starts → B-2 terminates → freed partition claimed → ...
  Deploy progresses one pod at a time over ~5-10 minutes

Phase 5: Steady state restored
  32 partitions across 120 instances (32 active, 88 standby)
  Region B has all new instance IDs; partition distribution unchanged
```

> **Key advantage of start-before-stop:** The new pod is already running and participating in LB cycles before the old pod terminates. When the old pod relinquishes its partition on graceful shutdown, the freed partition is claimed almost instantly by any standby instance (potentially the new pod itself). There is no dark window where the partition is unprocessed — unless the old pod crashes without graceful shutdown, in which case the 2-minute `OwnershipExpirationInterval` applies.

> **Key insight for region-by-region deploy:** The 100 surviving instances across 5 other regions act as a buffer. Partitions are never unprocessed -- survivors absorb the deploying region's load. The deploying region's new instances rejoin as standby and participate in future rebalancing. This is much smoother than a full-fleet deploy where all 120 instances restart simultaneously.

### Partition Stealing: ETag-Based Safety

Unlike the blob lease approach (where active leases cannot be broken), ETag ownership **allows stealing partitions from active owners**. This is safe because of the optimistic concurrency guarantee:

> **At most one instance successfully claims a given partition per ETag version.** If two instances race to steal the same partition, only one conditional PUT succeeds. The loser gets HTTP 412.

However, there is a brief **dual-ownership window** after a steal:
1. Instance A's conditional PUT succeeds -- A is now the owner in blob storage
2. Instance B (the old owner) doesn't know yet -- it continues reading until its next renewal attempt fails with 412
3. Both A and B may process events from the same partition during this window

**Mitigation:** This window is bounded by `LoadBalancingUpdateInterval` (30 seconds max). Idempotent processing and checkpoint-based resume ensure correctness despite the brief overlap.

### Caller Impact

Rebalancing is transparent -- the library handles it automatically -- but it has real consequences for the caller:

| Effect | Detail |
|--------|--------|
| **`OwnedPartitions` changes over time** | The set of partitions returned by `FetchAsync` is not static; it grows and shrinks as rebalancing occurs |
| **`FetchAsync` adapts automatically** | Only fetches from currently-owned partitions; no caller action needed |
| **`UpdateCheckpointAsync` can return `false`** | If a partition was lost between fetch and checkpoint (ownership stolen during rebalancing), the checkpoint write is a no-op |
| **Events may be re-delivered** | When a partition moves to a new owner, the new owner resumes from the last checkpoint -- any events fetched but not yet checkpointed by the old owner will be re-processed |
| **`PartitionAcquired` / `PartitionLost` events fire** | Callers can hook these for logging, metrics, or application-level cleanup (e.g., flushing in-memory state for a lost partition) |

The at-least-once delivery guarantee and idempotency requirement described in the Library API document are direct consequences of rebalancing behavior.

---

## Checkpoint Management

Checkpoints record the last successfully processed event offset per partition, enabling resume after restart.

### Writing a Checkpoint

```csharp
private async Task UpdateCheckpointAsync(
    string partitionId, EventData eventData, CancellationToken ct)
{
    var blobClient = _checkpointContainer.GetBlobClient(
        $"{_eventHubName}/{_consumerGroup}/partition-{partitionId}");

    var metadata = new Dictionary<string, string>
    {
        ["Offset"]         = eventData.Offset.ToString(),
        ["SequenceNumber"] = eventData.SequenceNumber.ToString(),
        ["UpdatedAt"]      = DateTimeOffset.UtcNow.ToString("O")
    };

    // Blob must exist; create if first checkpoint
    await blobClient.CreateIfNotExistsAsync(cancellationToken: ct);
    await blobClient.SetMetadataAsync(metadata, cancellationToken: ct);
}
```

### Reading a Checkpoint on Startup

```csharp
private async Task<EventPosition> GetStartingPositionAsync(string partitionId)
{
    var blobClient = _checkpointContainer.GetBlobClient(
        $"{_eventHubName}/{_consumerGroup}/partition-{partitionId}");

    try
    {
        BlobProperties props = await blobClient.GetPropertiesAsync();
        if (props.Metadata.TryGetValue("Offset", out string? offset) && offset != null)
            return EventPosition.FromOffset(offset, isInclusive: false);
    }
    catch (RequestFailedException ex) when (ex.Status == 404)
    {
        // No checkpoint yet — use default
    }

    return _options.DefaultStartingPosition;   // e.g. EventPosition.Earliest
}
```

---

## Graceful Shutdown

```
DisposeAsync()
  │
  ├── 1. Cancel ownership renewal timer and rebalance timer
  │
  ├── 2. foreach owned partition:
  │       ├── await receiver.CloseAsync()          ← closes AMQP link
  │       └── await RelinquishOwnershipAsync()     ← set OwnerIdentifier = string.Empty
  │                                                   (conditional PUT; blob remains)
  │
  └── 3. await consumerClient.CloseAsync()         ← closes AMQP connection
```

Relinquishing (actively giving up) ownership on graceful shutdown means other instances can **immediately** detect these partitions as unowned on their next load-balancing cycle, rather than waiting for the ownership to expire (up to 2 minutes). The blobs are not deleted -- only the `OwnerIdentifier` metadata is cleared.

---

## End-to-End Flow Diagram

```
                         ┌──────────────────────────────────────┐
                         │          Azure Blob Storage          │
                         │  ┌──────────────┐  ┌─────────────┐  │
                         │  │  Ownership   │  │ Checkpoints │  │
                         │  │(ETag-versioned)│ │(per partition)│ │
                         │  └──────▲───────┘  └──────▲──────┘  │
                         └─────────┼─────────────────┼─────────┘
                                   │                 │
┌──────────────────────────────────┼─────────────────┼──────────────────────┐
│  PullModeConsumer Instance       │                 │                      │
│                                  │                 │                      │
│  Startup                         │                 │                      │
│    ├── GetPartitionIds()         │                 │                      │
│    ├── ClaimOwnership() ─────────┘                 │                      │
│    │    (conditional PUT w/ If-Match)              │                      │
│    └── RestoreCheckpoints() ──────────────────────┘                      │
│                                                                           │
│  Background: LoadBalancingTimer (every 30s)                              │
│    ├── RenewOwnership (conditional PUT) per partition                    │
│    │     └── On 412: partition stolen → remove from owned set            │
│    └── Rebalance: claim unowned/expired, steal from overloaded           │
│          └── Greedy mode: ~15ms cycles when unbalanced                   │
│                                                                           │
│  On-Demand: FetchEventsAsync() ← called by your application             │
│    ├── Task.WhenAll across owned partitions                              │
│    │     └── PartitionReceiver.ReceiveBatchAsync(maxMessages=1, 500ms)  │
│    │               ├── drain AMQP prefetch buffer (instant if available) │
│    │               └── await broker push (up to 500ms if empty)         │
│    └── UpdateCheckpointAsync() for each partition that yielded an event  │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │     Event Hubs Broker        │
                    │  (AMQP, persistent TCP)      │
                    │  Partition 0..N              │
                    └─────────────────────────────┘
```

---

## Key Design Properties

| Property | Detail |
|----------|--------|
| **Ownership mechanism** | Azure Blob Storage ETag-based optimistic concurrency (conditional PUT with `If-Match`) |
| **Active processor discovery** | Inferred from non-expired `OwnerIdentifier` metadata on ownership blobs (`LastModifiedTime` within `OwnershipExpirationInterval`) |
| **Partition claiming** | Conditional PUT on unowned/expired blob -- first writer wins the ETag race |
| **Partition stealing** | Conditional PUT on active owner's blob -- wins if ETag matches (owner hasn't renewed since read) |
| **Crash detection** | Ownership record expiry (`LastModifiedTime` older than `OwnershipExpirationInterval`, default 2 minutes) |
| **AMQP mechanism** | Credit-based push into local prefetch buffer (same as push mode) |
| **Fetch granularity** | Exactly N events per partition per call (configurable) |
| **Concurrency** | Concurrent across partitions (`Task.WhenAll`); sequential per partition |
| **Backpressure** | Explicit -- caller controls fetch rate; buffer absorbs bursts |
| **Checkpoint timing** | After successful fetch (at-least-once); or before (at-most-once) |
| **Graceful shutdown** | Relinquish ownership (clear `OwnerIdentifier`) -- immediate detection by other instances |

---

## Configuration Reference

```csharp
public class PullModeConsumerOptions
{
    // Event Hub connection
    public string FullyQualifiedNamespace { get; set; }    // "<ns>.servicebus.windows.net"
    public string EventHubName            { get; set; }
    public string ConsumerGroup           { get; set; } = "$Default";
    public TokenCredential Credential     { get; set; }    // e.g. new DefaultAzureCredential()

    // Blob Storage
    public string BlobConnectionString   { get; set; }
    public string BlobContainerName      { get; set; } = "eventhub-pullmode";

    // Ownership management
    public TimeSpan OwnershipExpirationInterval { get; set; } = TimeSpan.FromMinutes(2);
    public TimeSpan LoadBalancingUpdateInterval  { get; set; } = TimeSpan.FromSeconds(30);
    public LoadBalancingStrategy LoadBalancingStrategy { get; set; } = LoadBalancingStrategy.Greedy;

    // Fetch behavior
    public int      MaxEventsPerFetch    { get; set; } = 1;
    public TimeSpan FetchWaitTime        { get; set; } = TimeSpan.FromMilliseconds(500);
    public int      PrefetchCount        { get; set; } = 1;   // set low for strict on-demand semantics

    // Startup
    public EventPosition DefaultStartingPosition { get; set; } = EventPosition.Earliest;
    public string InstanceId             { get; set; } = $"{Environment.MachineName}-{Guid.NewGuid():N}";
}
```

---

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| **Instance crash (ungraceful)** | Ownership expires after `OwnershipExpirationInterval` (2 min); another instance claims on next LB cycle |
| **GC pause / network blip causes late renewal** | Renewal conditional PUT may fail with 412 if another instance stole the partition during the gap → instance stops reading that partition → brief duplicate window if new owner already started |
| **Rolling deployment (region-by-region)** | One region's 20 instances restart while 100 survive across 5 other regions. Old ownership records expire → survivors absorb partitions via greedy mode. New instances rejoin as standby. Partitions are never unprocessed. |
| **New instance joins** | Lists ownership, finds it owns 0 < minimumOwned → claims unowned or steals from overloaded owners. Greedy mode: converges in sub-seconds. |
| **Instance scales down gracefully** | Relinquishes ownership (clears `OwnerIdentifier`) → freed partitions detectable immediately |
| **All partitions actively owned** | New instance steals from owners with count > maximumOwned. If all owners are at or below maximum, waits for natural expiry or graceful relinquish. |
| **Concurrent `FetchEventsAsync` calls** | Must be serialized -- `PartitionReceiver` is not thread-safe for concurrent reads on the same partition |
| **No events on a partition** | `ReceiveBatchAsync` returns empty after `FetchWaitTime`; that partition returns `null` in result dict |
| **Checkpoint blob missing** | Created on first write; `DefaultStartingPosition` used for initial read |
| **Event Hub partition count increases** | Discovered on next LB cycle's `GetPartitionIdsAsync()` call; new partitions enter as unowned |
| **Cross-region blob storage latency** | 2-minute ownership TTL tolerates latency spikes (50-200ms typical, seconds in worst case). Even if 2-3 renewal cycles are slow, ownership doesn't expire prematurely. |
| **ETag race (two instances steal same partition)** | Only one conditional PUT succeeds; loser gets 412 and retries next cycle. No split-brain. |

---

## Comparison: ETag Ownership vs. Blob Leases

Both approaches use Blob Storage for distributed coordination, but the mechanisms differ significantly. ETag ownership is the chosen design; blob leases are documented here for reference.

| Aspect | ETag Ownership (Chosen) | Blob Leases (Alternative) |
|--------|------------------------|---------------------------|
| **Mechanism** | ETag conditional PUT (`If-Match`) on blob metadata | Azure exclusive lease API (`AcquireLeaseAsync`) |
| **TTL enforcement** | Application logic (checks `LastModifiedTime` against configurable interval) | Azure Storage service (automatic lease expiry) |
| **TTL duration** | 2 minutes (configurable `OwnershipExpirationInterval`) | 15-60 seconds (Azure-enforced `LeaseDuration`) |
| **Stealing** | Possible -- conditional PUT wins even on active owners | Not possible on an active lease (HTTP 409) |
| **Gradual rebalancing** | One partition per ~15ms cycle (greedy) or 30s cycle (balanced) | One partition per 60s rebalance cycle |
| **Crash detection** | `OwnershipExpirationInterval` (2 min default) | Lease expiry (30-60s) |
| **Rolling deploy impact** | Old ownership expires; surviving regions absorb partitions immediately | Orphaned leases block partitions for full `LeaseDuration` per restarting pod |
| **Cross-region tolerance** | 2-min window accommodates latency spikes | 30s lease renewal is fragile with high latency |
| **Duplicate processing risk** | Brief window during steals (mitigated by idempotency) | Minimal -- lease guarantees exclusive access |
| **Renewal cost** | 1 conditional PUT per partition per LB cycle | 1 `RenewLeaseAsync` per partition per renewal interval |
| **Active processor count** | Inferred from non-expired ownership record `OwnerIdentifier` values | Inferred from non-expired `OwnerId` metadata on lease blobs |
| **Scale-up convergence** | Seconds (greedy mode steals from overloaded) | Minutes (wait for voluntary release + rebalance cycles) |

### When to Prefer Blob Leases

Blob leases may be preferable when:
- **Strict single-owner guarantee** is more important than fast convergence (e.g., financial transactions where even brief duplicates are costly)
- **Few instances** (< 10) where rebalancing speed is less critical
- **Single-region deployment** where blob storage latency is consistently low (< 10ms)

---

## Alternative: Blob Lease Ownership (Reference)

This section documents the blob lease approach for reference. It is **not** the chosen design.

### How Blob Leases Work

Azure Blob Storage offers a first-class lease API on blobs:

| Operation | Behavior |
|-----------|----------|
| `AcquireLeaseAsync(duration)` | Acquires an exclusive 15-60s lease on a blob. Returns a `leaseId`. Only one client can hold a lease at a time. |
| `RenewLeaseAsync(leaseId)` | Renews an existing lease, resetting its TTL. Must be called before expiry. |
| `ReleaseLeaseAsync(leaseId)` | Explicitly releases the lease. Other instances can acquire immediately. |
| `BreakLeaseAsync()` | Forces a lease to expire after a break period. Used for recovery. |

A blob becomes leasable simply by existing. The blob body can be empty; ownership metadata is stored in blob metadata fields.

### Blob Lease Layout

```
{container}/
├── leases/
│   ├── {eventhub}/{consumer-group}/partition-0    ← lease blob (empty body)
│   ├── {eventhub}/{consumer-group}/partition-1
│   └── ...
└── checkpoints/
    ├── {eventhub}/{consumer-group}/partition-0    ← checkpoint blob
    ├── {eventhub}/{consumer-group}/partition-1
    └── ...
```

**Lease blob metadata:**

| Field | Value |
|-------|-------|
| `OwnerId` | Instance unique ID (e.g. `hostname-{Guid}`) |
| `AcquiredAt` | UTC timestamp |

### Blob Lease Startup Flow

```
StartAsync()
  ├── GetPartitionIdsAsync()
  ├── EnsureLeaseBlobsExistAsync()
  ├── ScanExistingLeasesAsync()
  │     → Categorize: owned by self, owned by other, unowned/expired
  ├── AcquireLeasesAsync(targetCount)
  │     → AcquireLeaseAsync on unowned partitions (first-come-first-served)
  ├── RestoreCheckpointsAsync()
  └── OpenPartitionReadersAsync()
```

### Blob Lease Rebalance

The blob lease rebalance algorithm is **conservative** -- it only claims unowned/expired leases and voluntarily releases excess. It never forcibly steals an actively-held lease. This provides a strong invariant (at most one owner at any time) at the cost of slower convergence:

- Crash detection: up to `LeaseDuration` (30-60s)
- Scale-up: multiple 60s rebalance cycles
- Rolling deploys: orphaned leases block partitions for the full `LeaseDuration`

### Blob Lease Renewal

```
LeaseRenewalTimer (fires every LeaseDuration / 2, e.g. every 15s)
  └── foreach owned partition:
        try → RenewLeaseAsync(leaseId)
        catch → lease expired/broken → stop reading partition
```

### Blob Lease Shutdown

```
DisposeAsync()
  └── foreach owned partition:
        ├── CloseAsync() AMQP link
        └── ReleaseLeaseAsync(leaseId) → immediate partition availability
```

---

## Open Questions

1. **At-least-once vs. at-most-once** -- Should checkpoint advance before or after the caller confirms processing?
2. **PrefetchCount = 0** -- Disable prefetch entirely for maximum pull fidelity? Trade-off: higher per-fetch latency.
3. **Concurrent fetch calls** -- Should `FetchEventsAsync` be guarded with a `SemaphoreSlim(1,1)` or left to the caller?
4. **Partition count changes** -- How frequently to re-query `GetPartitionIdsAsync()`? On every LB cycle?
5. **Ownership expiration tuning** -- Is 2 minutes optimal for 6-region deployment, or should it be tuned per-region based on observed blob storage latency?

---

## Related Documents

- [EventHub-PushMode-Internals.md](./EventHub-PushMode-Internals.md) -- Deep dive into `EventProcessorClient` internals: AMQP push, LongRunning threads, ETag-based load balancing