# Deployment

This page covers running Cresco beyond a single dev node: multi-host meshes, secured/multi-tenant fabrics,
and operational hygiene. For first bring-up see the [Quickstart](../getting-started/quickstart.md).

## Topology building blocks

| You want | Run | Notes |
|----------|-----|-------|
| A self-contained fabric | one **global** | acts as its own region + agent; exposes wsapi `:8282` |
| Scale within a site | globals + **regions** | each region runs its own broker; agents attach locally, region federates up |
| Leaf workloads | **agents** | no broker; attach directly to a region's broker |
| Redundancy / scale-out | multiple **globals** | globals federate to each other (multi-global mesh) |

Each node needs a unique `regionname`/`agentname`. Regions and globals need reachable
[discovery](../architecture/discovery.md) + broker ports.

## Ports

| Port (default) | Service | Who needs it |
|----------------|---------|--------------|
| `32005` | discovery listener | parents (region/global) |
| `32010` | ActiveMQ broker | parents; children connect here |
| `8282` | wsapi (`wss://`) | globals that serve clients |
| `8080` | Felix console / HTTP | optional |

On a **single host**, give each node distinct `broker_port` / `netdiscoveryport` and its own working
directory (the launch scripts do this — `run/nodes/<region>-<agent>/`).

## One node, one directory

Every node keeps on-disk state (Felix cache, Derby DB, ActiveMQ journal) under `cresco-data/` in its
working directory. **Two nodes must never share a directory** or they corrupt each other's state. Run each
from its own directory.

## Securing a fabric

Turn the security stack on in order (details in [Security](../architecture/security.md) and
[Multi-Tenancy](../architecture/tenancy.md)); each step is flag-gated and can be staged region-by-region:

```bash
# 1. shared join secrets + service key (fabric-wide, fixed)
-Ddiscovery_secret_global=... -Ddiscovery_secret_region=... -Ddiscovery_secret_agent=...
-Dcresco_service_key=...
# 2. distributed trust
-Dsecurity_regional_ca=true
# 3. mutual TLS (non-spoofable identity)
-Dbroker_require_client_auth=true
# 4. tenant authorization + isolation
-Dbroker_security_enabled=true
-Dtenant_namespacing=true -Dtenant_id=<tenant>
```

Validate each stage with the [test harnesses](testing.md) before enabling the next.

## Multi-global meshes

Multiple globals can coexist and federate for redundancy and scale. On the same host, distinguish them with
distinct ports (`global_controller_port` / `discovery_port_remote`) and stagger their launches so discovery
settles. This has been exercised at mesh scale (see the [design docs](../reference/design-docs.md)).

## Operational hygiene

- **Reset state between test runs / after rebuilds:** `rm -rf run/cresco-data run/nodes`.
- **Shut down cleanly:** `run/stop.sh` (or `pkill -9 -f agent-1.3-SNAPSHOT.jar`).
- **Watch health:** the [health](../architecture/health.md) subsystem logs a periodic rollup; a stale
  parent link self-heals via re-discovery.
- **Watch metrics:** pull `getmetricinventory` at `region`/`global` scope for a mesh-wide view.
