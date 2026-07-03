# Library API

The `library` module (`io.cresco.library`) is the API surface that **plugins and the controller** build
on. If you are writing a Cresco plugin, this is your SDK; if you are integrating, it's the shared vocabulary
the whole fabric speaks. For the module overview see [library](../plugins/library.md).

## PluginBuilder — the plugin's handle to the fabric

`PluginBuilder` is injected into a plugin and is its gateway to everything: identity, config, logging,
messaging, RPC, and services.

| Area | Key methods |
|------|-------------|
| Identity | `getRegion()`, `getAgent()`, `getPluginID()` |
| Config | `getConfig()` → [`Config`](#config) |
| Logging | `getLogger(name, level)` → `CLogger` |
| Messaging | `msgIn(MsgEvent)`, `msgOut(MsgEvent)`, `sendRPC(MsgEvent, timeoutMs)` |
| Message factories | `getGlobalControllerMsgEvent(type)`, `getRegionalControllerMsgEvent(type)`, `getAgentMsgEvent(type)`, `getKPIMsgEvent()` |
| Services | access to `AgentService` / `DataPlaneService` |

## Config

`Config` provides typed access to a node's [configuration](../getting-started/configuration.md):

```java
Config c = plugin.getConfig();
int    p = c.getIntegerParam("broker_port", 32010);
long   t = c.getLongParam("controlplane_ttl", 300000L);
boolean s = c.getBooleanParam("broker_security_enabled", false);
String  n = c.getStringParam("tenant_id", "default");
double  d = c.getDoubleParam("prefetch_rate_multiplier", 2.5);
```

Resolution is **system property → config map → default**; the default preserves shipped behavior. See the
full [Configuration Parameters reference](../reference/configuration.md).

## MsgEvent — the message

The unit of the control plane. Full structure and method reference on its own page: **[MsgEvent](msgevent.md)**.
In brief: a typed message (`EXEC`, `CONFIG`, `INFO`, `KPI`, `WATCHDOG`, …) with source/destination
`(region, agent, plugin)`, scope flags, string parameter maps, optional compressed params, and a file
list. Build one from a `PluginBuilder` factory, set params (including `action` for a plugin
[action](plugin-actions.md)), and `msgOut()` or `sendRPC()` it.

## DataPlaneService — streaming

The pub/sub streaming interface; see [Data Plane](../architecture/dataplane.md) for concepts.

| Method | Purpose |
|--------|---------|
| `addMessageListener(TopicType, MessageListener, selector)` | subscribe to a scope with an optional JMS selector |
| `sendMessage(TopicType, Message)` | publish to a scope |
| `createTextMessage(...)` / `createBytesMessage(...)` | build messages |
| `removeMessageListener(id)` | unsubscribe |

`TopicType` is `AGENT`, `REGION`, or `GLOBAL`.

## MeasurementEngine — metrics

Register and update metrics; see [Metrics & Measurements](../architecture/metrics.md).

| Method | Purpose |
|--------|---------|
| `setTimer` / `updateTimer` | latency timers (nanoTime deltas) |
| `setGauge` / `updateIntGauge` / `updateLongGauge` / `updateDoubleGauge` | point-in-time values |
| `setDistributionSummary` / `updateDistributionSummary` | value distributions |
| `getAllMetrics()` | `Map<group, List<Map<String,String>>>` — the wire schema behind `getmetrics` |

## Security primitives

Under `io.cresco.library.security` (see [Security & Identity](../architecture/security.md) and
[Multi-Tenancy](../architecture/tenancy.md)):

| Type | Purpose |
|------|---------|
| `CrescoIdentity` | identity ⇄ cert DN (`tenant`/`region`/`agent`/`uid`) |
| `TenantNamespace` | qualify/parse tenant-namespaced destinations (`T.<tenant>.<raw>`) |
| `MessageSigner` | sign/verify a message payload with the leaf key |
| `CertTrust` | PKIX chain-verify a leaf to a trusted region CA |

## Capability annotations

Plugin actions are declared with `@CrescoAction` (and `@CrescoParam` / `@CrescoReturn`), which makes them
self-describing and aggregatable into the fabric-wide [capability inventory](plugin-actions.md).
