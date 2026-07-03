# Glossary

Core Cresco terms, in one place.

| Term | Meaning |
|------|---------|
| **Agent** | A leaf node — a JVM running the Cresco host + plugins, attached as a client to a regional controller's broker. Also, the identity of any node (`region_agent`). |
| **Region / Regional controller** | A controller that runs its own embedded ActiveMQ broker; the agents in its region attach to it. Federates up to a global. |
| **Global / Global controller** | The top of the hierarchy; federates regions (and other globals) together. Serves clients via [wsapi](../plugins/wsapi.md). |
| **Controller mode** | A node's role: `STANDALONE`, `AGENT`, `REGION`, `REGION_GLOBAL`, or `GLOBAL`. |
| **Controller** | The [fabric-brain module](../plugins/controller.md) present on every node (messaging, discovery, security, health, db, data plane). |
| **Plugin** | An OSGi bundle providing functionality and exposing [actions](../api/plugin-actions.md) (e.g. `sysinfo`, `wsapi`, `stunnel`, `repo`). |
| **MsgEvent** | The unit of the [control plane](../architecture/messaging.md) — a typed, addressed, routed message. See [MsgEvent](../api/msgevent.md). |
| **Action** | A callable operation a plugin exposes, declared with `@CrescoAction`; invoked by sending a MsgEvent. |
| **Capability inventory** | The fabric-wide catalog of all actions (`getcapabilityinventory`). |
| **Data plane** | The [pub/sub streaming channel](../architecture/dataplane.md) over ActiveMQ topics (`agent.event` / `region.event` / `global.event`). |
| **Control plane** | The addressed [MsgEvent](../architecture/messaging.md) channel over ActiveMQ queues. |
| **MsgRouter** | The controller component that makes the deterministic per-hop routing decision. |
| **Federation / Network of brokers** | Broker-to-broker ActiveMQ bridges that connect regions to globals (and globals to each other). |
| **Bridge** | A duplex, demand-forwarding ActiveMQ network connector between two brokers. |
| **QoS tier** | The priority class of a message: `LIVENESS` > `CONTROL` > `TELEMETRY` > `BULK`. |
| **Discovery** | Finding a parent controller (UDP broadcast or static TCP), gated by a tier [join secret](../architecture/discovery.md). |
| **Join secret** | The pre-shared `discovery_secret_{agent,region,global}` a node must present to join a tier. |
| **Service key** | `cresco_service_key` — the shared symmetric key authenticating agent traffic; must match fabric-wide. |
| **CrescoIdentity** | A node's identity `tenant/region/agent(/uid)`, bound into its X.509 leaf DN (`CN=agent, OU=region, O=tenant`). |
| **Regional CA** | A region/global controller acting as a certificate authority, signing joiners' leaves — distributed trust (O(regions)). |
| **Tenant** | An isolation domain (`O=` in the cert DN). Multiple tenants can share one fabric; see [Multi-Tenancy](../architecture/tenancy.md). |
| **Tenant namespacing** | Qualifying destinations `T.<tenant>.<raw>` so isolation is enforceable by a local prefix check at every broker. |
| **Role** | An authorization role: `SUPERUSER` (all tenants) or `TENANT` (own tenant only). |
| **Health check** | A Felix `HealthCheck` contributing a pass/fail verdict; run centrally by `CrescoHealthExecutor`. |
| **MeasurementEngine** | The per-bundle Micrometer registry; metrics are aggregated mesh-wide via `getmetricinventory`. |
| **LinkMetrics** | Per-edge network quality (RTT/jitter/throughput/backlog) in the `netlink` metric group. |
| **wsapi** | The [WebSocket API plugin](../plugins/wsapi.md) that bridges external [clients](../clients/overview.md) to the fabric (`wss://host:8282`). |
| **clientlib / pycrescolib** | The external [Java](../clients/java.md) and [Python](../clients/python.md) client SDKs. |
| **CADL / pipeline** | A described application (a graph of plugins/edges) submitted to and orchestrated by the global controller. |
