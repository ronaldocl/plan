- Use branch lcao/mr_container_migration_3 in d:\Code\picasso as the baseline.

# Model Routing: Actual Queue to Virtual Queue Migration

## 1. Summary

Replace the actual per-document queue model with a virtual queue model across the entire model routing distributed system. Today the aggregator fetches every queued request document from Cosmos DB, sorts them, builds per-instance views, and sends individual `RemoteModelRoutingRequest` entries to each router instance via gRPC. The router uses these to compute exact queue positions.

The virtual queue model replaces this with server-side aggregation: Cosmos DB returns one `VirtualQueue` row per `(scenarioId, priority)` group containing the count and time range. The router estimates queue position via time-based interpolation against `FirstCreatedAt`/`LastCreatedAt` instead of counting individual remote entries.

This is an end-to-end change spanning four system boundaries:

1. **Storage** — new `GetVirtualQueueByFeedRangesAsync` on `IModelRoutingStorageProvider`
2. **Aggregator** — `EnableVirtualQueueQuery` flag gates virtual queue fetch in `ProcessAggregationsCoreAsync`; new `GetAggregatedViewV2` serves cached V2 data
3. **gRPC API** — new endpoint serving virtual queue aggregations
4. **Router** — new `ModelRouterV2` using a global virtual queue snapshot with time-based interpolation

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

Each `VirtualQueue(ScenarioId, Priority, QueuedCount, FirstCreatedAt, LastCreatedAt)` summarizes one `(scenarioId, priority)` group. In V2, the router uses this Cosmos-derived global snapshot as the sole source of truth for queue-full and queue-position checks; it does not add local in-memory queued requests on top of the aggregate. This intentionally accepts snapshot lag for newly queued local requests in exchange for avoiding overlap/reconciliation logic between local memory and the global aggregate.

The router estimates queue position via time-based interpolation rather than exact counting:

- If `createdAt >= LastCreatedAt` → all requests ahead (newest)
- If `createdAt <= FirstCreatedAt` → 0 requests ahead (oldest)
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
3. **Aggregator V2 path** — `EnableVirtualQueueQuery` flag in `ProcessAggregationsCoreAsync` gates virtual queue fetch; new `GetAggregatedViewV2` on `IModelRoutingAggregator` serves cached V2 data.
4. **gRPC V2 endpoint** — new RPC serving virtual queue aggregations to router instances.
5. **Router V2** — `ModelRouterV2` consuming a global virtual queue snapshot with time-based interpolation for queue position.
6. **State model update** — `IModelRoutingState` gains `GlobalVirtualQueues` and `SyncGlobalState(ModelRoutingAggregationsV2)`.
7. **Runtime switch** — Three feature flags (`EnableVirtualQueueQuery`, `EnableVirtualQueueSync`, `UseVirtualQueue`) for incremental rollout and safe rollback via Azure App Configuration.

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

Add `EnableVirtualQueueQuery` check inside the existing `ProcessAggregationsCoreAsync`. When the flag is true, the aggregator additionally fetches virtual queue data from Cosmos DB and stores it in `cachedAggregationsV2`. The original aggregation path is unchanged.

**`IModelRoutingAggregator` — add V2 method:**

```csharp
internal interface IModelRoutingAggregator
{
    ModelRoutingAggregations GetAggregatedView(string instanceId);        // existing
    ModelRoutingAggregationsV2 GetAggregatedViewV2();                     // new
    Task ProcessAggregationsAsync(CancellationToken cancellationToken);   // existing
}
```

**Add `IOptionsMonitor<ModelRoutingOptions>` to constructor and `GetAggregatedViewV2Core`:**

```csharp
internal sealed partial class ModelRoutingAggregator(
    ILogger<ModelRoutingAggregator> logger,
    Metrics metrics,
    IModelRoutingStorageProvider modelRoutingStorageProvider,
    IOptionsMonitor<ModelRoutingOptions> options)   // new
    : IModelRoutingAggregator
{
    private volatile ModelRoutingAggregationsCache cachedAggregations = new([], [], []);
    private volatile ModelRoutingAggregationsCacheV2 cachedAggregationsV2 = new([], [], []);  // new

    [Activity("ModelRouting.GetAggregatedViewV2", Owner.ModelRouting)]
    private ModelRoutingAggregationsV2 GetAggregatedViewV2Core()
    {
        var cache = cachedAggregationsV2;
        return new(cache.VirtualQueues, cache.ScenariosUsage, cache.EndpointsUsage);
    }
```

**Update `ProcessAggregationsCoreAsync`:**

```csharp
[Activity("ModelRouting.ProcessAggregations", Owner.ModelRouting)]
private async Task ProcessAggregationsCoreAsync(CancellationToken cancellationToken)
{
    var enableVirtualQueueQuery = options.CurrentValue.EnableVirtualQueueQuery;

    var getQueuedRequestsTask = GetQueuedRequestsAsync(cancellationToken);
    var getUsageTask = GetUsageAsync(cancellationToken);
    var getVirtualQueuesTask = enableVirtualQueueQuery
        ? GetVirtualQueuesAsync(cancellationToken)
        : null;

    var queuedRequests = await getQueuedRequestsTask;
    var (scenariosUsage, endpointsUsage) = await getUsageTask;

    cachedAggregations = new(queuedRequests, scenariosUsage, endpointsUsage);

    if (getVirtualQueuesTask is not null)
    {
        var virtualQueues = await getVirtualQueuesTask;
        cachedAggregationsV2 = new(virtualQueues, scenariosUsage, endpointsUsage);
    }

    ReportAggregatorMetrics(scenariosUsage, endpointsUsage);
}
```

**Add `GetVirtualQueuesAsync`:**

```csharp
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

**Add `ModelRoutingAggregationsCacheV2`:**

```csharp
internal sealed record ModelRoutingAggregationsCacheV2(
    IReadOnlyList<VirtualQueue> VirtualQueues,
    IReadOnlyList<ScenarioUsage> ScenariosUsage,
    IReadOnlyList<EndpointUsage> EndpointsUsage);
```

`ModelRoutingAggregationsCache` is unchanged. The V2 cache reuses the same `ScenariosUsage` and `EndpointsUsage` from each polling cycle but replaces the per-request queue data with aggregated `VirtualQueues`. When `EnableVirtualQueueQuery` is `false`, `cachedAggregationsV2` is not refreshed. It remains empty until the first successful V2 fetch, and after that it retains the last successful V2 snapshot unless the implementation explicitly clears it.

### 4.4 gRPC API

**New RPC in `modelrouting.proto`:**

```protobuf
message VirtualQueue {
  string scenarioId = 1;
  int32 priority = 2;
  int32 queuedCount = 3;
  google.protobuf.Timestamp firstCreatedAt = 4;
  google.protobuf.Timestamp lastCreatedAt = 5;
}

message GetModelRoutingAggregationsV2Request {
}

message GetModelRoutingAggregationsV2Response {
  repeated VirtualQueue virtualQueues = 1;
  repeated ScenarioUsage scenariosUsage = 2;
  repeated EndpointUsage endpointsUsage = 3;
}

service ModelRouting {
  // Existing
  rpc GetModelRoutingAggregations (GetModelRoutingAggregationsRequest) returns (GetModelRoutingAggregationsResponse);
  // New — no instanceId needed since virtual queues are instance-agnostic
  rpc GetModelRoutingAggregationsV2 (GetModelRoutingAggregationsV2Request) returns (GetModelRoutingAggregationsV2Response);
}
```

**`ModelRoutingService` (gRPC server) — add handler:**

```csharp
public override Task<External.GetModelRoutingAggregationsV2Response> GetModelRoutingAggregationsV2(
    External.GetModelRoutingAggregationsV2Request request, ServerCallContext context)
{
    var aggregations = modelRoutingAggregator.GetAggregatedViewV2();
    return Task.FromResult(ToExternal(aggregations));
}
```

Same pattern as V1 — reads from the volatile `cachedAggregationsV2` snapshot populated by `ProcessAggregationsCoreAsync`.

### 4.5 Router — `ModelRouterV2`

Create `ModelRouterV2` as a new class implementing `IModelRouter` and `IRequestAllocationManager`. The V2 router replaces `RemoteModelRoutingRequests` with `GlobalVirtualQueues` and uses time-based interpolation for queue position estimation.

**Why a new class instead of branching in `ModelRouter`:**

- `ModelRouter` uses `IReadOnlyList<RemoteModelRoutingRequest>` (per-instance, ordered entries) throughout `GetQueuePosition`, `GetQueuedCount`, and `AllocateQueuedRequests`. The virtual queue model uses `FrozenDictionary<(string ScenarioId, int Priority), VirtualQueue>` representing the global queue snapshot — a fundamentally different data structure and access pattern.
- Queue position estimation changes from exact counting to time-based interpolation — the algorithm itself is different, not just the data source.
- `IModelRoutingState` requires different fields (`GlobalVirtualQueues` vs `RemoteModelRoutingRequests`) and a different sync method (`SyncGlobalState(ModelRoutingAggregationsV2)` vs `SyncWithRemoteState(ModelRoutingAggregations)`).
- Branching every access point in a 730-line class with concurrent state management creates risk disproportionate to the transition period. A separate class keeps both paths clean and testable.
- `VersionedModelRouter` delegates runtime decisions based on the `UseVirtualQueue` flag, while the background service keeps both V1 and V2 state snapshots warm during rollout for safe rollback.

**Queue position estimation — time-based interpolation:**

The core algorithmic change is in `GetGlobalNumberOfRequestsAhead`. Without individual documents, the router cannot count exact queued requests ahead. Instead it interpolates against the global time range:

```csharp
private int GetGlobalNumberOfRequestsAhead(string scenarioId, int priority, DateTimeOffset createdAt)
{
    // PriorityValues: pre-computed array of all priority int values, e.g. [0, 1, 2, 3, 4, 5, 6]
    // Count all queued requests with strictly higher priority (lower priority value)
    int globalRequestsWithHigherPriority = 0;
    foreach (int p in PriorityValues)
    {
        if (p >= priority)
            break;

        if (state.GlobalVirtualQueues.TryGetValue((scenarioId, p), out var virtualQueue))
            globalRequestsWithHigherPriority += virtualQueue.QueuedCount;
    }

    if (!state.GlobalVirtualQueues.TryGetValue((scenarioId, priority), out var priorityQueue))
        return globalRequestsWithHigherPriority;

    // Interpolate within the same priority group assuming uniform distribution (FIFO — earlier requests served first)
    int requestsAheadAtSamePriority = createdAt switch
    {
        _ when createdAt <= priorityQueue.FirstCreatedAt => 0,
        _ when createdAt > priorityQueue.LastCreatedAt => priorityQueue.QueuedCount,
        _ when createdAt == priorityQueue.LastCreatedAt => priorityQueue.QueuedCount - 1,
        _ => Math.Min(
            priorityQueue.QueuedCount - 1,
            (int)Math.Ceiling(
                1.0 * (createdAt - priorityQueue.FirstCreatedAt).Ticks
                    / (priorityQueue.LastCreatedAt - priorityQueue.FirstCreatedAt).Ticks
                    * priorityQueue.QueuedCount))
    };

    return globalRequestsWithHigherPriority + requestsAheadAtSamePriority;
}
```

The interpolation assumes FIFO ordering (earlier requests are ahead), matching V1's `GetQueuePosition` which counts remotes where `r.CreatedAt < createdAt`. The fraction `(createdAt - FirstCreatedAt) / (LastCreatedAt - FirstCreatedAt)` estimates what proportion of queued requests were created before the current request. This is an approximation — it's exact when requests are uniformly distributed and degrades gracefully when they're not, because the queue position only needs to be correct enough to determine if capacity is available (a threshold check, not a ranking).

In V2, queue position is derived from the global snapshot only:

```csharp
private int GetQueuePosition(string scenarioId, int priority, DateTimeOffset createdAt) =>
    1 + GetGlobalNumberOfRequestsAhead(scenarioId, priority, createdAt);
```

This intentionally does not add local queued requests that are not yet visible in Cosmos DB. Newly queued local requests become visible to V2 queue math after their upsert is observed by the next aggregation snapshot.

**Queue-full check:**

```csharp
private int GetQueuedCount(string scenarioId, bool isBackfill)
{
    var globalQueuedCount = 0;
    foreach (int p in PriorityValues)
    {
        if (isBackfill != (p == BackfillPriorityInt))
            continue;

        if (state.GlobalVirtualQueues.TryGetValue((scenarioId, p), out var virtualQueue))
            globalQueuedCount += virtualQueue.QueuedCount;
    }

    return globalQueuedCount;
}
```

This is the simplest reconciliation rule: V2 queue-full checks use the same lagging global snapshot as queue position. That avoids double-counting local requests already reflected in Cosmos DB, at the cost of delayed visibility for the newest local enqueues.

### 4.6 `IModelRoutingState` Changes

Add virtual queue support alongside existing fields:

```csharp
internal interface IModelRoutingState : IDisposable
{
    // ... existing members ...

    // New: Global virtual queues indexed by (scenarioId, priority)
    FrozenDictionary<(string ScenarioId, int Priority), VirtualQueue> GlobalVirtualQueues { get; }

    // New: Sync from V2 aggregations
    void SyncGlobalState(ModelRoutingAggregationsV2 aggregations);
}
```

**`ModelRoutingState.SyncGlobalState`:**

```csharp
public void SyncGlobalState(ModelRoutingAggregationsV2 aggregations)
{
    // Build virtual queue lookup from aggregation
    var newGlobalVirtualQueues = aggregations.VirtualQueues
        .ToFrozenDictionary(
            vq => (vq.ScenarioId, vq.Priority),
            vq => vq);

    GlobalVirtualQueues = newGlobalVirtualQueues;

    // Convert lists to dictionaries for O(1) lookup (matching V1's pattern)
    var scenarioUsageLookup = aggregations.ScenariosUsage
        .ToDictionary(s => s.ScenarioId, StringComparer.Ordinal);
    var endpointUsageLookup = aggregations.EndpointsUsage
        .ToDictionary(e => e.Endpoint, StringComparer.Ordinal);

    // Update scenario and endpoint usage (same as V1)
    ScenarioUsages = ScenarioUsages.Keys.ToFrozenDictionary(
        scenarioId => scenarioId,
        scenarioId => scenarioUsageLookup.GetValueOrDefault(scenarioId, new(scenarioId, 0, 0)),
        StringComparer.Ordinal);

    EndpointUsages = EndpointUsages.Keys.ToFrozenDictionary(
        endpoint => endpoint,
        endpoint => endpointUsageLookup.GetValueOrDefault(endpoint, new(endpoint, 0)),
        StringComparer.Ordinal);
}
```

### 4.7 `ModelRoutingAggregationsClient` Changes

**`IModelRoutingAggregationsClient` — add V2 method:**

```csharp
internal interface IModelRoutingAggregationsClient
{
    Task<ModelRoutingAggregations> GetAggregatedStateAsync(PicassoId instanceId, CancellationToken cancellationToken);  // existing
    Task<ModelRoutingAggregationsV2> GetAggregatedStateV2Async(CancellationToken cancellationToken);                    // new
    Task<IEnumerable<ModelRoutingConfig>> GetModelRoutingConfigsAsync(CancellationToken cancellationToken);              // existing
}
```

**Add V2 client implementation:**

Note: `ModelRoutingAggregationsV2` in the inference namespace (`Picasso.Inference.ModelRouting`) uses `IReadOnlyList` fields — matching the aggregator-side pattern since virtual queues are already keyed by `(ScenarioId, Priority)` and the dictionary conversion happens in `IModelRoutingState.SyncGlobalState`.

```csharp
public async Task<ModelRoutingAggregationsV2> GetAggregatedStateV2Async(CancellationToken cancellationToken)
{
    var aggregations = await modelRoutingGrpcClient.GetModelRoutingAggregationsV2Async(
        new(),
        deadline: timeProvider.GetGrpcCallDeadline(TimeSpan.FromSeconds(5)),
        cancellationToken: cancellationToken);

    var virtualQueues = aggregations.VirtualQueues.Select(ToInternalVirtualQueue).ToArray();
    var scenariosUsage = aggregations.ScenariosUsage.Select(ToInternal).ToArray();
    var endpointsUsage = aggregations.EndpointsUsage.Select(ToInternal).ToArray();

    return new(virtualQueues, scenariosUsage, endpointsUsage);
}

private static VirtualQueue ToInternalVirtualQueue(Picasso.ModelRouting.Api.External.VirtualQueue entry) =>
    new(entry.ScenarioId, entry.Priority, entry.QueuedCount, entry.FirstCreatedAt.ToDateTimeOffset(), entry.LastCreatedAt.ToDateTimeOffset());
```

### 4.8 `ModelRoutingService` (BackgroundService) Changes

The background service on each inference instance syncs state through `IRequestAllocationManager` (resolved to `VersionedModelRouter`). To support safe cutover, sync and allocation are split: the service always refreshes the V1 snapshot, optionally refreshes the V2 snapshot when `EnableVirtualQueueSync` is enabled, and runs allocation once through the currently selected router. Change from `IOptions` to `IOptionsMonitor` to support runtime flag toggling without restart:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    logger.Starting();
    do
    {
        try
        {
            var aggregations = await aggregationsClient.GetAggregatedStateAsync(requestManager.InstanceId, stoppingToken);
            requestManager.SyncAggregations(aggregations);

            if (options.CurrentValue.EnableVirtualQueueSync)
            {
                var aggregationsV2 = await aggregationsClient.GetAggregatedStateV2Async(stoppingToken);
                requestManager.SyncAggregationsV2(aggregationsV2);
            }

            requestManager.AllocateQueuedRequests();

            var configs = await aggregationsClient.GetModelRoutingConfigsAsync(stoppingToken);
            configClient.UpdateModelRoutingConfiguration(configs.ToArray());
        }
        catch (Exception ex) when (ex.IsNotCancelled())
        {
            logger.FailedToProcess(ex);
        }
    }
    while (await processingTimer.WaitForNextTickAsync(stoppingToken));

    logger.Stopping();
}
```

### 4.9 Runtime Switch

Three feature flags control incremental rollout, each gated independently via Azure App Configuration (already supported by the Picasso repo). Toggling any flag takes effect within the refresh interval — no deployment or restart needed.

Flags are split across options classes matching their consumers:

**`ModelRoutingOptions`** (aggregator-side, `Picasso.ModelRouting`):

```csharp
internal sealed record ModelRoutingOptions
{
    // ... existing properties ...

    // Stage 1: Aggregator fetches virtual queue data from Cosmos DB and holds it in memory.
    //          The original aggregation path is unaffected.
    public bool EnableVirtualQueueQuery { get; init; }
}
```

**`ModelRoutingWithAuthOptions`** (inference-side, `Picasso.Inference.ModelRouting`):

```csharp
internal sealed record ModelRoutingWithAuthOptions
{
    // ... existing properties ...

    // Stage 2: Background service fetches and warms the V2 snapshot, but routing decisions stay on V1.
    public bool EnableVirtualQueueSync { get; init; }

    // Stage 3: Router switches from V1 to V2 for acquire/release decisions.
    public bool UseVirtualQueue { get; init; }
}
```

**Rollout order:**

| Stage | Flag | Component | Effect | Risk |
|-------|------|-----------|--------|------|
| 1 | `EnableVirtualQueueQuery` | Aggregator | Queries virtual queue from Cosmos DB, stores in memory. V1 aggregation unchanged. | Low — read-only, no downstream impact |
| 2 | `EnableVirtualQueueSync` | Router background sync | Fetches and warms V2 state on every sync cycle, but leaves live decisions on V1. | Low/Medium — extra gRPC and memory, no decision change |
| 3 | `UseVirtualQueue` | Router | Switches `VersionedModelRouter` from V1 to V2 after both snapshots are warm. | Medium — changes routing decisions |

Each stage can be enabled independently and rolled back by flipping the flag. Stage 2 depends on stage 1 being active (the aggregator must have virtual queue data for the V2 gRPC endpoint to serve). Stage 3 depends on stage 2 being active so the V2 router already has a fresh snapshot before it receives live traffic. Stage 1 can run safely without stages 2 and 3, and stage 2 can run safely without stage 3.

**DI registration** — `VersionedModelRouter` implements both `IModelRouter` and `IRequestAllocationManager`, replacing the direct `ModelRouter` registration for `IRequestAllocationManager`:

```csharp
// Existing keyed registrations (already in AppStartup.cs)
services
    .AddKeyedSingleton<IModelRouter, ModelRouter>("ModelRouter")
    .AddKeyedSingleton<IModelRouter, WeightedModelRouter>("WeightedModelRouter");

// New: register ModelRouterV2 as keyed singleton
services
    .AddKeyedSingleton<IModelRouter, ModelRouterV2>("ModelRouterV2");

// New: register VersionedModelRouter as both IModelRouter and IRequestAllocationManager
// Replaces the existing: .AddSingleton<IRequestAllocationManager, ModelRouter>()
services.AddSingleton<VersionedModelRouter>(sp =>
{
    var v1 = sp.GetRequiredKeyedService<IModelRouter>("ModelRouter");
    var v2 = sp.GetRequiredKeyedService<IModelRouter>("ModelRouterV2");
    var optionsMonitor = sp.GetRequiredService<IOptionsMonitor<ModelRoutingWithAuthOptions>>();
    return new VersionedModelRouter(v1, v2, optionsMonitor);
});
services.AddSingleton<IModelRouter>(sp => sp.GetRequiredService<VersionedModelRouter>());
services.AddSingleton<IRequestAllocationManager>(sp => sp.GetRequiredService<VersionedModelRouter>());
```

**`ModelRoutingLeaseFactory` change** — update to resolve non-keyed `IModelRouter` so it routes through `VersionedModelRouter`:

```csharp
// Before: [FromKeyedServices("ModelRouter")] IModelRouter modelRouter
// After:
internal sealed partial class ModelRoutingLeaseFactory(
    ...
    IModelRouter modelRouter) : IModelRoutingLeaseFactory   // resolves to VersionedModelRouter
```

**`IRequestAllocationManager` — split sync from allocation:**

```csharp
internal interface IRequestAllocationManager
{
    PicassoId InstanceId { get; }
    void SyncAggregations(ModelRoutingAggregations aggregations);      // V1 sync only
    void SyncAggregationsV2(ModelRoutingAggregationsV2 aggregations);  // V2 sync only
    void AllocateQueuedRequests();                                     // delegates to current router
}
```

**`VersionedModelRouter`** — implements both `IModelRouter` and `IRequestAllocationManager`, delegating to V1 and V2 routers:

```csharp
/// <summary>
/// Delegates live routing decisions to V1 or V2 based on the live UseVirtualQueue flag.
/// V1 and V2 state sync are routed explicitly so both snapshots can stay warm
/// during rollout while allocation runs only once through the active router.
/// </summary>
internal sealed class VersionedModelRouter(
    IModelRouter v1,
    IModelRouter v2,
    IOptionsMonitor<ModelRoutingWithAuthOptions> optionsMonitor) : IModelRouter, IRequestAllocationManager
{
    private IModelRouter Current =>
        optionsMonitor.CurrentValue.UseVirtualQueue ? v2 : v1;

    // IModelRouter — delegates based on UseVirtualQueue flag
    public Task<IAcquireResult> AcquireAsync(string scenarioId, InferencePriority priority, TimeSpan? acquireTimeout, CancellationToken cancellationToken) =>
        Current.AcquireAsync(scenarioId, priority, acquireTimeout, cancellationToken);

    public void Release(PicassoId requestId, ModelRoutingUnifiedStatusCode statusCode, TimeSpan duration) =>
        Current.Release(requestId, statusCode, duration);

    // IRequestAllocationManager — V1 sync always goes to V1 router, V2 sync always goes to V2 router
    public PicassoId InstanceId => ((IRequestAllocationManager)v1).InstanceId;

    public void SyncAggregations(ModelRoutingAggregations aggregations) =>
        ((IRequestAllocationManager)v1).SyncAggregations(aggregations);

    public void SyncAggregationsV2(ModelRoutingAggregationsV2 aggregations) =>
        ((IRequestAllocationManager)v2).SyncAggregationsV2(aggregations);

    public void AllocateQueuedRequests() =>
        ((IRequestAllocationManager)Current).AllocateQueuedRequests();
}
```

**Rollback strategy:** Roll back in reverse order by flipping flags in Azure App Configuration. Changes propagate within ≤ 35 seconds (30s refresh interval + up to 5s jitter) — no deployment, no restart.

| Scenario | Action |
|----------|--------|
| V2 routing issues | Set `UseVirtualQueue = false` — reverts live decisions to V1 immediately while `EnableVirtualQueueSync` can stay enabled to keep V2 warm for retry. |
| V2 sync issues | Set `UseVirtualQueue = false`, then set `EnableVirtualQueueSync = false` — stops fetching V2 state on inference instances and returns to V1-only sync. |
| Virtual queue fetch issues (e.g., RU spike) | Set `UseVirtualQueue = false`, set `EnableVirtualQueueSync = false`, then set `EnableVirtualQueueQuery = false` — disables V2 in reverse dependency order. The last successful V2 snapshot may remain cached in memory, so this is safe only when V2 consumers are also disabled or the cache is explicitly cleared. |
| Full rollback | Set all three flags to `false` — system returns to pure V1 behaviour. |

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
| Aggregator cache | All sorted documents | Merged global virtual queues in `cachedAggregationsV2` |
| Per-instance view | Required (`BuildAggregatedQueuedRequests`) | Not needed (virtual queues are instance-agnostic) |
| Queue position | Exact count | Time-based interpolation |
| State structure | `IReadOnlyList<RemoteModelRoutingRequest>` per scenario | Global `VirtualQueue` per (scenario, priority) |

---

## 6. Files Changed

### New Files

| File | Description |
|------|-------------|
| `Shared/ModelRouting/VirtualQueue.cs` | `VirtualQueue` internal record |
| `Shared/ModelRouting/Storage/External/VirtualQueue.cs` | `External.VirtualQueue` Cosmos DB serialization record |
| `Shared/ModelRouting/ModelRoutingAggregationsCacheV2.cs` | `ModelRoutingAggregationsCacheV2` record (or added to existing `ModelRoutingAggregations.cs`) |
| `Shared/ModelRouting/ModelRoutingAggregationsV2.cs` | `ModelRoutingAggregationsV2` record (or added to existing `ModelRoutingAggregations.cs`) |
| `Shared/Inference/ModelRouting/ModelRouterV2.cs` | V2 router with time-based interpolation |
| `Shared/Inference/ModelRouting/VersionedModelRouter.cs` | Delegating router that keeps both sync paths warm and switches live decisions based on `UseVirtualQueue` |

### Modified Files

| File | Change |
|------|--------|
| `Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs` | Add `GetVirtualQueueByFeedRangesAsync` |
| `Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs` | Add query constant, `QueryDefinition`, and implementation |
| `Shared/ModelRouting/Storage/Converters.cs` | Add `ToInternal` for `External.VirtualQueue` |
| `Shared/ModelRouting/Storage/External/SerializationContext.cs` | Add `[JsonSerializable(typeof(External.VirtualQueue))]` |
| `Service.ModelRouting/InternalContracts/IModelRoutingAggregator.cs` | Add `GetAggregatedViewV2` |
| `Service.ModelRouting/Aggregator/ModelRoutingAggregator.cs` | Add `GetAggregatedViewV2Core`, `GetVirtualQueuesAsync`, `EnableVirtualQueueQuery` check in `ProcessAggregationsCoreAsync` |
| `Service.ModelRouting/Api/Services/ModelRoutingService.cs` | Add `GetModelRoutingAggregationsV2` gRPC handler |
| `Service.ModelRouting/Api/Proto/ModelRouting.proto` | Add `VirtualQueue` message, V2 request/response messages, and V2 RPC |
| `Shared/Inference/ModelRouting/IModelRoutingState.cs` | Add `GlobalVirtualQueues` and `SyncGlobalState(ModelRoutingAggregationsV2)` |
| `Shared/Inference/ModelRouting/ModelRoutingState.cs` | Implement `GlobalVirtualQueues` storage and `SyncGlobalState` |
| `Shared/Inference/ModelRouting/IModelRoutingAggregationsClient.cs` | Add `GetAggregatedStateV2Async` and inference-side `ModelRoutingAggregationsV2` record |
| `Shared/Inference/ModelRouting/ModelRoutingAggregationsClient.cs` | Add V2 client method and `ToInternalVirtualQueue` mapper |
| `Shared/Inference/ModelRouting/IRequestAllocationManager.cs` | Split sync from allocation (`SyncAggregations`, `SyncAggregationsV2`, `AllocateQueuedRequests`) |
| `Shared/Inference/ModelRouting/ModelRoutingService.cs` | Change `IOptions` to `IOptionsMonitor`, always sync V1, optionally sync V2 behind `EnableVirtualQueueSync`, then allocate once |
| `Shared/ModelRouting/ModelRoutingOptions.cs` | Add `EnableVirtualQueueQuery` flag |
| `Shared/Inference/ModelRouting/ModelRoutingWithAuthOptions.cs` | Add `EnableVirtualQueueSync` and `UseVirtualQueue` flags |
| `Shared/Inference/ModelRouting/ModelRoutingLeaseFactory.cs` | Change `[FromKeyedServices("ModelRouter")] IModelRouter` to non-keyed `IModelRouter` |
| `Shared/Inference/AppStartup.cs` | Register `ModelRouterV2`, `VersionedModelRouter` as non-keyed `IModelRouter` |

---

## 7. Migration Plan

### Phase 1 — Storage + Aggregator

Add `GetVirtualQueueByFeedRangesAsync` to storage provider. Add `EnableVirtualQueueQuery` flag and virtual queue fetch to `ProcessAggregationsCoreAsync`. Add `GetAggregatedViewV2` and `ModelRoutingAggregationsCacheV2`. Add gRPC V2 endpoint. Deploy with all flags `false`. No production behaviour change.

### Phase 2 — Router V2 + Background Service

Add `ModelRouterV2`, `VersionedModelRouter`, `IModelRoutingState` changes (`GlobalVirtualQueues`, `SyncGlobalState`), `ModelRoutingAggregationsClient` V2 path, and background service dual-sync path with sync/allocation split. Change `ModelRoutingLeaseFactory` to resolve non-keyed `IModelRouter`. Add `EnableVirtualQueueSync` and `UseVirtualQueue` to `ModelRoutingWithAuthOptions`. Deploy with all flags `false`. V2 code is present but inactive — only V1 sync and V1 allocation run when `EnableVirtualQueueSync` and `UseVirtualQueue` are false.

### Phase 3 — Incremental Rollout

Enable flags one at a time per environment, validating each stage before proceeding to the next:

| Step | Flag | What to validate |
|------|------|-----------------|
| 3a | `EnableVirtualQueueQuery = true` | Aggregator queries virtual queue from Cosmos DB successfully. Monitor RU consumption. V1 behaviour unchanged since inference is still V1-only. |
| 3b | `EnableVirtualQueueSync = true` | Inference instances fetch and warm V2 snapshots. Validate gRPC health, memory/cpu overhead, and that allocation still follows V1 because `UseVirtualQueue` is still false. |
| 3c | `UseVirtualQueue = true` | Router switches live decisions to V2. Validate queue position behaviour, queue-full behaviour, allocation behaviour, and RU reduction. |

**Sequence per step:**
1. Staging — validate behaviour and metrics
2. Production

**Rollback:** Flip the problematic flag to `false` via Azure App Configuration. Takes effect within ≤ 35 seconds — no deployment, no restart. Roll back in reverse dependency order if needed (`UseVirtualQueue` → `EnableVirtualQueueSync` → `EnableVirtualQueueQuery`).

### Phase 4 — Cleanup

After V2 is stable in all environments:

1. Remove `ModelRouter` (V1) class and `VersionedModelRouter`
2. Remove `GetQueuedRequestsByFeedRangesAsync` from `IModelRoutingStorageProvider` and implementation
3. Remove `BuildAggregatedQueuedRequests` from aggregator
4. Remove `GetAggregatedView(instanceId)`, `ModelRoutingAggregationsCache`, and V1 gRPC endpoint
5. Remove `RemoteModelRoutingRequests` from `IModelRoutingState`
6. Remove `SyncWithRemoteState(ModelRoutingAggregations)` from state
7. Remove `EnableVirtualQueueQuery`, `EnableVirtualQueueSync`, and `UseVirtualQueue` flags
8. Rename `ModelRouterV2` → `ModelRouter`, `ModelRoutingAggregationsCacheV2` → `ModelRoutingAggregationsCache`

---

## 8. Testing

### Unit Tests

**Storage:**
- `ModelRoutingStorageProvider.GetVirtualQueueByFeedRangesAsync`: verify feed-range fan-out and conversion.
- `Converters.ToInternal(External.VirtualQueue)`: verify field mapping.

**Aggregator:**
- `ModelRoutingAggregator.GetVirtualQueuesAsync`: verify feed-range merging (sum counts, min/max timestamps for same group across feed ranges).
- `GetAggregatedViewV2Core`: verify returns snapshot from `cachedAggregationsV2`.
- `ProcessAggregationsCoreAsync` with `EnableVirtualQueueQuery = true`: verify `cachedAggregationsV2` is populated.
- `ProcessAggregationsCoreAsync` with `EnableVirtualQueueQuery = false`: verify `cachedAggregationsV2` is not refreshed; before first V2 fetch it remains empty, and after a prior successful V2 fetch it retains the last snapshot unless explicitly cleared.

**Router V2:**
- `GetGlobalNumberOfRequestsAhead` interpolation:
  - `createdAt` before `FirstCreatedAt` → returns 0 (oldest, first to be served)
  - `createdAt` after `LastCreatedAt` → returns full `QueuedCount` (newest, everyone ahead)
  - `createdAt` at midpoint → returns ~half of `QueuedCount`
  - Single-element queue (`FirstCreatedAt == LastCreatedAt`) → returns `QueuedCount - 1`
- `GetQueuePosition`: verify it is derived from the global snapshot only and does not add local queued requests on top.
- `GetQueuedCount(scenarioId, isBackfill)`: verify it returns the global queued count with correct backfill filtering.
- Visibility lag scenario: a newly queued local request that is not yet present in the global snapshot should not affect V2 queue position or queue-full checks until the next successful sync.
- `AllocateQueuedRequests`: verify allocation order respects priority.
- Existing `ModelRouter` V1 tests remain unchanged.

**VersionedModelRouter:**
- `SyncAggregations` always updates V1.
- `SyncAggregationsV2` always updates V2.
- `AllocateQueuedRequests` delegates to V2 when `UseVirtualQueue = true`.
- `AllocateQueuedRequests` delegates to V1 when `UseVirtualQueue = false`.
- Runtime flag change → switches allocation implementation on next call without requiring a cold sync.

**Background service:**
- `EnableVirtualQueueSync = false` → V1 snapshot is refreshed, V2 fetch is skipped, and allocation runs once.
- `EnableVirtualQueueSync = true` → V1 and V2 snapshots are both refreshed, and allocation still runs exactly once.

### Staging Validation

- Virtual queue results match expected aggregation from the actual queue query.
- RU reduction consistent with benchmarks.
- Queue position estimation produces allocation behaviour comparable to V1 under normal load.
- No 429 throttling.
- gRPC payload size reduction measurable.
