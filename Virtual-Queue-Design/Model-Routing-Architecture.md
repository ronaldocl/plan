# Model Routing Architecture - Complete Reference

## Overview

The Model Routing system is a highly efficient, lock-free request distribution 
mechanism that routes inference requests to endpoints based on scenarios and priorities.

---

## System Components

### 1. Storage Layer

File: Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs

Responsibilities:
- CRUD operations on Cosmos DB modelRoutingRequestsV2
- Parallel feed range query execution
- Request charge (RU) tracking by partition key range

Key Methods:
- UpsertRequestAsync(LocalModelRoutingRequest, TimeSpan ttl)
- DeleteRequestAsync(PicassoId, ScenarioId)
- GetQueuedRequestsByFeedRangesAsync()
- GetUsageByFeedRangesAsync()

Queries:
1. Queued: SELECT * FROM r WHERE endpoint IS NULL
2. Usage: GROUP BY scenarioId, endpoint with counts

---

### 2. Aggregation Layer

File: Service.ModelRouting/Aggregator/ModelRoutingAggregator.cs

Responsibilities:
- Periodic polling (30ms default)
- Sorting and grouping requests
- Atomic caching of aggregations
- Instance-specific view filtering

Sorting Precedence:
1. ScenarioId (alphabetically)
2. Priority (numerically, lower first)
3. CreatedAt (timestamp, FIFO)

---

### 3. Service Layer

File: Service.ModelRouting/Api/ModelRoutingAggregatorService.cs

Type: BackgroundService with PeriodicTimer

Features:
- Hot-reload support via IOptionsMonitor
- Configurable polling interval (20ms-1hr)
- Resilient exception handling

---

## Data Models

### LocalModelRoutingRequest (Internal)
Fields: Id, ScenarioId, Priority, OwnerInstanceId, Endpoint, CreatedAt
Runtime (not persisted): AcquireTimeout, TaskCompletionSource

### LocalModelRoutingRequest (External/Cosmos)
Fields: Id, ScenarioId, Priority, OwnerInstanceId, Endpoint, CreatedAt
Storage: ttl (int seconds for Cosmos TTL)

### RemoteModelRoutingRequest (Aggregated View)
Fields: ScenarioId, Priority, CreatedAt, Count (aggregated count)

---

## Data Flow

Cosmos DB (every 30ms)
  -> ModelRoutingStorageProvider
       (parallel feed range queries)
  -> ModelRoutingAggregator
       (sort, group, atomic cache)
  -> Service Consumers
       (instance-specific aggregated view)

---

## Thread Safety

### Volatile Field Pattern
```
private volatile ModelRoutingAggregationsCache cachedAggregations;
```

Atomic snapshot read: immediate load barrier
Atomic update: immediate store barrier

### ScenarioUsage (CAS Loop)
Uses Interlocked.CompareExchange with retry until success.
Packs two 32-bit ints into single 64-bit long.

### EndpointUsage (Atomic Add)
Direct Interlocked.Add(ref activeCount, delta)

---

## Configuration

ModelRoutingOptions:
- AggregatorPollingInterval: 30ms default (range: 20ms-1hr)
- ConfigFetcherPollingInterval: 3 minutes default
- Models: Dictionary of endpoint and scenario specifications

---

## Metrics

### Storage Metrics
Meter: model_routing.storage_request_charge
Tags:
- model_routing.operation (UpsertRequest, DeleteRequest)
- model_routing.scenario_id
- model_routing.partition_key_range_id
- model_routing.status_code

### Aggregator Metrics
- model_routing_aggregator.scenario_usage_active_requests
- model_routing_aggregator.scenario_usage_queued_requests
- model_routing_aggregator.endpoint_usage_active_requests

---

## Key Algorithm: BuildAggregatedQueuedRequests

State machine for grouping requests by instance:

For each request in sorted list:

1. If own request (OwnerInstanceId == target):
   - Flush current group
   - Use as boundary marker

2. If different scenario/priority:
   - Flush current group
   - Start new group

3. If same scenario/priority:
   - Increment count

Example:
Input:  [A(Chat,1), B(Chat,1), B(Chat,1), A(Chat,1), B(Davinci,2)]
Target: A
Output: [Remote(Chat,1,Count=2), Remote(Davinci,2,Count=1)]

---

## Performance Characteristics

### Lock-Free Design
- No synchronization primitives
- Atomic operations only (CAS, Interlocked)
- No memory contention

### Parallel Query Execution
Feed ranges processed in parallel:
- Feed Range 1: 500 requests
- Feed Range 2: 300 requests
- Feed Range 3: 200 requests
- Feed Range 4: 150 requests
(all concurrent -> merge -> 1150 total)

### 30ms Polling Interval
- Configurable from 20ms to 1 hour
- Launches two parallel tasks (GetQueued + GetUsage)
- Atomic volatile field update: O(1) operation

---

## Summary

The Model Routing system achieves high-performance through:

- Lock-free design (atomic operations only)
- Implicit queue (WHERE endpoint IS NULL in Cosmos)
- Parallel feed range query execution
- Fast polling (30ms default interval)
- Deterministic ordering (ScenarioId, Priority, CreatedAt)
- Hot reload support (configuration changes without restart)
- Self-cleaning (TTL-based automatic expiration)
- Observable (comprehensive RU and request count metrics)

