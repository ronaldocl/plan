# Model Routing: Actual Queue to Virtual Queue Migration

## 1. Summary

Replace the actual per-document queue model with a virtual queue model across the entire model routing distributed system. Today the aggregator fetches every queued request document from Cosmos DB, sorts them, builds per-instance views, and sends individual `RemoteModelRoutingRequest` entries to each router instance via gRPC. The router uses these to compute exact queue positions.

The virtual queue model replaces this with server-side aggregation: Cosmos DB returns one `VirtualQueue` row per `(scenarioId, priority)` group containing the count and time range. The router estimates queue position via time-based interpolation against `FirstCreatedAt`/`LastCreatedAt` instead of counting individual remote entries.

This is an end-to-end change spanning four system boundaries:

1. **Storage** — new `GetVirtualQueueByFeedRangesAsync` on `IModelRoutingStorageProvider`
2. **Aggregator** — new `GetAggregationsV2Async` method returning `ModelRoutingAggregationsV2`
3. **gRPC API** — new endpoint serving virtual queue aggregations
4. **Router** — new `ModelRouterV2` using `RemoteVirtualQueues` with time-based interpolation

At 60K items with 10% queue depth, the virtual queue query reduces RU/feed-range/request by ~55% and latency by ~34%. The reduction compounds across the system: less data transferred from Cosmos DB, less memory in the aggregator, smaller gRPC payloads, and O(groups) instead of O(documents) processing on the router side.

---

## 2. Background

### 2.1 Current System Architecture

The model routing system is distributed across two services:

**Service.ModelRouting (Aggregator Service):**
- `ModelRoutingAggregator` polls Cosmos DB every 30 ms via `IModelRoutingStorageProvider`
- Caches aggregated state in `ModelRoutingAggregationsCache`
- Serves per-instance views via gRPC (`GetModelRoutingAggregations`)

**Inference Service Instances (Router):**
- `ModelRoutingService` (BackgroundService) polls the aggregator via gRPC
- `ModelRoutingAggregationsClient` deserializes the gRPC response into `ModelRoutingAggregations`
- `ModelRouter.ProcessAggregationsAndAllocate` syncs remote state and allocates queued requests

Data flow:
```
Cosmos DB → ModelRoutingStorageProvider → ModelRoutingAggregator → gRPC → ModelRoutingAggregationsClient → ModelRouter → ModelRoutingState
```

### 2.2 Current Actual Queue Model

The aggregator fetches **every queued document** from Cosmos DB:

```sql
SELECT * FROM modelRoutingRequestsV2 r
WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
```

The aggregator then:
1. Sorts all documents by `(ScenarioId, Priority, CreatedAt)`
2. For each gRPC request, runs `BuildAggregatedQueuedRequests(instanceId, sortedDocuments)` to produce per-instance `RemoteModelRoutingRequest` groups (own requests act as group boundaries)

The router stores these as `FrozenDictionary<string, IReadOnlyList<RemoteModelRoutingRequest>>` keyed by scenarioId and uses them in:
- **`GetQueuePosition`** — counts remote requests with higher priority or earlier timestamp
- **`GetQueuedCount`** — sums remote request counts per scenario for queue-full checks

### 2.3 Problems with the Actual Queue Model

| Problem | Impact |
|---------|--------|
| **O(N) data transfer** | Every queued document is fetched from Cosmos DB, transferred over gRPC, and stored in each router instance's memory. Cost scales linearly with queue depth. |
| **Per-instance view computation** | `BuildAggregatedQueuedRequests` runs once per gRPC request. Each router instance triggers a full scan of all cached documents. |
| **Memory pressure** | The aggregator holds all queued documents in `ModelRoutingAggregationsCache.SortedQueuedRequests`. Under load, this can be tens of thousands of objects. |
| **RU cost** | `SELECT *` fetches full documents when only aggregate statistics are needed. RU scales linearly with document count. |

### 2.4 Virtual Queue Model

The virtual queue model replaces per-document tracking with aggregate statistics:

```sql
SELECT
    r.scenarioId, r.priority,
    COUNT(1) AS queuedCount,
    MIN(r.createdAt) AS firstCreatedAt,
    MAX(r.createdAt) AS lastCreatedAt
FROM modelRoutingRequestsV2 r
WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
GROUP BY r.scenarioId, r.priority
```

Each `VirtualQueue(ScenarioId, Priority, QueuedCount, FirstCreatedAt, LastCreatedAt)` summarizes one `(scenarioId, priority)` group. The router estimates queue position via time-based interpolation rather than exact counting:

- If `createdAt >= LastCreatedAt` → 0 requests ahead (newest)
- If `createdAt < FirstCreatedAt` → all requests ahead (oldest)
- Otherwise → interpolate assuming uniform distribution over `[FirstCreatedAt, LastCreatedAt]`

This trades exact position for O(groups) data transfer and computation. In practice the number of groups is small (number of scenarios × number of priority levels) and bounded by configuration, while the number of documents is unbounded.

### 2.5 Performance Comparison

**No-Queue Baseline (Queue = 0%, Item Count = 60K):**

| Scenario | Avg Latency (ms) | RU/FR/req |
|---|---|---|
| QR Q=0%  | 13.83 | 3.09  |
| VQ Q=0%  | 14.82 | 3.09  |
| QR Q=10% | 35.57 | 20.91 |
| VQ Q=10% | 23.52 | 9.48  |

At zero queue depth both methods are equivalent — the `WHERE` filter eliminates all documents before aggregation, so `GROUP BY` adds negligible overhead. Under 10% queue load, VQ reduces latency by ~34% and RU/FR/req by ~55% compared to QR.

**Performance data at 10% queue depth across item counts (8 feed ranges):**

| Item Count | Avg Lat QR (ms) | Avg Lat VQ (ms) | Latency Delta | RU/FR/req QR | RU/FR/req VQ | RU Saving |
|---|---|---|---|---|---|---|
| 10K | 17.67 | 19.10 | -8.1% | 6.02 | 7.60 | -26.2% |
| 20K | 20.87 | 20.42 | +2.2% | 9.10 | 7.76 | 14.7% |
| 30K | 23.66 | 21.06 | +11.0% | 12.08 | 8.80 | 27.1% |
| 40K | 28.13 | 23.38 | +16.9% | 14.90 | 8.20 | 44.9% |
| 50K | 30.23 | 21.79 | +27.9% | 17.90 | 8.75 | 51.1% |
| 60K | 35.57 | 23.52 | +33.9% | 20.91 | 9.48 | 54.7% |
| 80K | 40.65 | 23.67 | +41.8% | 26.74 | 11.00 | 58.9% |
| 100K | 47.30 | 24.85 | +47.5% | 32.58 | 12.54 | 61.5% |

QR cost scales linearly with document count; VQ cost scales with the number of distinct groups and grows much more slowly.

---

## 3. Requirements

1. **New storage method** — `GetVirtualQueueByFeedRangesAsync` on `IModelRoutingStorageProvider`.
2. **New data model** — `VirtualQueue` record and `ModelRoutingAggregationsV2` carrying virtual queues instead of per-document queued requests.
3. **Aggregator V2 path** — new method on `IModelRoutingAggregator` returning `ModelRoutingAggregationsV2`.
4. **gRPC V2 endpoint** — new RPC serving virtual queue aggregations to router instances.
5. **Router V2** — `ModelRouterV2` consuming `RemoteVirtualQueues` with time-based interpolation for queue position.
6. **State model update** — `IModelRoutingState` gains `RemoteVirtualQueues` and `SyncRemoteState(ModelRoutingAggregationsV2)`.
7. **Runtime switch** — V1/V2 coexistence via configuration flag, enabling incremental rollout and instant rollback.

---

## 4. Design

### 4.1 Data Models

**`VirtualQueue` (internal):**

```csharp
namespace Picasso.ModelRouting;

internal sealed record VirtualQueue(
    string ScenarioId,
    int Priority,
    int QueuedCount,
    DateTimeOffset FirstCreatedAt,
    DateTimeOffset LastCreatedAt);
```

**`VirtualQueue` (external, Cosmos DB deserialization):**

```csharp
namespace Picasso.ModelRouting.Storage.External;

internal sealed record VirtualQueue(
    string ScenarioId,
    int Priority,
    int QueuedCount,
    DateTimeOffset FirstCreatedAt,
    DateTimeOffset LastCreatedAt);
```

**`ModelRoutingAggregationsV2`:**

```csharp
namespace Picasso.ModelRouting;

internal sealed record ModelRoutingAggregationsV2(
    IReadOnlyList<VirtualQueue> VirtualQueues,
    IReadOnlyList<ScenarioUsage> ScenariosUsage,
    IReadOnlyList<EndpointUsage> EndpointsUsage);
```

Replaces `ModelRoutingAggregations.QueuedRequests` (list of `RemoteModelRoutingRequest`) with `VirtualQueues` (list of `VirtualQueue`). The usage fields are unchanged.

### 4.2 Storage Layer

**`IModelRoutingStorageProvider` — add method:**

```csharp
Task<IReadOnlyList<VirtualQueue>> GetVirtualQueueByFeedRangesAsync(CancellationToken cancellationToken);
```

**`ModelRoutingStorageProvider` — add query and implementation:**

```csharp
private const string QueryVirtualQueueByFeedRange = """
    SELECT
        r.scenarioId AS scenarioId,
        r.priority AS priority,
        COUNT(1) AS queuedCount,
        MIN(r.createdAt) AS firstCreatedAt,
        MAX(r.createdAt) AS lastCreatedAt
    FROM modelRoutingRequestsV2 r
    WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
    GROUP BY r.scenarioId, r.priority
    """;

private static readonly QueryDefinition VirtualQueueByFeedRangeQueryDefinition = new(QueryVirtualQueueByFeedRange);

[Activity("Cosmos.GetVirtualQueueByFeedRanges", Owner.ModelRouting)]
private async Task<IReadOnlyList<VirtualQueue>> GetVirtualQueueByFeedRangesCoreAsync(CancellationToken cancellationToken)
{
    var container = ModelRoutingRequests;
    var feedRanges = await container.GetFeedRangesAsync(cancellationToken);

    var virtualQueueArrays = await feedRanges.WhenAllAsync(
        (feedRange, ct) => container.ToListAsync<External.VirtualQueue>(feedRange, VirtualQueueByFeedRangeQueryDefinition, FeedRangeRequestOptions, ct),
        cancellationToken);

    return virtualQueueArrays.SelectMany(static vq => vq).Select(Converters.ToInternal).ToArray();
}
```

**`Converters` — add mapping:**

```csharp
internal static VirtualQueue ToInternal(this External.VirtualQueue virtualQueue) =>
    new(
        virtualQueue.ScenarioId,
        virtualQueue.Priority,
        virtualQueue.QueuedCount,
        virtualQueue.FirstCreatedAt,
        virtualQueue.LastCreatedAt);
```

**`SerializationContext` — add:**

```csharp
[JsonSerializable(typeof(External.VirtualQueue))]
```

### 4.3 Aggregator

**`IModelRoutingAggregator` — add method:**

```csharp
Task<ModelRoutingAggregationsV2> GetAggregationsV2Async(CancellationToken cancellationToken);
```

This coexists with the existing `GetAggregatedView(instanceId)`. The V1 path remains for rollback.

**`ModelRoutingAggregator` — add implementation:**

```csharp
public async Task<ModelRoutingAggregationsV2> GetAggregationsV2Async(CancellationToken cancellationToken)
{
    var getVirtualQueueTask = GetVirtualQueuesAsync(cancellationToken);
    var getUsageTask = GetUsageAsync(cancellationToken);

    var virtualQueues = await getVirtualQueueTask;
    var (scenariosUsage, endpointsUsage) = await getUsageTask;

    ReportAggregatorMetrics(scenariosUsage, endpointsUsage);
    return new(virtualQueues, scenariosUsage, endpointsUsage);
}

private async Task<IReadOnlyList<VirtualQueue>> GetVirtualQueuesAsync(CancellationToken cancellationToken)
{
    var virtualQueues = await modelRoutingStorageProvider.GetVirtualQueueByFeedRangesAsync(cancellationToken);

    // Merge partial results from different feed ranges for the same (scenarioId, priority) group
    return virtualQueues
        .GroupBy(static vq => (vq.ScenarioId, vq.Priority))
        .Select(static g => new VirtualQueue(
            g.Key.ScenarioId,
            g.Key.Priority,
            g.Sum(static vq => vq.QueuedCount),
            g.Min(static vq => vq.FirstCreatedAt),
            g.Max(static vq => vq.LastCreatedAt)))
        .ToArray();
}
```

Note: feed-range queries return partial results per physical partition. The same `(scenarioId, priority)` group may appear in multiple feed ranges and must be merged — sum `QueuedCount`, min `FirstCreatedAt`, max `LastCreatedAt`.

**Key difference from V1 aggregation:** `GetAggregationsV2Async` does **not** call `GetQueuedRequestsByFeedRangesAsync` or `BuildAggregatedQueuedRequests`. It replaces the entire queued-request fetch + in-memory aggregation pipeline with a single `GetVirtualQueueByFeedRangesAsync` call followed by feed-range merging.

### 4.4 gRPC API

**New RPC in `modelrouting.proto`:**

```protobuf
message VirtualQueueEntry {
    string scenario_id = 1;
    int32 priority = 2;
    int32 queued_count = 3;
    google.protobuf.Timestamp first_created_at = 4;
    google.protobuf.Timestamp last_created_at = 5;
}

message GetModelRoutingAggregationsV2Response {
    repeated VirtualQueueEntry virtual_queues = 1;
    repeated ScenarioUsage scenarios_usage = 2;
    repeated EndpointUsage endpoints_usage = 3;
}

service ModelRouting {
    // Existing
    rpc GetModelRoutingAggregations(GetModelRoutingAggregationsRequest) returns (GetModelRoutingAggregationsResponse);
    // New
    rpc GetModelRoutingAggregationsV2(GetModelRoutingAggregationsV2Request) returns (GetModelRoutingAggregationsV2Response);
}
```

**`ModelRoutingService` (gRPC server) — add handler:**

```csharp
public override async Task<External.GetModelRoutingAggregationsV2Response> GetModelRoutingAggregationsV2(
    External.GetModelRoutingAggregationsV2Request request, ServerCallContext context)
{
    var aggregations = await modelRoutingAggregator.GetAggregationsV2Async(context.CancellationToken);
    return ToExternal(aggregations);
}
```

Note: unlike the V1 `GetModelRoutingAggregations` which calls `GetAggregatedView(instanceId)` synchronously (reads from cache), the V2 endpoint calls `GetAggregationsV2Async` which queries Cosmos DB on demand. This is because the virtual queue data is compact enough to compute per-request without pre-caching, and avoids the staleness inherent in the polling cache.

### 4.5 Router — `ModelRouterV2`

Create `ModelRouterV2` as a new class implementing `IModelRouter` and `IRequestAllocationManager`. The V2 router replaces `RemoteModelRoutingRequests` with `RemoteVirtualQueues` and uses time-based interpolation for queue position estimation.

**Why a new class instead of branching in `ModelRouter`:**

- `ModelRouter` uses `IReadOnlyList<RemoteModelRoutingRequest>` (per-instance, ordered entries) throughout `GetQueuePosition`, `GetQueuedCount`, and `AllocateQueuedRequests`. The virtual queue model uses `IReadOnlyDictionary<(string ScenarioId, int Priority), VirtualQueue>` — a fundamentally different data structure and access pattern.
- Queue position estimation changes from exact counting to time-based interpolation — the algorithm itself is different, not just the data source.
- `IModelRoutingState` requires different fields (`RemoteVirtualQueues` vs `RemoteModelRoutingRequests`) and a different sync method (`SyncRemoteState(ModelRoutingAggregationsV2)` vs `SyncWithRemoteState(ModelRoutingAggregations)`).
- Branching every access point in a 730-line class with concurrent state management creates risk disproportionate to the transition period. A separate class keeps both paths clean and testable.
- DI registration switches which implementation is bound to `IModelRouter` — clean cutover with no runtime branching.

**Queue position estimation — time-based interpolation:**

The core algorithmic change is in `GetRemoteNumberOfRequestsAhead`. Without individual documents, the router cannot count exact remote requests ahead. Instead it interpolates based on the time range:

```csharp
private int GetRemoteNumberOfRequestsAhead(string scenarioId, int priority, DateTimeOffset createdAt)
{
    // Count all remote requests with strictly higher priority (lower priority value)
    int remoteRequestsWithHigherPriority = 0;
    foreach (int p in PriorityValues)
    {
        if (p >= priority)
            break;

        if (state.RemoteVirtualQueues.TryGetValue((scenarioId, p), out var virtualQueue))
            remoteRequestsWithHigherPriority += virtualQueue.QueuedCount;
    }

    if (!state.RemoteVirtualQueues.TryGetValue((scenarioId, priority), out var priorityQueue))
        return remoteRequestsWithHigherPriority;

    // Interpolate within the same priority group assuming uniform distribution
    int requestsAheadAtSamePriority = createdAt switch
    {
        _ when createdAt >= priorityQueue.LastCreatedAt => 0,
        _ when createdAt < priorityQueue.FirstCreatedAt => priorityQueue.QueuedCount,
        _ when createdAt == priorityQueue.FirstCreatedAt => priorityQueue.QueuedCount - 1,
        _ => Math.Min(
            priorityQueue.QueuedCount - 1,
            (int)Math.Ceiling(
                1.0 * (priorityQueue.LastCreatedAt - createdAt).Ticks
                    / (priorityQueue.LastCreatedAt - priorityQueue.FirstCreatedAt).Ticks
                    * priorityQueue.QueuedCount))
    };

    return remoteRequestsWithHigherPriority + requestsAheadAtSamePriority;
}
```

The interpolation assumes FIFO ordering (earlier requests are ahead). The fraction `(LastCreatedAt - createdAt) / (LastCreatedAt - FirstCreatedAt)` estimates what proportion of queued requests were created before the current request. This is an approximation — it's exact when requests are uniformly distributed and degrades gracefully when they're not, because the queue position only needs to be correct enough to determine if capacity is available (a threshold check, not a ranking).

**Queue-full check:**

```csharp
private int GetQueuedCount(string scenarioId)
{
    var localQueuedCount = state.GetQueuedRequestsForScenario(scenarioId).Count();

    var remoteQueuedCount = 0;
    foreach (int p in PriorityValues)
    {
        if (state.RemoteVirtualQueues.TryGetValue((scenarioId, p), out var virtualQueue))
            remoteQueuedCount += virtualQueue.QueuedCount;
    }

    return localQueuedCount + remoteQueuedCount;
}
```

### 4.6 `IModelRoutingState` Changes

Add virtual queue support alongside existing fields:

```csharp
internal interface IModelRoutingState : IDisposable
{
    // ... existing members ...

    // New: Remote virtual queues indexed by (scenarioId, priority)
    FrozenDictionary<(string ScenarioId, int Priority), VirtualQueue> RemoteVirtualQueues { get; }

    // New: Sync from V2 aggregations
    void SyncRemoteState(ModelRoutingAggregationsV2 aggregations);
}
```

**`ModelRoutingState.SyncRemoteState`:**

```csharp
public void SyncRemoteState(ModelRoutingAggregationsV2 aggregations)
{
    // Build virtual queue lookup from aggregation
    var newRemoteVirtualQueues = aggregations.VirtualQueues
        .ToFrozenDictionary(
            vq => (vq.ScenarioId, vq.Priority),
            vq => vq);

    RemoteVirtualQueues = newRemoteVirtualQueues;

    // Update scenario and endpoint usage (same as V1)
    ScenarioUsages = ScenarioUsages.Keys.ToFrozenDictionary(
        scenarioId => scenarioId,
        scenarioId => aggregations.ScenariosUsage
            .FirstOrDefault(s => s.ScenarioId == scenarioId, new(scenarioId, 0, 0)),
        StringComparer.Ordinal);

    EndpointUsages = EndpointUsages.Keys.ToFrozenDictionary(
        endpoint => endpoint,
        endpoint => aggregations.EndpointsUsage
            .FirstOrDefault(e => e.Endpoint == endpoint, new(endpoint, 0)),
        StringComparer.Ordinal);
}
```

### 4.7 `ModelRoutingAggregationsClient` Changes

Add V2 client method that calls the new gRPC endpoint and returns `ModelRoutingAggregationsV2`:

```csharp
public async Task<ModelRoutingAggregationsV2> GetAggregatedStateV2Async(CancellationToken cancellationToken)
{
    var aggregations = await modelRoutingGrpcClient.GetModelRoutingAggregationsV2Async(
        new(),
        deadline: timeProvider.GetGrpcCallDeadline(TimeSpan.FromSeconds(5)),
        cancellationToken: cancellationToken);

    var virtualQueues = aggregations.VirtualQueues.Select(ToInternalVirtualQueue).ToArray();
    var scenariosUsage = aggregations.ScenariosUsage.ToDictionary(su => su.ScenarioId, ToInternal, StringComparer.Ordinal);
    var endpointsUsage = aggregations.EndpointsUsage.ToDictionary(eu => eu.Endpoint, ToInternal, StringComparer.Ordinal);

    return new(virtualQueues, scenariosUsage.Values.ToArray(), endpointsUsage.Values.ToArray());
}

private static VirtualQueue ToInternalVirtualQueue(External.VirtualQueueEntry entry) =>
    new(entry.ScenarioId, entry.Priority, entry.QueuedCount, entry.FirstCreatedAt.ToDateTimeOffset(), entry.LastCreatedAt.ToDateTimeOffset());
```

### 4.8 `ModelRoutingService` (BackgroundService) Changes

The background service on each inference instance needs to call the V2 path when enabled:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    do
    {
        try
        {
            if (options.Value.UseVirtualQueue)
            {
                var aggregations = await aggregationsClient.GetAggregatedStateV2Async(stoppingToken);
                requestManager.ProcessAggregationsAndAllocateV2(aggregations);
            }
            else
            {
                var aggregations = await aggregationsClient.GetAggregatedStateAsync(requestManager.InstanceId, stoppingToken);
                requestManager.ProcessAggregationsAndAllocate(aggregations);
            }

            var configs = await aggregationsClient.GetModelRoutingConfigsAsync(stoppingToken);
            configClient.UpdateModelRoutingConfiguration(configs.ToArray());
        }
        catch (Exception ex) when (ex.IsNotCancelled())
        {
            logger.FailedToProcess(ex);
        }
    }
    while (await processingTimer.WaitForNextTickAsync(stoppingToken));
}
```

### 4.9 Runtime Switch

Add `UseVirtualQueue` to `ModelRoutingOptions`:

```csharp
internal sealed record ModelRoutingOptions
{
    // ... existing properties ...

    // When true, the aggregator serves virtual queue data and routers use time-based interpolation.
    // When false, the existing ectual queue model is used.
    // Updatable at runtime via Azure App Configuration. Default: false.
    public bool UseVirtualQueue { get; init; }
}
```

**DI registration** switches the `IModelRouter` binding based on configuration:

```csharp
// Option A: Register both, use configuration to select at runtime
// Option B: Conditional registration at startup
if (options.UseVirtualQueue)
    services.AddSingleton<IModelRouter, ModelRouterV2>();
else
    services.AddSingleton<IModelRouter, ModelRouter>();
```

Note: since `IModelRouter` is a singleton and the flag is read at startup, changing `UseVirtualQueue` requires a restart (unlike `UseV2Container` which was hot-swappable). This is acceptable because the router holds long-lived state and concurrent data structures that can't be safely swapped at runtime.

---

## 5. System Data Flow Comparison

### V1 (Actual Queue)

```
Cosmos DB ──SELECT *──→ StorageProvider ──all docs──→ Aggregator
    ↓ caches sorted docs
    ↓ per gRPC request: BuildAggregatedQueuedRequests(instanceId)
    ↓
gRPC ──RemoteModelRoutingRequest[]──→ AggregationsClient
    ↓
ModelRouter ──exact count──→ GetQueuePosition
    ↓ uses FrozenDictionary<string, IReadOnlyList<RemoteModelRoutingRequest>>
```

### V2 (Virtual Queue)

```
Cosmos DB ──GROUP BY──→ StorageProvider ──VirtualQueue[]──→ Aggregator
    ↓ merges feed-range partials
    ↓ per gRPC request: returns merged VirtualQueue[]
    ↓
gRPC ──VirtualQueue[]──→ AggregationsClient
    ↓
ModelRouterV2 ──interpolation──→ GetQueuePosition
    ↓ uses FrozenDictionary<(string, int), VirtualQueue>
```

Key differences:

| Aspect | V1 | V2 |
|--------|----|----|
| Cosmos query | `SELECT *` (all docs) | `GROUP BY` (aggregates) |
| Data over gRPC | O(queued documents) | O(scenarios × priorities) |
| Aggregator cache | All sorted documents | Not needed (computed per request) |
| Per-instance view | Required (`BuildAggregatedQueuedRequests`) | Not needed (virtual queues are instance-agnostic) |
| Queue position | Exact count | Time-based interpolation |
| State structure | `IReadOnlyList<RemoteModelRoutingRequest>` per scenario | `VirtualQueue` per (scenario, priority) |

---

## 6. Files Changed

### New Files

| File | Description |
|------|-------------|
| `Shared/ModelRouting/VirtualQueue.cs` | `VirtualQueue` internal record |
| `Shared/ModelRouting/Storage/External/VirtualQueue.cs` | `External.VirtualQueue` serialization record |
| `Shared/ModelRouting/ModelRoutingAggregationsV2.cs` | `ModelRoutingAggregationsV2` record (or added to existing `ModelRoutingAggregations.cs`) |
| `Shared/Inference/ModelRouting/ModelRouterV2.cs` | V2 router with time-based interpolation |

### Modified Files

| File | Change |
|------|--------|
| `Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs` | Add `GetVirtualQueueByFeedRangesAsync` |
| `Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs` | Add query constant, `QueryDefinition`, and implementation |
| `Shared/ModelRouting/Storage/Converters.cs` | Add `ToInternal` for `External.VirtualQueue` |
| `Shared/ModelRouting/Storage/External/SerializationContext.cs` | Add `[JsonSerializable(typeof(External.VirtualQueue))]` |
| `Service.ModelRouting/Aggregator/IModelRoutingAggregator.cs` | Add `GetAggregationsV2Async` |
| `Service.ModelRouting/Aggregator/ModelRoutingAggregator.cs` | Add `GetAggregationsV2Async` and `GetVirtualQueuesAsync` |
| `Service.ModelRouting/Api/Services/ModelRoutingService.cs` | Add `GetModelRoutingAggregationsV2` gRPC handler |
| `Service.ModelRouting/Api/Protos/modelrouting.proto` | Add `VirtualQueueEntry` message and V2 RPC |
| `Shared/Inference/ModelRouting/IModelRoutingState.cs` | Add `RemoteVirtualQueues` and `SyncRemoteState(ModelRoutingAggregationsV2)` |
| `Shared/Inference/ModelRouting/ModelRoutingState.cs` | Implement `RemoteVirtualQueues` storage and `SyncRemoteState` |
| `Shared/Inference/ModelRouting/IModelRoutingAggregationsClient.cs` | Add `GetAggregatedStateV2Async` |
| `Shared/Inference/ModelRouting/ModelRoutingAggregationsClient.cs` | Add V2 client method |
| `Shared/Inference/ModelRouting/IRequestAllocationManager.cs` | Add `ProcessAggregationsAndAllocateV2` |
| `Shared/Inference/ModelRouting/ModelRoutingService.cs` | Branch on `UseVirtualQueue` in polling loop |
| `Shared/ModelRouting/ModelRoutingOptions.cs` | Add `UseVirtualQueue` flag |
| `Shared/Inference/AppStartup.cs` | Conditional DI registration for `IModelRouter` |

---

## 7. Migration Plan

### Phase 1 — Storage + Aggregator

Add `GetVirtualQueueByFeedRangesAsync` to storage provider and `GetAggregationsV2Async` to aggregator. Add gRPC V2 endpoint. Deploy with `UseVirtualQueue = false`. No production behaviour change.

### Phase 2 — Router V2

Add `ModelRouterV2`, `IModelRoutingState` changes, and `ModelRoutingAggregationsClient` V2 path. Deploy with `UseVirtualQueue = false`. V2 code is present but inactive.

### Phase 3 — Rollout

Set `UseVirtualQueue = true` per environment. Requires restart (singleton DI binding).

**Sequence:**
1. Staging — validate queue position accuracy, allocation behaviour, RU reduction
2. Production

**Rollback:** Set `UseVirtualQueue = false` and restart. V1 path remains fully functional.

### Phase 4 — Cleanup

After V2 is stable in all environments:

1. Remove `ModelRouter` (V1) class
2. Remove `GetQueuedRequestsByFeedRangesAsync` from `IModelRoutingStorageProvider` and implementation
3. Remove `BuildAggregatedQueuedRequests` from aggregator
4. Remove `GetAggregatedView(instanceId)` and V1 gRPC endpoint
5. Remove `RemoteModelRoutingRequests` from `IModelRoutingState`
6. Remove `SyncWithRemoteState(ModelRoutingAggregations)` from state
7. Remove `UseVirtualQueue` flag
8. Rename `ModelRouterV2` → `ModelRouter`

---

## 8. Testing

### Unit Tests

**Storage:**
- `ModelRoutingStorageProvider.GetVirtualQueueByFeedRangesAsync`: verify feed-range fan-out and conversion.
- `Converters.ToInternal(External.VirtualQueue)`: verify field mapping.

**Aggregator:**
- `ModelRoutingAggregator.GetAggregationsV2Async`: verify feed-range merging (sum counts, min/max timestamps for same group across feed ranges).

**Router V2:**
- `GetRemoteNumberOfRequestsAhead` interpolation:
  - `createdAt` before `FirstCreatedAt` → returns full `QueuedCount`
  - `createdAt` after `LastCreatedAt` → returns 0
  - `createdAt` at midpoint → returns ~half of `QueuedCount`
  - Single-element queue (`FirstCreatedAt == LastCreatedAt`) → returns `QueuedCount - 1`
- `GetQueuedCount`: verify sums local + remote across all priorities.
- `AllocateQueuedRequests`: verify allocation order respects priority.
- Existing `ModelRouter` V1 tests remain unchanged.

### Staging Validation

- Virtual queue results match expected aggregation from the actual queue query.
- RU reduction consistent with benchmarks.
- Queue position estimation produces allocation behaviour comparable to V1 under normal load.
- No 429 throttling.
- gRPC payload size reduction measurable.
