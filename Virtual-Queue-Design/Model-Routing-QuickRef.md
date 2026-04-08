# Model Routing - Quick Reference Card

## One-Sentence Summary
A lock-free, Cosmos DB-backed request distribution system that polls every 30ms, 
sorts requests deterministically, and provides instance-specific aggregated views.

---

## The Queue
**What**: WHERE endpoint IS NULL in Cosmos DB  
**How**: Implicit—no VirtualQueue class  
**Why**: Leverages Cosmos partitioning, no in-memory structure  

---

## The Polling Cycle (30ms Default)

```
GetQueuedRequestsByFeedRangesAsync()  ┐
    → WHERE endpoint IS NULL          │ (parallel)
    → Return List<LocalRequest>       ├→ Sort by (Scenario, Priority, CreatedAt)
                                      │
GetUsageByFeedRangesAsync()           │
    → GROUP BY scenarioId, endpoint   ├→ Aggregate ScenarioUsage + EndpointUsage
    → Return List<Usage>              │
                                      └→ Atomically update cachedAggregations
                                         
                      ↓ (every consumer's next read)
                      
GetAggregatedView(instanceId):
    → Filter out own requests
    → Group consecutive other-instance requests
    → Return List<RemoteRequest> with Count
```

---

## Sorting (3-Level)

```
Primary:   ScenarioId (alphabetically)
Secondary: Priority   (numerically, lower first)
Tertiary:  CreatedAt  (timestamp, FIFO)

Example Order:
1. ChatGPT, Priority 1, 2026-04-08 10:00:00
2. ChatGPT, Priority 1, 2026-04-08 10:00:01
3. ChatGPT, Priority 2, 2026-04-08 10:00:00
4. Davinci, Priority 1, 2026-04-08 10:00:00
```

---

## BuildAggregatedQueuedRequests Algorithm

```
Input:  Sorted list of LocalModelRoutingRequest
Target: InstanceId to view from

Output: List<RemoteModelRoutingRequest>

Logic:
  For each request:
    if request.OwnerInstanceId == target:
      Flush current group (boundary)
      Continue (don't add to output)
    
    else if scenario/priority changed:
      Flush current group
      Create new group
    
    else:
      Increment group count

Result: Consecutive requests from other instances grouped with count
```

---

## Thread Safety

| Component | Mechanism | Method |
|-----------|-----------|--------|
| Cache | Volatile field | Atomic load/store |
| ScenarioUsage | CAS loop | Interlocked.CompareExchange |
| EndpointUsage | Atomic add | Interlocked.Add |

**Key**: No locks, mutexes, or semaphores

---

## Configuration

```
AggregatorPollingInterval: 30ms (default)
Range: 20ms - 1hr
Configurable: Yes (hot-reload via IOptionsMonitor)

Models: Dictionary<ScenarioId, ModelOptions>
  - Endpoints: List<EndpointOptions>
  - Scenarios: List<ScenarioOptions>
```

---

## Data Models at a Glance

### LocalModelRoutingRequest (Internal)
```
Id, ScenarioId, Priority, OwnerInstanceId, Endpoint, CreatedAt
+ AcquireTimeout, TaskCompletionSource (runtime only)
```

### LocalModelRoutingRequest (External/Cosmos)
```
Id, ScenarioId, Priority, OwnerInstanceId, Endpoint, CreatedAt
+ ttl (int seconds)
```

### RemoteModelRoutingRequest (Aggregated)
```
ScenarioId, Priority, CreatedAt, Count
```

### Usage
```
ScenarioUsage:   ScenarioId, ActiveCount, TotalCount
EndpointUsage:   Endpoint, ActiveCount
```

---

## Query Patterns

### Get Queued Requests
```sql
SELECT * FROM modelRoutingRequestsV2 r
WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
```

### Get Usage
```sql
SELECT r.scenarioId, r.endpoint, 
       COUNT(r.endpoint) AS activeCount, 
       COUNT(1) AS totalCount
FROM modelRoutingRequestsV2 r
GROUP BY r.scenarioId, r.endpoint
```

---

## Metrics

| Metric | Type | Tags |
|--------|------|------|
| model_routing.storage_request_charge | Histogram (RU) | operation, scenario_id, partition_key_range_id, status_code |
| model_routing_aggregator.scenario_usage_active_requests | Histogram | inference.scenario |
| model_routing_aggregator.scenario_usage_queued_requests | Histogram | inference.scenario |
| model_routing_aggregator.endpoint_usage_active_requests | Histogram | inference.endpoint |

---

## Request Lifecycle

```
Create Request (Endpoint: null)
    ↓
UpsertRequestAsync(ttl) → Cosmos
    ↓
Poll queries: WHERE endpoint IS NULL
    ↓
Assignment (set Endpoint) OR Expiration (ttl=0) OR Deletion
    ↓
Result: endpoint != null OR not in Cosmos
```

---

## Performance Characteristics

| Aspect | Value |
|--------|-------|
| Polling Interval | 30ms (configurable) |
| Feed Range Queries | Parallel (no sequential waits) |
| Lock Contention | None (lock-free) |
| Cache Latency | <1 microsecond (atomic store) |
| Sorting Complexity | O(n log n) |
| Aggregation Complexity | O(n) |

---

## File Map

```
Shared/ModelRouting/
├── IModelRoutingStorageProvider.cs      (interface)
├── ModelRoutingStorageProvider.cs       (storage layer)
├── ModelRoutingAggregator.cs            (aggregation logic)
├── ModelRoutingOptions.cs               (configuration)
├── ScenarioUsage.cs                     (atomic counter)
├── EndpointUsage.cs                     (atomic counter)
├── ModelRoutingRequest.cs               (LocalRequest, RemoteRequest)
└── Storage/
    ├── Converters.cs
    └── External/
        ├── LocalModelRoutingRequest.cs
        └── ScenarioEndpointUsage.cs

Service.ModelRouting/
├── ModelRoutingAggregatorService.cs     (background service)
└── Aggregator/
    └── ModelRoutingAggregator.cs
```

---

## Key Insights

✅ **No Queue Data Structure**  
The queue is the result of a WHERE clause, not an in-memory collection.

✅ **Lock-Free Concurrency**  
Pure atomic operations—no blocking synchronization.

✅ **Deterministic Ordering**  
Same sort order across all instances and polling cycles.

✅ **Instance-Specific Views**  
Each instance sees requests from other instances but not itself.

✅ **Self-Cleaning**  
TTL field automatically expires old requests from Cosmos.

✅ **Parallel Execution**  
Feed ranges queried concurrently; results merged.

---

## How to Extend

**Add a new scenario**:  
→ Update ModelRoutingOptions.Models configuration

**Change polling interval**:  
→ Update AggregatorPollingInterval in ModelRoutingOptions (hot-reload)

**Track new metric**:  
→ Add histogram/counter to Aggregator Metrics

**Custom aggregation logic**:  
→ Modify BuildAggregatedQueuedRequests state machine

---

## Debug Commands

```bash
# View configuration
grep -r "ModelRoutingOptions" /path/to/config

# Find polling interval
grep -r "AggregatorPollingInterval" /path/to/code

# Find aggregation algorithm
grep -r "BuildAggregatedQueuedRequests" /path/to/code

# Find queue queries
grep -r "WHERE.*endpoint" /path/to/code

# Find atomic operations
grep -r "Interlocked\|volatile" /path/to/code
```

---

## Branch
**`lcao/mr_container_migration_3`** on `/d/Code/picasso`

---

## For More Details
→ See Model-Routing-Index.md for complete documentation guide
→ See Model-Routing-Summary.md for system overview
→ See Model-Routing-Architecture.md for implementation details
→ See Model-Routing-Virtual-Queue.md for queue mechanics

