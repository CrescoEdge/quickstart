# Messaging & Routing

The control plane moves [MsgEvent](../api/msgevent.md) objects between agents, regions, and globals over a
federated network of ActiveMQ brokers. This page covers addressing, routing, the broker fabric,
federation, and quality-of-service.

## Addressing

Every node has an identity `region_agent`, and every plugin `region_agent_plugin`. A MsgEvent carries a
source and destination `(region, agent, plugin)` plus two scope flags, `isRegional` and `isGlobal`. The
per-node inbox is a JMS **queue** named `<region>_<agent>` (or `T.<tenant>.<region>_<agent>` under
[tenant namespacing](tenancy.md)).

## Routing

`MsgRouter.route()` makes a **deterministic, local** forwarding decision. It computes a 16-bit route path
from the message's scope flags and destination against the local controller's state (am I the region? the
global? is the destination local?), then dispatches to one of a fixed set of forwarders:

| Forwarder | Meaning |
|-----------|---------|
| `forwardToLocalAgent` / `forwardToLocalPlugin` | deliver in-process |
| `forwardToLocalRegionalController` | up to this region's controller |
| `forwardToRemoteRegionalController` / `forwardToRemoteRegion` | across a broker to another region |
| `forwardToLocalGlobal` / `forwardToRemoteGlobal` | up to / across to a global |

Routing is **single-path and metric-blind** today: the message follows the tree (agent → region → global →
region → agent). Loop safety is provided by a **BrokerPath** trail on each message plus a TTL cap. A
performance-aware, multi-path evolution is designed in the
[optimal global routing plan](../reference/design-docs.md).

!!! note "Relays preserve, they do not re-address"
    A message keeps its `(dst_region, dst_agent, dst_plugin)` end-to-end; each relay recomputes only the
    *next hop* from `getForwardDst()`. Under [tenant namespacing](tenancy.md) the origin tenant is stamped
    once and preserved, so every relay qualifies the destination queue identically.

## The broker fabric

Region and global controllers run an embedded ActiveMQ (`SslBrokerService`) via `ActiveBroker`. Agents
connect as clients via `ActiveClient` using a `failover:(nio+ssl://host:port)` URL; the in-JVM controller
connects over `vm://localhost`. Producers and consumers are managed by `AgentProducer` / `AgentConsumer`.

Notable broker configuration (all tunable — see the
[Configuration reference](../reference/configuration.md)):

- **KahaDB** persistence with journal disk-syncs off by default (persistence is used as a flow-control
  mechanism, not for crash durability).
- Producer flow control off, prioritized messages on, per-destination memory limits, large socket buffers.
- **Network bridges** (`AddNetworkConnector` / `buildConnector`) are duplex, demand-forwarding, with a
  configurable `networkTTL`; parallel connectors per peer can be added/removed at runtime
  (`addBridgeConnections`) for throughput.

## Federation (network of brokers)

Regions federate to globals (and globals to each other) via ActiveMQ **network bridges**. A bridge only
forwards a message when the far side has advertised **demand** (a consumer subscription); loop prevention
uses the message's BrokerPath. ActiveMQ resolves *reachability* across the mesh but does **not** select
paths by latency/throughput — Cresco owns any smarter routing at the `MsgRouter` layer.

## Quality of service

To guarantee that a flood of bulk data can never starve agent-liveness traffic, outbound MsgEvents are
classified by `MsgQoS` into four tiers:

| Tier | JMS priority | Delivery | Traffic |
|------|-------------|----------|---------|
| **LIVENESS** | 9 | persistent | ping/pong, watchdog |
| **CONTROL** | 7 | persistent | config, commands |
| **TELEMETRY** | 4 | non-persistent | info/log/KPI (evictable under pressure) |
| **BULK** | 1 | persistent | file/binary transfers |

Liveness + control ride a **dedicated, isolated producer/session** (`ControlPlaneSender`), separate from
the per-destination telemetry workers and the bulk executor, so control traffic is never serialized behind
or evicted by a flood. See the broker-performance design doc via [Design Docs](../reference/design-docs.md).

## RPC

Request/response is built on MsgEvent: a sender sets a reply destination and blocks for the correlated
reply (`PluginBuilder.sendRPC(msg, timeoutMs)`). The [client libraries](../clients/overview.md) expose this
as ordinary method calls.
