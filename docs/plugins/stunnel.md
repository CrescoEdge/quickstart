# stunnel — Secure TCP Tunnel

The `stunnel` plugin (bundle `io.cresco.stunnel`) builds bidirectional TCP tunnels across the fabric. It bridges a local TCP listener (SRC) to a remote target (DST) by tunneling raw bytes as JMS messages over the [data plane](../architecture/dataplane.md), using [Netty](https://netty.io/) for socket I/O. This lets a TCP service on one agent's network be reached through Cresco from another agent.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/stunnel` |
| Bundle symbolic name | `io.cresco.stunnel` |
| Java files | 8 |
| Loaded | At runtime by the controller's `StaticPluginLoader`. |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Transport | Raw bytes as JMS `BytesMessage` over `TopicType.GLOBAL`. |
| Socket I/O | Netty event loops. |

## How a tunnel works

A tunnel has a **SRC** side (a local TCP listener accepting connections) and a **DST** side (a Netty client connecting to the real target). Bytes accepted on the SRC socket are wrapped as `BytesMessage`s and published on `TopicType.GLOBAL`; the DST side reassembles them and writes them to the target socket, and vice versa. Tunnel configs are persisted so tunnels auto-reconnect after a restart.

## Key classes

| Class | Role |
|-------|------|
| `PluginExecutor` | Handles the CONFIG/EXEC actions (below). Every handler sets `status`/`status_desc` (10 ok, 9 fail, 99 unknown, 400 missing param, 500 internal). |
| `SocketController` | The core: Netty event loops, tunnel lifecycle, config persistence, health checks, and reconnection. `startSrcTunnel`/`createDstTunnel`/`createDstSession`, `connectWithRetry`, `startHealthCheck`, `checkStartUpConfig` (rescan + reconnect at startup), teardown, and an inner `ReconnectTask`. |
| `SrcChannelInitializer` / `DstChannelInitializer` | Netty pipelines for accepted SRC connections / DST target connections; the handlers forward bytes both ways as `BytesMessage`s and handle `eos`/status `8` (graceful close)/`9` (failure) markers with half-close. |
| `PerformanceMonitor` | Per-direction throughput meter (Micrometer `DistributionSummary`); periodically publishes a throughput record (tagged `cresco_msg_type="stunnel_trace"`) carrying `bits_per_second`, the observed broker **hop path** (`setHops`, from the arriving `cresco_hops` property), and the src/dst region/agent endpoints. |
| `SocketControllerSM` | UMPLE-generated tunnel state machine (only its initial `pluginActive` state is read). |

## Executor actions

CONFIG actions ([`MsgEvent`](../api/msgevent.md) type `CONFIG`):

| Action | Behaviour |
|--------|-----------|
| `configsrctunnel` | Create the SRC-side listener for a tunnel. |
| `configdsttunnel` | Create the DST-side tunnel to the target. |
| `configdstsession` | Establish a DST session for an accepted SRC connection. |
| `removesrctunnel` / `removedsttunnel` | Tear down a tunnel side. |

EXEC actions (type `EXEC`): `tunnelhealthcheck`, `listtunnels`, `gettunnelstatus`, `gettunnelconfig`.

## Health checking

`SocketController.startHealthCheck` runs every ~5s. Recent byte activity is treated as proof-of-life; otherwise it issues an active `tunnelhealthcheck` RPC. Two consecutive failures trigger a tunnel rebuild. On startup, `checkStartUpConfig` rescans persisted configs and reconnects tunnels.

## Configuration read

Per-tunnel config keys:

| Key | Default | Meaning |
|-----|---------|---------|
| `stunnel_id` | — | Tunnel identifier. |
| `src_port` | — | Local SRC listen port. |
| `dst_host` / `dst_port` | — | Remote target host/port. |
| `dst_region` / `dst_agent` / `dst_plugin` | — | DST-side stunnel plugin address. |
| `src_region` / `src_agent` / `src_plugin` | — | SRC-side stunnel plugin address. |
| `buffer_size` | `8192` | I/O buffer size. |
| `watchdog_timeout` | — | Health-check timeout. |
| `performance_report_rate` | `5000` | `PerformanceMonitor` report interval (ms). |
| `debug_performance` | `false` | Verbose throughput logging. |

Plugin-level config keys (read from `getConfig`, apply to the whole stunnel plugin):

| Key | Default | Meaning |
|-----|---------|---------|
| `stunnel_beacon_ms` | `3000` | How often each SRC-configured tunnel pushes an existence/status beacon on the `stunnel_trace` stream. |
| `stunnel_socket_buffer_bytes` | `4194304` | Netty socket send/receive buffer (4 MB). |
| `stunnel_read_chunk_bytes` | `262144` | Max adaptive read chunk (256 KB). |
| `stunnel_write_high_water_bytes` | `2097152` | Write-buffer high-water mark for backpressure (2 MB). |

See [Configuration](../getting-started/configuration.md).

## Path tracing & existence beacons

Because tunnel bytes ride the JMS data plane, transit nodes never see them — only the brokers do. stunnel makes the full path observable and pushes it on a subscribable stream (`cresco_msg_type="stunnel_trace"`):

- **Hop tracing.** Outbound tunnel `BytesMessage`s are marked `cresco_trace="1"` (`SrcChannelInitializer`/`DstChannelInitializer`), so each broker they transit stamps its node identity onto a `cresco_hops` property (see [`CrescoTraceBroker`](../architecture/tunnel-tracing.md)). On arrival, the handler reads `cresco_hops` and feeds it to `PerformanceMonitor.setHops`, so the throughput record carries the real broker path the tunnel crossed.
- **Existence beacons.** `SocketController` publishes one status beacon per SRC-configured tunnel every `stunnel_beacon_ms` (default 3000) on the same `stunnel_trace` stream, so a subscriber knows a tunnel **exists even with zero traffic** (status ACTIVE/INACTIVE, endpoints).

See [Tunnel Path Tracing & Push Observability](../architecture/tunnel-tracing.md) for the end-to-end design (broker hop stamping, the pushed trace stream, per-link bandwidth attribution, and RTT-driven connector auto-scaling).

## Observability

Distinct from the per-tunnel self-healing check above, stunnel is wired into the fabric's central
observability:

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `stunnel.active.tunnels`,
  `stunnel.active.clients`, and `stunnel.active.targets`, aggregated by `getmetricinventory`.
  Per-tunnel throughput is streamed separately as a dataplane `stats` message
  (`bits_per_second`/`bytes_delta`/`total_bytes`). See
  [Metrics & Measurements › stunnel](../architecture/metrics.md#plugin-stunnel).
- **Health** — registers the `stunnel` Felix HealthCheck (tag `local`) reporting the configured-tunnel
  count, discovered by the controller's health executor. See
  [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Tunnel Path Tracing & Push Observability](../architecture/tunnel-tracing.md) — hop tracing, the `stunnel_trace` stream, per-link bandwidth, and connector auto-scaling.
- [Data Plane](../architecture/dataplane.md) — the JMS transport tunneled bytes travel over.
- [Metrics & Measurements](../architecture/metrics.md) — `PerformanceMonitor` throughput reporting.
- [Overview](overview.md) — the functional-plugin model.
