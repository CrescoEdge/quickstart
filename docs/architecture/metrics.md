# Metrics & Measurements

Cresco has a single, unified measurement model: **every bundle** (the controller and every plugin)
registers its metrics in a Micrometer registry and exposes them through one contract, and the controller
aggregates them locally, region-wide, and global-wide. This is distinct from [health](health.md), which
consumes measurements to form pass/fail verdicts.

## The model

| Piece | Role |
|-------|------|
| `MeasurementEngine` | per-bundle Micrometer registry; register timers/gauges/counters/distribution-summaries; `getAllMetrics()` returns `Map<group, List<Map<String,String>>>` |
| `CMetric` | metadata wrapper for one metric (name, group, type, node/app/edge scope) |
| `CrescoMeterRegistry` | Micrometer (Dropwizard-backed) registry integrated with Cresco reporting |

Metrics are organized into **groups** — the metric analogue of health tags. The controller populates
`jvm`, `processor`, `controller`, `netlink`, `regional`, and `global`; each plugin contributes its own
(e.g. `sysinfo` → `processor`/`memory`/`disk`/`network`, `wsapi` → `wsapi`, `stunnel` → `stunnel`).

## The single plugin contract

Every bundle answers the same EXEC action:

```java
case "getmetrics":
    incoming.setParam("metrics", gson.toJson(measurementEngine.getAllMetrics()));
    incoming.setParam("status", "10");
    return incoming;
```

`getAllMetrics()` returns `group → list of {name, class, type, value|count|mean|max|...}` — the wire schema
for the whole fabric.

## Mesh-wide aggregation

`getmetricinventory` is the single aggregating surface (metrics' analogue of the health executor). It
merges the controller's own metrics + every local plugin's `getmetrics` + a resource summary, and can
**fan out** by scope:

| Scope | Returns |
|-------|---------|
| `node` | this node |
| `region` | this node + its child agents |
| `global` | the whole mesh |

Both [client libraries](../clients/overview.md) expose `get_metric_inventory(...)` to pull this. The legacy
`getsysinfo`/`resourceinfo`/KPI paths are retained for back-compat and folded in as Micrometer gauges.

## Network link metrics

The `netlink` group carries **per-edge link quality** (`LinkMetrics` / `LinkMetricsRegistry`), measured
from signals the fabric already pays for:

| Signal | Source |
|--------|--------|
| RTT + jitter (Jacobson/Karels smoothed) | the existing parent ping/pong RPC, timed |
| tx/rx throughput, producer send-latency | passive instrumentation of the send path |
| broker backlog | ActiveMQ `QueueSize` via JMX |
| NIC capacity ceiling | `sysinfo` link speed |

These feed the [`link:quality`](health.md) health check, an **auto-tuner** control loop (`AutoTuner`) that
adapts socket buffers, block sizes, and bridge connection counts to the bandwidth-delay product, and a
composite link **cost** model intended for performance-aware routing (see the link-metrics and
optimal-routing design docs via [Design Docs](../reference/design-docs.md)).

## Benchmarks

On-demand CPU benchmarking (SciMark2) is available via `sysinfo`'s `getbenchmark` action — deliberately not
run at startup, to keep node bring-up cheap.
