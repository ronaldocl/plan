# Model Routing Codebase - Complete Understanding

## Branch
`lcao/mr_container_migration_3`

---

## Files Located

### Interface & Implementation
1. **IModelRoutingStorageProvider** → `Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs`
   - UpsertRequestAsync, DeleteRequestAsync
   - GetQueuedRequestsByFeedRangesAsync, GetUsageByFeedRangesAsync

2. **ModelRoutingStorageProvider** → `Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs`
   - Queries: Queued requests (WHERE endpoint IS NULL)
   - Queries: Usage aggregation (GROUP BY scenarioId, endpoint)
   - Feed range parallelization via WhenAllAsync
   - RU tracking by partition key range

3. **ModelRoutingAggregator** → `Service.ModelRouting/Aggregator/ModelRoutingAggregator.cs`
   - ProcessAggregationsAsync: Main polling loop
   - GetQueuedRequests: Fetches + sorts by (ScenarioId, Priority, CreatedAt)
   - GetUsage: Aggregates by scenario and endpoint
   - GetAggregatedView: Instance-specific filtered view
   - BuildAggregatedQueuedRequests: Complex grouping algorithm

### Data Models
4. **LocalModelRoutingRequest (Internal)** → `Shared/ModelRouting/ModelRoutingRequest.cs`
   - Fields: Id, ScenarioId, Priority, OwnerInstanceId, Endpoint, CreatedAt
   - Also: AcquireTimeout, TaskCompletionSource (not persisted)

5. **LocalModelRoutingRequest (External/Cosmos)** → `Shared/ModelRouting/Storage/External/LocalModelRoutingRequest.cs`
   - Same fields minus AcquireTimeout/TCS
   - Has: ttl (int seconds for Cosmos TTL)

6. **RemoteModelRoutingRequest** → `Shared/ModelRouting/ModelRoutingRequest.cs`
   - Aggregated view: ScenarioId, Priority, CreatedAt, Count

7. **Converters** → `Shared/ModelRouting/Storage/Converters.cs`
   - ToInternal: External → Internal (clears runtime fields)
   - ToExternal: Internal → External (converts TimeSpan to seconds)

8. **ScenarioUsage, EndpointUsage** → `.../ScenarioUsage.cs`, `.../EndpointUsage.cs`
   - Atomic updates via CAS loop (ScenarioUsage) or Interlocked.Add

9. **ScenarioEndpointUsage** → `Storage/External/ScenarioEndpointUsage.cs`
   - Cosmos query result (struct)

### Background Service
10. **ModelRoutingAggregatorService** → `Service.ModelRouting/Api/ModelRoutingAggregatorService.cs`
    - BackgroundService with PeriodicTimer
    - Polling interval: 30ms (default, 20ms-1hr)
    - Hot-reload support

### Configuration & Extensions
11. **ModelRoutingOptions** → `Shared/ModelRouting/ModelRoutingOptions.cs`
    - AggregatorPollingInterval
    - ConfigFetcherPollingInterval
    - Models configuration (endpoint + scenario specs)

12. **CosmosExtensions.ToListAsync** → `Shared/DotNetExtensions/Cosmos/CosmosExtensions.cs`
    - Executes queries against specific feed ranges
    - Returns flattened list of results

13. **WhenAllAsyncExtensions** → `Shared/DotNetExtensions/Concurrency/WhenAllAsyncExtensions.cs`
    - Parallelizes tasks: `feedRanges.WhenAllAsync((range, ct) => ...)`

---

## Data Flow

```
Cosmos DB (modelRoutingRequestsV2)
  ↑
  │ Every 30ms via ModelRoutingStorageProvider
  │
  ├─ GetQueuedRequestsByFeedRangesAsync()
  │  └─ Query: SELECT * WHERE endpoint IS NULL
  │  └─ Parallel across all feed ranges
  │  └─ Results → sorted by (ScenarioId, Priority, CreatedAt)
  │
  └─ GetUsageByFeedRangesAsync()
     └─ Query: GROUP BY scenarioId, endpoint
     └─ Parallel across all feed ranges
     └─ Results → aggregated by scenario & endpoint

  ↓ (Cached atomically in volatile field)

ModelRoutingAggregator
  ├─ cachedAggregations (volatile):
  │  ├─ SortedQueuedRequests
  │  ├─ ScenariosUsage
  │  └─ EndpointsUsage
  │
  └─ GetAggregatedView(instanceId)
     └─ Filters out own requests
     └─ Groups consecutive requests from other instances
```

---

## Key Algorithms

### BuildAggregatedQueuedRequests (Complex Grouping)

State machine processes sorted requests:

1. **Own request**: Flushes current group (boundary marker)
2. **Different scenario/priority**: Flushes current, starts new group
3. **Same scenario/priority**: Increments count

Example:
```
Input: [A(ChatGPT,1), B(ChatGPT,1), B(ChatGPT,1), A(ChatGPT,1), B(Davinci,2)]
Target: A

Output: [RemoteRequest(ChatGPT,1,T2,Count=2), RemoteRequest(Davinci,2,T5,Count=1)]
```

Own requests act as boundaries; consecutive requests from other instances are aggregated.

### Feed Range Query Parallelization

```csharp
var feedRanges = await container.GetFeedRangesAsync();
var results = await feedRanges.WhenAllAsync(
    (range, ct) => container.ToListAsync<T>(range, query, ...),
    cancellationToken);
return results.SelectMany(r => r).ToArray();
```

- Gets all physical partitions
- Executes query on each in parallel
- Flattens results

---

## Thread Safety

1. **Volatile Field**: `cachedAggregations` volatile field captures atomic snapshot
2. **Atomic Updates**: ScenarioUsage uses CAS loop; EndpointUsage uses Interlocked.Add
3. **No Locks**: Pure interlocked operations, no blocking synchronization

---

## Metrics

### Storage Metrics (model_routing.storage_request_charge)
- Operation: UpsertRequest, DeleteRequest
- ScenarioId
- PartitionKeyRangeId
- StatusCode

### Aggregator Metrics
- model_routing_aggregator.scenario_usage_active_requests
- model_routing_aggregator.scenario_usage_queued_requests
- model_routing_aggregator.endpoint_usage_active_requests

---

## Configuration

```csharp
ModelRoutingOptions {
  AggregatorPollingInterval: 30ms (min 20ms, max 1hr)
  ConfigFetcherPollingInterval: 3 minutes
  Models: IReadOnlyDictionary<string, ModelOptions>
}
```

---

## VirtualQueue Status

**Not Found.** Queue is represented implicitly:
- Queued requests: LocalModelRoutingRequest where Endpoint == null
- Cache: ModelRoutingAggregationsCache.SortedQueuedRequests
- Aggregated view: RemoteModelRoutingRequest (count-based)

---

## Summary

The Model Routing system polls Cosmos DB every 30ms across all feed ranges in parallel, caches aggregations atomically, and provides instance-specific views with consistent, deterministic sorting. It uses no locks—only atomic operations for thread safety.
