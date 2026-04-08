# Design: Add `GetVirtualQueueByFeedRangesAsync` to `IModelRoutingStorageProvider`

**Branch:** `lcao/mr_container_migration_3`  
**Date:** 2026-04-07  
**Status:** Draft

---

## Overview

This doc describes the design for adding a new `GetVirtualQueueByFeedRangesAsync` method to `IModelRoutingStorageProvider`. The method computes a **virtual queue** view — a per-scenario summary of queued (unassigned) request counts with timing metadata — by querying the `modelRoutingRequestsV2` Cosmos DB container using the existing feed-range parallelization pattern.

---

## Background

### Current State

`IModelRoutingStorageProvider` currently exposes two feed-range-based read methods:

| Method | Returns | Query Pattern |
|---|---|---|
| `GetQueuedRequestsByFeedRangesAsync` | All individual queued `LocalModelRoutingRequest` records (no endpoint) | `SELECT *` per feed range |
| `GetUsageByFeedRangesAsync` | Per-(scenarioId, endpoint) counts | `GROUP BY scenarioId, endpoint` per feed range |

The aggregator layer (`ModelRoutingAggregator`) then post-processes the raw queued requests to build derived summaries for consumers.

### Motivation

Callers that need a **compact, per-scenario queue summary** — total queued count, priority, and time window (`FirstCreatedAt` / `LastCreatedAt`) — currently have to:

1. Call `GetQueuedRequestsByFeedRangesAsync`, which returns every individual document (potentially thousands of rows across all partitions).
2. Group and aggregate in memory in the service layer.

This is expensive in both RU cost and memory when the queue is large. A Cosmos DB-side `GROUP BY` query can return exactly the needed shape at a fraction of the RU cost, and avoids materializing every document in the service tier.

---

## Proposed Change

### 1. New Types

#### `VirtualQueue` (internal model)

Defined in `Shared/ModelRouting/` alongside the other internal models (`ScenarioEndpointUsage`, `ScenarioUsage`, etc.):

```csharp
// Shared/ModelRouting/VirtualQueue.cs
namespace Picasso.ModelRouting;

internal sealed record VirtualQueue(
    string ScenarioId,
    int Priority,
    int QueuedCount,
    DateTimeOffset FirstCreatedAt,
    DateTimeOffset LastCreatedAt);
```

#### `External.VirtualQueue` (storage external model)

Defined in `Shared/ModelRouting/Storage/External/` — the deserialization target for the Cosmos DB query result:

```csharp
// Shared/ModelRouting/Storage/External/VirtualQueue.cs
namespace Picasso.ModelRouting.Storage.External;

internal sealed record VirtualQueue(
    string ScenarioId,
    int Priority,
    int QueuedCount,
    DateTimeOffset FirstCreatedAt,
    DateTimeOffset LastCreatedAt);
```

Registration in the existing `SerializationContext`:

```csharp
// Shared/ModelRouting/Storage/External/SerializationContext.cs
[JsonSerializable(typeof(VirtualQueue))]   // add this line
internal sealed partial class SerializationContext : JsonSerializerContext;
```

### 2. Interface Addition

```csharp
// Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs
internal interface IModelRoutingStorageProvider : IHealthCheckResource
{
    Task UpsertRequestAsync(LocalModelRoutingRequest request, TimeSpan? timeToLive, CancellationToken cancellationToken);
    Task DeleteRequestAsync(PicassoId requestId, string scenarioId, CancellationToken cancellationToken);
    Task<IReadOnlyList<LocalModelRoutingRequest>> GetQueuedRequestsByFeedRangesAsync(CancellationToken cancellationToken);
    Task<IReadOnlyList<ScenarioEndpointUsage>> GetUsageByFeedRangesAsync(CancellationToken cancellationToken);

    // NEW
    Task<IReadOnlyList<VirtualQueue>> GetVirtualQueueByFeedRangesAsync(CancellationToken cancellationToken);
}
```

### 3. Cosmos DB Query

The new query groups queued requests (no endpoint) by `scenarioId` and `priority`, and computes `COUNT`, `MIN(createdAt)`, and `MAX(createdAt)` — all within a single feed-range-scoped pass:

```sql
SELECT
    r.scenarioId AS scenarioId,
    r.priority   AS priority,
    COUNT(1)     AS queuedCount,
    MIN(r.createdAt) AS firstCreatedAt,
    MAX(r.createdAt) AS lastCreatedAt
FROM modelRoutingRequestsV2 r
WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
GROUP BY r.scenarioId, r.priority
```

**Why this works:**
- The `WHERE` clause mirrors `QueryQueuedRequestsByFeedRange` to restrict to unassigned requests.
- `GROUP BY scenarioId, priority` is safe because the existing composite index `(scenarioId, endpoint)` already covers the filtering predicate; Cosmos DB can satisfy the group-by pass efficiently.
- Per-feed-range execution means results from different physical partitions are independent and then merged in the service layer.

**Post-merge deduplication:** Because the same `(scenarioId, priority)` group can appear in multiple feed ranges, the service layer must sum `QueuedCount` and take the global `MIN(FirstCreatedAt)` / `MAX(LastCreatedAt)` across all feed-range results before returning. This mirrors the same aggregation pattern used in `GetUsageCoreAsync`.

### 4. Implementation in `ModelRoutingStorageProvider`

```csharp
// Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs (additions)

private const string QueryVirtualQueueByFeedRange = """
    SELECT
        r.scenarioId    AS scenarioId,
        r.priority      AS priority,
        COUNT(1)        AS queuedCount,
        MIN(r.createdAt) AS firstCreatedAt,
        MAX(r.createdAt) AS lastCreatedAt
    FROM modelRoutingRequestsV2 r
    WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
    GROUP BY r.scenarioId, r.priority
    """;

private static readonly QueryDefinition VirtualQueueByFeedRangeQueryDefinition =
    new(QueryVirtualQueueByFeedRange);

[Activity("Cosmos.GetVirtualQueueByFeedRanges", Owner.ModelRouting)]
private async Task<IReadOnlyList<VirtualQueue>> GetVirtualQueueByFeedRangesCoreAsync(
    CancellationToken cancellationToken)
{
    var feedRanges = await ModelRoutingRequests.GetFeedRangesAsync(cancellationToken);

    var queueArrays = await feedRanges.WhenAllAsync(
        (feedRange, ct) =>
            ModelRoutingRequests.ToListAsync<External.VirtualQueue>(
                feedRange,
                VirtualQueueByFeedRangeQueryDefinition,
                FeedRangeRequestOptions,
                ct),
        cancellationToken);

    // Merge per-feed-range partial groups into final per-(scenarioId, priority) aggregates.
    return queueArrays
        .SelectMany(static q => q)
        .GroupBy(static q => (q.ScenarioId, q.Priority))
        .Select(static g => new VirtualQueue(
            g.Key.ScenarioId,
            g.Key.Priority,
            g.Sum(static q => q.QueuedCount),
            g.Min(static q => q.FirstCreatedAt),
            g.Max(static q => q.LastCreatedAt)))
        .ToArray();
}
```

The `[Activity]` attribute triggers the Roslyn source generator to emit the public `GetVirtualQueueByFeedRangesAsync` wrapper method (activity tracing, error status, cancellation handling) — exactly as it does for `GetQueuedRequestsByFeedRangesCoreAsync` and `GetUsageByFeedRangesCoreAsync`.

### 5. Converter

Add a simple converter extension in `Converters.cs` for internal use, consistent with existing converters:

```csharp
// Shared/ModelRouting/Storage/Converters.cs (addition)
internal static VirtualQueue ToInternal(this External.VirtualQueue vq) =>
    new(vq.ScenarioId, vq.Priority, vq.QueuedCount, vq.FirstCreatedAt, vq.LastCreatedAt);
```

> **Note:** Because the merge aggregation (sum counts, min/max timestamps) cannot be expressed as a single `.ToInternal()` call, the group projection is the natural place to do it inline. The converter is still useful for mapping a single raw external row to an internal record if needed elsewhere.

### 6. Logging

Add a new log method to `Log.cs` in the storage layer, following the existing pattern:

```csharp
// Shared/ModelRouting/Storage/Log.cs (addition)
[LoggerMessage(Level = LogLevel.Error, Message = "Failed to get virtual queue by feed ranges.")]
internal static partial void FailedToGetVirtualQueueByFeedRanges(
    this ILogger<ModelRoutingStorageProvider> logger, Exception ex);
```

The `[Activity]` source generator automatically emits the error log call for unhandled exceptions, but an explicit log message is provided here for consistency with the rest of the file.

---

## File Changelist

| File | Change |
|---|---|
| `Shared/ModelRouting/VirtualQueue.cs` | **New** — internal `VirtualQueue` record |
| `Shared/ModelRouting/Storage/External/VirtualQueue.cs` | **New** — external deserialization record |
| `Shared/ModelRouting/Storage/External/SerializationContext.cs` | **Edit** — register `VirtualQueue` |
| `Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs` | **Edit** — add `GetVirtualQueueByFeedRangesAsync` |
| `Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs` | **Edit** — add query constant, `QueryDefinition`, and `*CoreAsync` method |
| `Shared/ModelRouting/Storage/Converters.cs` | **Edit** — add `ToInternal` for `External.VirtualQueue` |
| `Shared/ModelRouting/Storage/Log.cs` | **Edit** — add `FailedToGetVirtualQueueByFeedRanges` |

---

## Design Decisions

### Why a new method instead of reusing `GetQueuedRequestsByFeedRangesAsync`?

`GetQueuedRequestsByFeedRangesAsync` returns every individual document. For a busy scenario with tens of thousands of queued items, this means materializing every `LocalModelRoutingRequest` record in memory just to group and count them. The new method pushes the aggregation into Cosmos DB, returning only one row per `(scenarioId, priority)` group per feed range — orders of magnitude fewer bytes over the wire and lower RU cost.

### Why not merge this into `GetUsageByFeedRangesAsync`?

`GetUsageByFeedRangesAsync` groups by `(scenarioId, endpoint)` and is used to report active-request and total-request counts for the load-balancing path. The virtual queue query groups by `(scenarioId, priority)` and includes `MIN`/`MAX` timestamp aggregates. Combining them would require a more complex query with outer joins or a union, increasing query complexity and coupling two unrelated read paths.

### Why keep the per-feed-range merge in the service layer?

Cosmos DB's feed-range-scoped queries are inherently partition-local: `GROUP BY` operates within a single physical partition's key range. There is no native cross-partition `GROUP BY` that can be done in a single query pass without a cross-partition query (which would bypass the feed-range parallelization pattern entirely and revert to a global fan-out). The existing `GetUsageByFeedRangesAsync` uses the same in-process merge strategy, so this is consistent with the established pattern.

### Why does `VirtualQueue` include `Priority`?

Priority determines queue ordering and directly affects dispatch behavior. Callers that consume the virtual queue to make routing decisions (e.g., to determine how many high-priority requests are waiting) need `Priority` to be a first-class grouping key. Without it, the count for a single `scenarioId` would aggregate across all priority levels, losing information needed for priority-aware scheduling.

---

## Sequence Diagram

```
Caller
  │
  ├─► GetVirtualQueueByFeedRangesAsync(ct)          [generated wrapper: Activity tracing]
  │     │
  │     ├─► GetFeedRangesAsync(ct)                  [Cosmos SDK: get logical partition ranges]
  │     │     └─► [ feedRange₁, feedRange₂, ... ]
  │     │
  │     ├─► WhenAllAsync (parallel per feed range)
  │     │     ├─► ToListAsync<External.VirtualQueue>(feedRange₁, query, opts, ct)
  │     │     ├─► ToListAsync<External.VirtualQueue>(feedRange₂, query, opts, ct)
  │     │     └─► ...
  │     │
  │     └─► In-process merge (GroupBy + Sum/Min/Max)
  │           └─► IReadOnlyList<VirtualQueue>
  │
  └─◄ IReadOnlyList<VirtualQueue>
```

---

## No-Queue Baseline (Queue = 0%, Item Count = 60K)

| Scenario | Avg Latency (ms) | RU/FR/req |
|---|---|---|
| QR Q=0%  | 13.83 | 3.09  |
| VQ Q=0%  | 14.82 | 3.09  |
| QR Q=10% | 35.57 | 20.91 |
| VQ Q=10% | 23.52 | 9.48  |

At zero queue depth both methods are equivalent. Under 10% queue load, VQ reduces latency by ~34% and RU/FR/req by ~55% compared to QR.

---

## Performance Data (Queue = 10%, 8 Feed Ranges)

| Item Count | Avg Lat QR (ms) | Avg Lat VQ (ms) | Δ Avg Lat | P50 QR | P50 VQ | P90 QR | P90 VQ | P99 QR | P99 VQ | RU/FR/req QR | RU/FR/req VQ | Δ RU/FR/req | VQ saving % |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 10K  | 17.67 | 19.10 | +1.43 | 14 | 16 | 33 | 27 | 44  | 75  | 6.0200  | 7.5954  | +1.5754 | -26.2% |
| 20K  | 20.87 | 20.42 | -0.45 | 18 | 18 | 28 | 25 | 69  | 91  | 9.1013  | 7.7597  | -1.3416 | 14.7%  |
| 30K  | 23.66 | 21.06 | -2.60 | 21 | 19 | 30 | 27 | 64  | 42  | 12.0816 | 8.8037  | -3.2779 | 27.1%  |
| 40K  | 28.13 | 23.38 | -4.75 | 25 | 19 | 35 | 36 | 86  | 99  | 14.8990 | 8.2012  | -6.6978 | 44.9%  |
| 50K  | 30.23 | 21.79 | -8.44 | 28 | 20 | 40 | 30 | 66  | 36  | 17.9025 | 8.7526  | -9.1499 | 51.1%  |
| 60K  | 35.57 | 23.52 | -12.05 | 32 | 20 | 48 | 37 | 65  | 48  | 20.9088 | 9.4795  | -11.4293 | 54.7% |
| 70K  | 36.49 | 23.04 | -13.45 | 36 | 20 | 45 | 34 | 50  | 43  | 23.8575 | 10.1416 | -13.7159 | 57.5% |
| 80K  | 40.65 | 23.67 | -16.98 | 38 | 20 | 47 | 37 | 79  | 57  | 26.7413 | 11.0011 | -15.7402 | 58.9% |
| 90K  | 44.06 | 24.05 | -20.01 | 43 | 21 | 53 | 37 | 92  | 67  | 29.6727 | 11.7836 | -17.8891 | 60.3% |
| 100K | 47.30 | 24.85 | -22.45 | 44 | 21 | 60 | 34 | 82  | 159 | 32.5751 | 12.5375 | -20.0376 | 61.5% |

Δ Avg Lat = VQ − QR (negative = VQ is faster). Δ RU/FR/req = VQ − QR per feed range per request. VQ saving % = RU reduction relative to QR (negative = VQ costs more).

---

## Open Questions

1. **MaxItemCount:** The current `FeedRangeRequestOptions` sets `MaxItemCount = 4000`. For the virtual queue query, the result set is bounded by the number of distinct `(scenarioId, priority)` groups — likely very small (< 100). Should a separate `QueryRequestOptions` with a smaller `MaxItemCount` be used? Probably not worth the complexity; the existing option is fine.

2. **Index coverage:** The WHERE clause `NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)` is already used by `QueryQueuedRequestsByFeedRange` and the existing individual index on `endpoint` should cover it. No new index changes are anticipated, but this should be confirmed when the query is tested against production data volumes.

3. **Consumer:** This doc only covers the storage contract. A follow-up change will wire `GetVirtualQueueByFeedRangesAsync` into the aggregator and/or a new API endpoint — that design is out of scope here.
