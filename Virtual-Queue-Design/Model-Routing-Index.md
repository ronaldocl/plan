# Model Routing Codebase Documentation Index

Complete documentation of the Model Routing system on branch `lcao/mr_container_migration_3`.

---

## Documentation Files

### 1. Model-Routing-Summary.md
**High-level overview of the entire system**

Complete inventory of all 13 files in the Model Routing codebase with their purposes:
- Interface and Implementation files
- Data Models
- Background Service
- Configuration and Extensions

Covers:
- Data flow from Cosmos DB through StorageProvider to Aggregator
- Key algorithms (BuildAggregatedQueuedRequests state machine, feed range parallelization)
- Thread safety model (volatile fields, atomic operations)
- Metrics collection (storage and aggregator)
- Configuration options
- Virtual Queue status

**Length**: 186 lines | **When to read**: First—establishes overall system context

---

### 2. Model-Routing-Virtual-Queue.md
**Deep dive into the queue implementation**

Explains that the "virtual queue" is implicit—not an explicit class but a Cosmos query pattern.

Details:
- Queue representation (WHERE endpoint IS NULL)
- Queue access patterns (StorageProvider -> Aggregator -> Consumers)
- Queue lifecycle (enters, polls, leaves via assignment/expiration/deletion)
- Ordering and determinism (3-level sort key)
- Performance characteristics (no locks, parallel queries, 30ms polling)
- Code references with actual implementations

**Length**: 169 lines | **When to read**: Second—understand queue mechanics

---

### 3. Model-Routing-Architecture.md
**Comprehensive technical reference**

Complete system architecture with detailed component breakdown:
- Storage Layer responsibilities and methods
- Aggregation Layer polling and grouping logic
- Service Layer (BackgroundService with PeriodicTimer)
- All data models (Internal, External, Remote, Usage)
- Data flow diagram (text-based)
- Thread safety patterns (volatile, CAS loop, Interlocked.Add)
- Configuration schema
- Metrics definitions
- BuildAggregatedQueuedRequests algorithm with examples
- Performance characteristics

**Length**: 193 lines | **When to read**: Reference—implementation deep dive

---

## Quick Navigation

### I want to understand...

**...the overall system architecture**
→ Read Model-Routing-Summary.md first, then Model-Routing-Architecture.md

**...how the queue works**
→ Read Model-Routing-Virtual-Queue.md

**...how data flows from Cosmos DB to consumers**
→ Model-Routing-Summary.md → Data Flow section

**...the polling mechanism (30ms interval)**
→ Model-Routing-Architecture.md → Service Layer section

**...thread safety (no locks)**
→ Model-Routing-Architecture.md → Thread Safety section

**...the complex grouping algorithm**
→ Model-Routing-Summary.md → BuildAggregatedQueuedRequests section
→ Model-Routing-Virtual-Queue.md → Ordering and Determinism section
→ Model-Routing-Architecture.md → Key Algorithm section

**...metrics and observability**
→ Model-Routing-Summary.md → Metrics section
→ Model-Routing-Architecture.md → Metrics section

**...configuration options**
→ Model-Routing-Summary.md → Configuration section
→ Model-Routing-Architecture.md → Configuration section

---

## Key Concepts

### The Queue is Implicit
```sql
WHERE NOT IS_DEFINED(endpoint) OR IS_NULL(endpoint)
```
No VirtualQueue class exists—the queue is defined by this query.

### 30ms Polling Cycle
```
Every 30ms (configurable: 20ms-1hr):
  1. Query Cosmos for queued requests (parallel feed ranges)
  2. Query Cosmos for usage stats (parallel feed ranges)
  3. Sort queued requests by (ScenarioId, Priority, CreatedAt)
  4. Atomically cache in volatile field
  5. Report metrics
```

### Lock-Free Synchronization
- No locks, mutexes, or semaphores
- Only atomic operations (CAS, Interlocked.Add)
- Volatile field for atomic snapshot reads/writes

### Deterministic Ordering
1. **ScenarioId** (alphabetically)
2. **Priority** (numerically, lower first)
3. **CreatedAt** (timestamp, FIFO)

### State Machine Aggregation
Own requests act as boundary markers. Consecutive requests from other instances 
are aggregated into a single RemoteModelRoutingRequest with a count.

---

## File Locations in Codebase

### Core Files
- **IModelRoutingStorageProvider** → `Shared/ModelRouting/Storage/IModelRoutingStorageProvider.cs`
- **ModelRoutingStorageProvider** → `Shared/ModelRouting/Storage/ModelRoutingStorageProvider.cs`
- **ModelRoutingAggregator** → `Service.ModelRouting/Aggregator/ModelRoutingAggregator.cs`
- **ModelRoutingAggregatorService** → `Service.ModelRouting/Api/ModelRoutingAggregatorService.cs`

### Data Models
- **LocalModelRoutingRequest** (Internal) → `Shared/ModelRouting/ModelRoutingRequest.cs`
- **LocalModelRoutingRequest** (External) → `Shared/ModelRouting/Storage/External/LocalModelRoutingRequest.cs`
- **RemoteModelRoutingRequest** → `Shared/ModelRouting/ModelRoutingRequest.cs`
- **ScenarioUsage** → `Shared/ModelRouting/ScenarioUsage.cs`
- **EndpointUsage** → `Shared/ModelRouting/EndpointUsage.cs`
- **ScenarioEndpointUsage** → `Shared/ModelRouting/Storage/External/ScenarioEndpointUsage.cs`

### Configuration & Utilities
- **ModelRoutingOptions** → `Shared/ModelRouting/ModelRoutingOptions.cs`
- **Converters** → `Shared/ModelRouting/Storage/Converters.cs`
- **CosmosExtensions.ToListAsync** → `Shared/DotNetExtensions/Cosmos/CosmosExtensions.cs`
- **WhenAllAsyncExtensions** → `Shared/DotNetExtensions/Concurrency/WhenAllAsyncExtensions.cs`

---

## Data Flow at a Glance

```
┌─────────────────────────────────┐
│ Cosmos DB                       │
│ modelRoutingRequestsV2          │
│ (partition key: id)             │
└───────────────┬─────────────────┘
                │ Every 30ms
                v
        ┌──────────────────┐
        │ Storage Provider │
        │  (Feed Ranges)   │
        └────────┬─────────┘
                 │
        ┌────────v──────────┐
        │   Aggregator      │
        │  (Sort + Group)   │
        └────────┬──────────┘
                 │
        ┌────────v──────────────┐
        │ Service Consumers     │
        │ (Instance-Specific)   │
        └──────────────────────┘
```

---

## Performance Summary

| Aspect | Value |
|--------|-------|
| Polling Interval | 30ms (configurable) |
| Polling Range | 20ms to 1 hour |
| Query Parallelization | Per feed range (physical partition) |
| Synchronization | Lock-free (atomic operations only) |
| Cache Update Latency | < 1 microsecond (atomic store) |
| Sorting | O(n log n) per 30ms cycle |
| No Locks | ✓ Confirmed |
| TTL-Enabled Cleanup | ✓ Confirmed |
| Observable via Metrics | ✓ Confirmed |

---

## Branch Information

**Branch**: `lcao/mr_container_migration_3`  
**Repository**: `/d/Code/picasso`

All documentation based on this branch's codebase as of April 8, 2026.

