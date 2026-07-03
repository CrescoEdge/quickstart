# Cresco Documentation

![Cresco](images/cresco_logo.png){ width="220" }

**Cresco is a hierarchical, secure, multi-tenant distributed agent mesh.** It lets you deploy, connect,
and orchestrate software across many machines — from a single laptop to a global fabric of regions — with
a uniform messaging model, a streaming data plane, cryptographic identity, and per-tenant isolation.

This site is the **canonical documentation** for Cresco. All information about the project — architecture,
every module and plugin, every callable action, configuration, and both client libraries — lives here.

!!! info "New here? Start with the [Architecture Overview](architecture/overview.md), then the [Quickstart](getting-started/quickstart.md)."

## What Cresco is

A Cresco deployment is a **mesh of agents**. Each agent is a JVM process (JDK 21) running an OSGi
(Apache Felix) host that loads the Cresco **controller** and a set of **plugins**. Agents self-organize
into a three-tier hierarchy:

| Tier | Role |
|------|------|
| **Agent** | A leaf node. Runs plugins, connects to a regional controller. |
| **Region** | A regional controller. Runs an embedded ActiveMQ broker; the agents in its region connect to it. |
| **Global** | A global controller. Federates regions together; the top of the hierarchy. A node can be `REGION_GLOBAL` (both). |

Nodes communicate two ways:

- **[MsgEvent](api/msgevent.md) control plane** — addressed, routed messages (commands, config, RPC,
  telemetry) between agents, regions, and globals over ActiveMQ queues.
- **[Data plane](architecture/dataplane.md)** — high-throughput pub/sub topics for streaming application
  data.

## Key capabilities

- **Uniform messaging & routing** — every node addresses every other by `region_agent[_plugin]`; the
  [controller](plugins/controller.md) routes deterministically across a federated network of brokers. See
  [Messaging & Routing](architecture/messaging.md).
- **Plugins with self-describing actions** — functionality is packaged as OSGi bundles that expose
  `@CrescoAction` operations, aggregated fabric-wide into a [capability inventory](api/plugin-actions.md)
  (89 actions today).
- **Cryptographic identity & trust** — every node carries an identity-bearing X.509 leaf
  (`CN=agent, OU=region, O=tenant`), mutual TLS, and distributed **regional-CA** trust. See
  [Security & Identity](architecture/security.md).
- **Multi-tenancy** — end-to-end [tenant isolation](architecture/tenancy.md) with destination namespacing
  and role-based authorization (`SUPERUSER` / `TENANT`), enforced at every broker.
- **Health & metrics** — Felix [health checks](architecture/health.md) and a unified Micrometer
  [metrics](architecture/metrics.md) model with mesh-wide aggregation.
- **Client SDKs** — drive the fabric from [Java](clients/java.md) or [Python](clients/python.md) over a
  secure WebSocket.

## How this site is organized

- **[Getting Started](getting-started/installation.md)** — build from source, run a mesh, configure it.
- **[Architecture](architecture/overview.md)** — how the mesh, messaging, data plane, discovery, security,
  tenancy, health, and metrics work.
- **[Modules & Plugins](plugins/overview.md)** — a page for every module (`agent`, `library`, `controller`,
  `core`, `logger`, `repo`, `sysinfo`, `wsapi`, `stunnel`).
- **[API Reference](api/plugin-actions.md)** — all plugin actions, the Library API, the MsgEvent structure,
  and the full [configuration parameter](reference/configuration.md) reference.
- **[Client Libraries](clients/overview.md)** — Java and Python SDK references.
- **[Operations](operations/deployment.md)** — deployment and testing.
- **[Reference](reference/glossary.md)** — glossary and the underlying design documents.

## Project layout

```
cresco/
├── code/            # source (one OSGi module / git repo per directory)
│   ├── agent/       # OSGi host bootstrap → the runnable agent uber-jar
│   ├── library/     # the plugin & controller API (io.cresco.library)
│   ├── controller/  # the fabric brain (messaging, discovery, security, health, db)
│   ├── core/        # controller/JVM lifecycle service
│   ├── logger/      # logging bootstrap
│   ├── repo/        # plugin/file repository
│   ├── sysinfo/     # system info + benchmark plugin
│   ├── wsapi/       # WebSocket API (external client bridge, :8282)
│   ├── stunnel/     # secure TCP tunnel plugin
│   ├── clientlib/   # external Java SDK
│   └── pycrescolib/ # external Python SDK
└── run/             # staged agent jar + launch scripts + tests
```

Source repositories live under [github.com/CrescoEdge](https://github.com/CrescoEdge).
