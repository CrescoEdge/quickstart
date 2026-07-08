# sysinfo — System Info & Benchmark

The `sysinfo` plugin (bundle `io.cresco.sysinfo`) gathers live operating-environment metrics for an agent — CPU, memory, disk, filesystem, and network — via [OSHI](https://github.com/oshi/oshi), and can run an on-demand [SciMark2](https://math.nist.gov/scimark2/) CPU benchmark. It is the resource-information provider the [controller](controller.md)'s `PerfControllerMonitor` queries.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/sysinfo` |
| Bundle symbolic name | `io.cresco.sysinfo` |
| Java files | 7 |
| Loaded | At runtime by the controller's `StaticPluginLoader`. |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Metric source | OSHI (system) + SciMark2 (CPU benchmark). |

## Key classes

| Class | Role |
|-------|------|
| `ExecutorImpl` | Handles the EXEC actions (below). |
| `SysInfoBuilder` | `getSysInfoMap()` assembles `os` / `cpu` / `mem` / `disk` / `fs` / `part` / `net` JSON from OSHI. `getCPUInfo` sleeps ~1s for a load delta; the `fs` section is skipped on Raspbian. |
| `PerfSysMonitor` | Continuous sampler — a `Timer` that publishes a live-metrics `MapMessage` to `TopicType.AGENT` on the [data plane](../architecture/dataplane.md) every `perftimer` ms. |
| `Benchmark` | `bench()` runs the five SciMark2 kernels (FFT, SOR, Monte Carlo, Sparse Matmult, LU); `getJSON()` flattens the result. |
| `BenchMetric` | Immutable benchmark-result holder. |

## Executor actions

`ExecutorImpl` handles these EXEC actions ([`MsgEvent`](../api/msgevent.md) type `EXEC`):

| Action | Behaviour |
|--------|-----------|
| `getsysinfo` | Return a compressed `perf` param — the live `SysInfoBuilder.getSysInfoMap()`. This is what `PerfControllerMonitor` RPCs for resource info. |
| `getbenchmark` | Return a compressed `bench` param — `Benchmark.getJSON()`, running SciMark2 **on demand**. |

## Data-plane publishing and the on-demand benchmark

When `enable_perf` is set, `PerfSysMonitor` streams a live-metrics message to `TopicType.AGENT` every `perftimer` ms, feeding the [data plane](../architecture/dataplane.md) and, through it, dashboards and CEP.

!!! note "SciMark2 is on-demand only"
    The SciMark2 benchmark is deliberately **not** run in the continuous sampler. It was previously burned at startup and bundled into every heartbeat; it is now executed only when a `getbenchmark` action is received. This keeps steady-state telemetry cheap. See [Metrics & Measurements](../architecture/metrics.md).

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `enable_perf` | `false` | Run the continuous live-metrics sampler. |
| `perftimer` | `10000` | Sampler interval (ms). |
| `pluginname` | — | This plugin's name. |

See [Configuration](../getting-started/configuration.md).

## Where it fits

The [controller](controller.md)'s `PerfControllerMonitor` serves on-demand region/agent/plugin resource queries by RPCing `getsysinfo` to `io.cresco.sysinfo`. The plugin is therefore the authoritative source of a host's resource picture, both pushed (via the data-plane sampler) and pulled (via the controller's `getMetrics`/`getSysInfo` path).

## Observability

- **Metrics** — `getmetrics` exposes **30 `MeasurementEngine` gauges** across the `processor`,
  `memory`, `disk`, `network`, `sensors`, `power`, `system`, `gpu`, and `devices` groups (CPU load,
  RAM, disk/NIC I/O, temperature, battery, uptime, and more), aggregated by `getmetricinventory`.
  Every gauge, its OSHI source, and its meaning is catalogued in
  [Metrics & Measurements › sysinfo](../architecture/metrics.md#plugin-sysinfo-host-telemetry-richest-set).
- **Health** — registers the `sysinfo` Felix HealthCheck (tag `local`): OK while host telemetry is
  being served. See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Metrics & Measurements](../architecture/metrics.md) — how sysinfo feeds the measurement subsystem.
- [Data Plane](../architecture/dataplane.md) — the transport for the live-metrics stream.
- [controller](controller.md) — `PerfControllerMonitor`, the consumer of `getsysinfo`.
