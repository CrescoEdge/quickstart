# Cresco Network Link Metrics + Automated Tuning — Design

Status: **implemented + PROVEN (measurement + health + control loop + cost-aware routing).** Min-cost
selection is now live and proven on a redundant multi-region containerlab mesh (`RouteComputer` Dijkstra
over the pushed `RouteView`; `MsgRouter` source-route injection) — see §5/§7#5 and
[Dynamic Cost-Aware Routing](../architecture/dynamic-routing.md). Sibling to `health-check-design.md`
(which explicitly defers "the measurement/metrics unification" — this doc is that effort's link slice)
and `region-federation-design.md`.

## 1. The OSGi split (why this is measurements, not health)

The codebase already draws three lines; this design stays inside them:

| Concern | Mechanism | Home |
|---|---|---|
| **Health** | Felix Health Checks (`org.apache.felix.hc.api.HealthCheck`), OSGi services, pass/fail verdicts | `controller/health/` |
| **Measurements** | Micrometer via `MeasurementEngine.getCrescoMeterRegistry()` (push gauges/timers) | `library/metrics/` + `controller/measurement/` |
| **Benchmarks** | perf harnesses | `run/tests/` |

Link metrics are **measurements**. RTT/jitter/throughput/send-latency/backlog are continuous values,
registered as Micrometer gauges and surfaced through the normal metrics path — **not** health. Health
*consumes* them: a `link:quality` HealthCheck reads the metrics and emits `WARN` when degraded, without
re-measuring. The control loop (AutoTuner) also consumes them, and **lives in the controller**.

## 2. Components (all in `controller/netmetrics/`, controller-resident)

- **`LinkMetrics`** — one per neighbor edge (`region_agent`). TCP-style smoothed RTT + jitter
  (Jacobson/Karels), windowed tx/rx byte rate, producer send-latency EWMA, broker backlog, NIC ceiling.
  Lock-free; updated from hot paths, read by the tuner. Exposes a composite `cost()` for a future router.
- **`LinkMetricsRegistry`** — map keyed by edge path; registers each edge's values as Micrometer gauges
  in the shared `MeasurementEngine` and `publishAll()`s current values on the tuner interval. Owns the
  one canonical `parentLinkKey(ce)` so every producer/consumer of a link's metrics agrees on the key.
- **`NetTuningProfile`** — the single mutable source of truth for every I/O tunable: socket buffer,
  read/block size, write high-water, connections-per-link. Config-seeded (defaults == today), bounded,
  versioned. In-controller I/O reads it live; a snapshot is broadcast to out-of-controller plugins.
- **`AutoTuner`** — the controller control loop. **Always** samples + publishes metrics (measurement);
  **actuation is gated by `net_autotune`**. Adapts buffers to the bandwidth-delay product
  (throughput × RTT), block size to throughput, and connections to saturation (send-latency + backlog),
  all bounded + cooldown'd. Drives the broker's dynamic `addBridgeConnections`/`removeBridgeConnections`
  and broadcasts the profile as a `nettuning` CONFIG MsgEvent.

## 3. Signal sources (measure what you already pay for)

- **RTT — active probe, free.** `AgentHealthWatcher` / `RegionHealthWatcher` already run a synchronous
  ping RPC to the parent every 5s and discard the timing. We wrap it in `nanoTime()` → per-parent-link
  RTT + jitter. End-to-end application-path latency (incl. broker enqueue/dequeue) — better than ICMP
  for cost. (Health reads pong-liveness; measurements read the timing — one probe, two consumers.)
- **Send-latency + throughput — passive.** `DataPlaneServiceImpl` times `producer.send()` (dwell rises
  under broker flow-control pressure = downstream congestion) and counts bytes via a readable `dp_bytes`
  JMS property the wsapi/stunnel producers stamp (`getBodyLength()` fails on a just-sent write-mode msg).
- **Backlog — JMX (DONE).** `DestinationViewMBean.QueueSize` is the best native congestion signal;
  polled into `pendingBacklog` by `ActiveBroker.getBrokerPendingBacklog()` in-process each tuner cycle
  (see §5/§7#1).

## 4. Validated (2-node fabric, live)

- RTT harvest: `Link[global-region_parent] srtt=107.55ms jitter=21.19ms rttHi=192.29ms` (EWMA converging).
- Throughput: `Link[global-region_global-controller] tx=42.0MB/s` under load.
- `link:quality` HealthCheck: `OK -> WARN (parent link degraded: rttHi=330.5ms jitter=55.1ms)` — the
  health-consumes-measurements loop, live. (Same-host pings are genuinely jittery; the WARN is correct.)
- Metrics published to Micrometer under `netlink` group as `link.<path>.{rtt_ms,jitter_ms,tx_mbps,rx_mbps,sendlat_ms,backlog}`.

## 5. Actuation reach (honest scoping)

- **Buffer/block sizes** — `NetTuningProfile` is read live by in-controller I/O; the AutoTuner
  broadcasts it as a `nettuning` CONFIG to the local wsapi + stunnel plugins, which **apply it live**
  (`NettyWsServer.applyNetTuning` per-connection in `initChannel`; `SocketController.applyNetTuning`
  read at each tunnel bootstrap). **Validated end-to-end:** AutoTuner adapted block size 256KB→64KB
  from a measured 42 MB/s and both plugins logged `applyNetTuning readChunk=65536`.
- **Connections** — `addBridgeConnections`/`removeBridgeConnections` scale the **region↔region
  federation** bridge (validated for correctness; now exercised on a redundant multi-region mesh, where the
  live connector count is advertised in link-state and feeds [cost-aware routing](../architecture/dynamic-routing.md)).
  The **agent→region** funnel scales via per-shard dedicated connections (validated, +40%).
- **Backlog** — `ActiveBroker.getBrokerPendingBacklog()` reads DestinationViewMBean QueueSize in-process
  (JMX); the AutoTuner polls it onto the uplink each cycle.

## 6. Config surface (all default-off / default-today)

`net_autotune` (actuation on/off) · `net_autotune_interval_sec` · `net_autotune_sendlat_high_ms` /
`_low_ms` · `net_autotune_backlog_high` · `net_autotune_cooldown_ms` · `net_autotune_bdp_safety` ·
`net_socket_buffer_bytes`/`_min`/`_max` · `net_read_chunk_bytes`/`_min`/`_max` ·
`net_connections_per_link`/`_min`/`_max` · `net_metrics_log` (demo logging) ·
`link_quality_{rtt,jitter,sendlat}_warn_ms` · `link_quality_backlog_warn`.

## 7. Increment status

1. **DONE** — JMX `QueueSize` backlog poller → `pendingBacklog` (`ActiveBroker.getBrokerPendingBacklog`).
2. **DONE** — `nettuning` CONFIG handlers in wsapi + stunnel; validated live end-to-end.
3. **DONE (config)** — capacity ceiling via `net_link_speed_bps`; OSHI/sysinfo can supply the real NIC
   link speed later (not on the controller classpath today).
4. **DONE + PROVEN** — region↔global federation-edge RTT is harvested from RegionHealthWatcher's
   existing region→global ping (the broker-bridge edge), same pattern as the agent harvest. The
   earlier `BrokerMonitor.probeFederationRtt` was removed: it used an agent-addressed message the
   peer's ping responder never answers. Validated on a live global+region mesh (bridge forms;
   `Link[global-region_global-controller]` srtt≈109–121 ms, samples>150) and asserted by the suite's
   phase-2 (`federation:broker-bridge`, `federation:edge-rtt`).
5. **DONE + PROVEN** — cost model (`LinkMetrics.cost()`, `LinkMetricsRegistry.costOf/lowestCostEdge`) +
   `MsgRouter` per-hop cost annotation (`net_cost_routing`). Min-cost **selection is now live** on a
   redundant multi-region mesh: `RouteComputer` runs Dijkstra over the pushed-link-state `RouteView` and
   `MsgRouter` injects the chosen source route. Proven in the containerlab mesh with independent
   receiver-side hop stamps. See [Dynamic Cost-Aware Routing](../architecture/dynamic-routing.md).
