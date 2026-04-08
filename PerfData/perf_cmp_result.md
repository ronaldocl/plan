Perf comparison for the below 2 queries:

private const string QueryQueuedRequestsByFeedRange =
    """
    SELECT * FROM modelRoutingRequests r
    WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
    """;
        
private const string QueryVirtualQueueByFeedRange =
    """
    SELECT
        r.scenarioId AS scenarioId,
        r.priority AS priority,
        COUNT(1) AS queuedCount,
        MIN(r.createdAt) AS firstCreatedAt,
        MAX(r.createdAt) AS lastCreatedAt
    FROM modelRoutingRequests r
    WHERE NOT IS_DEFINED(r.endpoint) OR IS_NULL(r.endpoint)
    GROUP BY r.scenarioId, r.priority
    """;

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