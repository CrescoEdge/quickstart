# Tunnel Path Tracing & Push Observability

Cresco can trace the **full broker path of every stunnel tunnel** end-to-end, stream it live, and show
per-link bandwidth and automatic connector scaling — all **pushed**, never polled. This page documents how,
and the architectural principle it proves: *push survives load that pull cannot*.

## The problem: dataplane bytes are invisible to Cresco at transit

A [stunnel](../plugins/stunnel.md) tunnel carries its bytes over the **data plane** — JMS `BytesMessage`s on
the `global.event` topic, demand-forwarded by ActiveMQ's network-of-brokers. Only *tunnel setup* uses
addressed (cost-routed) RPCs; the **bytes themselves ride the broker bridges**, and the Cresco controllers at
*transit* nodes never see them — only the brokers do. So a tunnel's path cannot be traced from Cresco message
routing, and its per-link bandwidth does not show up in the controller's link metrics.

## Hop tracing at the broker (the only place that sees transit)

`CrescoTraceBroker` is an ActiveMQ `BrokerFilter` installed in every broker. On each `send`, if a
`global.event` message is marked `cresco_trace=1`, it appends **this broker's node identity** to a
`cresco_hops` message property. As the message is demand-forwarded broker→broker, each hop appends itself, so
it arrives at the consumer carrying the **full ordered broker path it traversed**.

- The stunnel plugin marks its data messages `cresco_trace=1` and reads `cresco_hops` off arriving messages.
- Gated by `net_trace_hops` (default on); only marked messages pay the cost.

> Proven: a tunnel between agents in region 3 and region 13 traced `R3 → G0 → G1 → R13` — across two regions
> and two globals — end to end.

## Pushed, subscribable trace stream

The stunnel `PerformanceMonitor` publishes each tunnel's live **throughput + observed hop path** on a
subscribable stream (`cresco_msg_type='stunnel_trace'`), and `SocketController` publishes a periodic
**existence beacon** (status ACTIVE/INACTIVE, endpoints) on the same stream — so a subscriber knows a tunnel
**exists even with zero traffic**. A dashboard *subscribes*; it does not poll per-tunnel.

## Per-link bandwidth &amp; automatic connector scaling

- **Bandwidth.** Because stunnel bytes are invisible to the controller's link metric, the dashboard
  **attributes each tunnel's known throughput to every link on its traced path** — the real per-link bandwidth
  of the tunnel across the mesh, coloured and sized on the graph.
- **Auto-scaling.** The [AutoTuner](../design/link-metrics-design.md) grows broker-bridge connectors under
  load. Dataplane byte floods don't register as Cresco send-latency/backlog, but they **inflate a bridge's
  measured RTT**, so an RTT-congestion trigger (`net_autotune_rtt_high_ms`, default 40) scales connectors on
  high RTT too.

> Proven: under a tunnel flood on a throttled backbone, the link RTT spiked and the broker connectors grew
> **c1 → c16 (the max)** automatically.

## Push beats pull under load (proven)

Under heavy tunnel load, the **`getnetworkstate` poll starved** — a global busy forwarding Gb/s could not
service the RPC, and the topology graph went dark. The **pushed** telemetry did not: the `stunnel_trace`
stream and the topology graph (rebuilt from the pushed `route_lsa` link-state stream) both **stayed live while
the connectors grew**. This is the operational proof of Cresco's rule: **metrics and state are pushed as they
change (scales); pulling per-request does not.**

## Configuration

| Parameter | Default | Effect |
|---|---|---|
| `net_trace_hops` | `true` | Broker stamps `cresco_hops` on `cresco_trace` data-plane messages (tunnel path tracing). |
| `stunnel_beacon_ms` | `3000` | How often each tunnel pushes its existence/status beacon on the trace stream. |
| `net_autotune_rtt_high_ms` | `40.0` | Smoothed-RTT threshold above which the AutoTuner scales connectors (catches dataplane floods). |

## See also

- [stunnel plugin](../plugins/stunnel.md) — the tunnel plane itself
- [Dynamic Cost-Aware Routing](dynamic-routing.md) · [Data Plane](dataplane.md) · [Network Link Metrics](../design/link-metrics-design.md)
