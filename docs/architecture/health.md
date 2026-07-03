# Health & State

Cresco separates **health** (pass/fail verdicts about whether things are working) from
[metrics](metrics.md) (continuous measurements). Health is built on **Felix Health Checks** (OSGi
`org.apache.felix.hc.api.HealthCheck`), so any bundle can contribute a check that is discovered and run
centrally.

## The health executor

`CrescoHealthExecutor` discovers all registered health-check services and runs them on a schedule. Its
verdict logic is deliberately anti-flap:

- **Grace window** — a check that goes `TEMPORARILY_UNAVAILABLE` only escalates to `CRITICAL` if it stays
  down past the grace window (avoids single-blip failovers).
- **Sticky window** — a recently-failed check stays `WARN` for a while after recovery (avoids rapid
  oscillation).
- Results are cached and rolled up; `aggregate(tags)` returns the worst status among matching checks.

## The checks

Checks are grouped by **tag**:

| Tag | Check | Verifies |
|-----|-------|----------|
| `local` | `broker` | embedded ActiveMQ broker up (region/global) |
| `local` | `dataplane` | JMS fault URI connection active |
| `local` | `db` | Derby database reachable |
| `local` | `disk` | free disk above a floor |
| `local` | `memory` | JVM heap below a threshold |
| `local` | `plugins` | worst plugin status |
| `local` | `link:quality` | parent link RTT/jitter within bounds (reads [metrics](metrics.md)) |
| `link` | `link:parent` | parent controller reachable (liveness) |
| `mesh` | `mesh:subtree` | rolled-up child health |

## Liveness and the mesh rollup

Each node actively pings its parent (agent → region, region → global) every few seconds and stamps the
last successful pong. `ParentLinkHealthCheck` reads that timestamp: a stale pong → `TEMPORARILY_UNAVAILABLE`
→ (after the grace window) `CRITICAL`. Health also **rides the existing ping/pong** — `MeshHealthPing`
stamps a child's rolled-up health on its ping and the parent's health on the pong, so a parent learns its
subtree's health and a child learns its parent's, with no extra messages. `MeshHealth`/`SubtreeHealthCheck`
surface child degradation at the parent.

## Health drives state

Only one thing crosses from health into the state machine: when `link:parent` goes `CRITICAL`,
`HealthMinaBridge` fires the corresponding loss event (`regionalControllerLost` for an agent,
`globalControllerLost` for a region) — which triggers recovery/re-discovery. All other checks are pure
observability. `StatusAdapter` maps legacy node/plugin status codes into Felix HC statuses.

## The state machine

The controller runs a state machine over the [controller modes](overview.md#nodes-and-the-hierarchy)
(`STANDALONE` / `AGENT` / `REGION` / `REGION_GLOBAL` / `GLOBAL`). Node, plugin, and edge status
(`ACTIVE`, `STALE`, `LOST`, …) is persisted per-controller in **Derby**; regional and global watchdogs age
out stale nodes. The health subsystem and the state machine together give the fabric its self-healing
behavior. Design detail: the health-check design doc via [Design Docs](../reference/design-docs.md).
