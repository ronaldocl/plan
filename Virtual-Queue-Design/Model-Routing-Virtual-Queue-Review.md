# Review: Model Routing Virtual Queue Design

Reviewed document: [Model-Routing-Virtual-Queue.md](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md)

## Findings

### 1. Critical: V2 queue math mixes exact local state with a lagging global aggregate without defining reconciliation

The V1 design explicitly builds a per-instance view where the caller's own requests are excluded and used as group boundaries, so the router can safely add local queued requests to remote queued requests when computing queue-full and queue-position logic. See [Model-Routing-Virtual-Queue.md#L54](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L54).

The V2 design removes `instanceId` from the RPC and describes the virtual queues as instance-agnostic global aggregates. See [Model-Routing-Virtual-Queue.md#L357](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L357) and [Model-Routing-Virtual-Queue.md#L738](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L738).

But the proposed router code still adds `localQueuedCount + remoteQueuedCount` in `GetQueuedCount`, while `remoteQueuedCount` is now derived from the global aggregate rather than a remote-only view. See [Model-Routing-Virtual-Queue.md#L432](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L432).

Because the global queue is produced from Cosmos DB, some local requests may already be reflected in that snapshot while newer local requests may still be missing due to upsert and query latency. The design does not define how the router reconciles that overlap, so queue-full and queue-position calculations can overcount or undercount depending on which local requests are already present in the global aggregate.

One simple resolution is to make V2 queue math rely on the Cosmos-derived global aggregate only and stop adding local in-memory queued requests on top. That avoids overlap and reconciliation logic entirely, but it changes the semantics: queue-full and queue-position decisions will lag behind the newest local enqueues until those requests become visible in Cosmos DB and in the next aggregation poll.

### 2. High: The runtime switch is not actually safe for immediate cutover

The background sync loop fetches either V1 or V2 aggregations based on the live `UseVirtualQueue` flag. See [Model-Routing-Virtual-Queue.md#L548](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L548).

`VersionedModelRouter` also resolves its active implementation from that same live flag on each call. See [Model-Routing-Virtual-Queue.md#L672](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L672).

That means a flag flip can route acquire/release calls to V2 before V2 state has been synced, or back to V1 after V1 state has gone stale. This contradicts the doc's claim that rollback is immediate and both routers remain warm with no functional penalty. See [Model-Routing-Virtual-Queue.md#L693](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L693) and [Model-Routing-Virtual-Queue.md#L697](/d:/Code/plan/Virtual-Queue-Design/Model-Routing-Virtual-Queue.md#L697).

The design needs either dual-sync during the rollout period or an explicit readiness barrier so a router version is not selected until its corresponding state snapshot is known to be fresh.

The simplest mitigation is to keep both V1 and V2 state snapshots warm on every sync cycle during rollout and use `UseVirtualQueue` only to choose which router handles decisions. To make that safe, the design should split remote-state sync from allocation, because dual-syncing `ProcessAggregationsAndAllocate` and `ProcessAggregationsAndAllocateV2` would otherwise run `AllocateQueuedRequests` twice. In practice, each router version should have a sync-only entry point, and only the currently selected router should run allocation after both snapshots are refreshed. If you want rollout control over the extra V2 fetch, add a separate flag such as `EnableVirtualQueueSync` so V2 state warm-up is independent from the live routing switch. That makes cutover and rollback operationally credible at the cost of one extra gRPC fetch and some temporary duplicate state.

Example shape:

```csharp
internal interface IRequestAllocationManager
{
    PicassoId InstanceId { get; }
    void SyncRemoteState(ModelRoutingAggregations aggregations);
    void SyncRemoteStateV2(ModelRoutingAggregationsV2 aggregations);
    void AllocateQueuedRequests();
}
```

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    do
    {
        try
        {
            var aggregationsV1 = await aggregationsClient.GetAggregatedStateAsync(requestManager.InstanceId, stoppingToken);
            requestManager.SyncRemoteState(aggregationsV1);

            if (options.CurrentValue.EnableVirtualQueueSync)
            {
                var aggregationsV2 = await aggregationsClient.GetAggregatedStateV2Async(stoppingToken);
                requestManager.SyncRemoteStateV2(aggregationsV2);
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
}
```

In this model, `VersionedModelRouter` always sends V1 sync to the V1 router, sends V2 sync to the V2 router only when `EnableVirtualQueueSync` is enabled, and delegates `AllocateQueuedRequests()` only to the currently active router selected by `UseVirtualQueue`.
