# Health & State

Cresco separates **health** (pass/fail verdicts about whether things are working) from
[metrics](metrics.md) (continuous measurements). Health is built on **Felix Health Checks** (OSGi
`org.apache.felix.hc.api.HealthCheck`): any bundle — the controller or any plugin — can contribute a
check that is discovered and run centrally, with no wiring beyond registering an OSGi service.

This page is exhaustive: it documents the health machinery (discovery, the grace/sticky state
machine, aggregation, propagation), then **every health check the fabric runs** — what it inspects,
every status it can return and the exact condition for each, and its message format.

## The status vocabulary

Every check returns one `Result.Status`. They are strictly ordered worst-wins (aggregation compares
ordinals):

```
OK  <  WARN  <  TEMPORARILY_UNAVAILABLE  <  CRITICAL  <  HEALTH_CHECK_ERROR
```

| Status | Meaning |
|--------|---------|
| `OK` | working normally |
| `WARN` | degraded but functional / recently recovered (see sticky) |
| `TEMPORARILY_UNAVAILABLE` | not ready or a transient outage; **not yet** fatal (see grace) |
| `CRITICAL` | a real, sustained failure |
| `HEALTH_CHECK_ERROR` | the check itself threw or returned null — the most severe, because the signal is untrustworthy |

## The health executor

`CrescoHealthExecutor` is the scheduler and aggregator. It is started during controller start (inside
a try/catch, so a missing Health-Check API bundle can never block startup) and opens a
`ServiceTracker` over `HealthCheck` services: each service is read **once at registration**, wrapped
in a `CheckState`, and scheduled on a 2-thread daemon pool named `CrescoHealth`.

Per-check properties are read from the OSGi service registration: `HealthCheck.NAME` (else the class
simple name), `HealthCheck.TAGS`, an optional per-check interval (`ASYNC_INTERVAL_IN_SEC`), grace
(`hc.cresco.graceInSec`), and sticky (`KEEP_NON_OK_RESULTS_STICKY_FOR_SEC`) — anything unset inherits
the executor defaults.

### Scheduling & configuration

| Config key | Default | Meaning |
|---|---|---|
| `health_check_interval_ms` | **5000 ms** | default run interval per check |
| `health_grace_ms` | **20000 ms** | default grace window |
| `health_sticky_ms` | **60000 ms** | default sticky window |
| `health_summary_interval_ms` | **30000 ms** | interval of the one-line rollup log (`0` disables) |

### The grace / sticky state machine (raw vs effective)

Each run computes both a **raw** status (what `execute()` literally returned) and an **effective**
status (what grace + sticky produce, and what aggregation uses). Before the first run a check is
`TEMPORARILY_UNAVAILABLE` ("not yet run").

1. **Execute + capture raw.** A `null` return → `HEALTH_CHECK_ERROR "null result"`; a thrown
   exception → `HEALTH_CHECK_ERROR "check threw: …"`. Otherwise raw = the returned status.
2. **Grace — sustained `TEMPORARILY_UNAVAILABLE` → `CRITICAL`.** The first time a check reports
   `TEMPORARILY_UNAVAILABLE`, a timer starts. Only once it has stayed unavailable for longer than the
   grace window is it promoted to `CRITICAL`. This absorbs a transient link blip or GC pause so it
   never triggers a spurious failover; a freshly-unavailable check stays `TEMPORARILY_UNAVAILABLE`
   until the window elapses. Any other status resets the timer.
3. **Sticky — recovered-but-recently-bad held at `WARN`.** When a check that was non-OK returns to
   `OK`, its **effective** status is held at `WARN` for the remainder of the sticky window. A flap
   that self-heals is therefore not silently masked — you see `WARN` for ~60 s after recovery, then
   `OK`. (This is why a freshly-registered, healthy check reads `WARN` briefly before settling.)
4. **Transition + listeners.** If the effective status changed, the executor logs
   `health '<name>' <old> -> <new> (<message>)` and notifies every `HealthListener`.

### Aggregation, snapshots & the summary log

- `aggregate(tags…)` returns the **worst effective** status among matching checks (OK if none match).
- Tag matching: an empty query matches all; a `-tag` entry is an explicit exclusion; positive tags
  are OR'd.
- `execute(tags…)` / `all()` return cached `HealthResult` snapshots — a cheap atomic read, **not** a
  re-run. (This is why an on-demand read can lag a just-changed state by up to one check interval;
  metric gauges are live, health snapshots are periodic.)
- Every `health_summary_interval_ms` the executor logs one line:
  `health summary [<worst>]: broker=OK db=OK … cep=OK executor=OK …`.

A `HealthResult` is immutable: `name`, `tags`, `status` (effective), `rawStatus`, `message`,
`lastRunTs`.

## Querying health — `gethealthinventory`

`gethealthinventory` is the queryable surface for health — the parallel of
[`getmetricinventory`](metrics.md#mesh-wide-aggregation-getmetricinventory). It returns this node's
full health snapshot as JSON:

```json
{
  "node": "<region>_<agent>",
  "aggregate": "<worst-effective-status>",
  "checks": [
    { "name": "cep", "status": "OK", "rawStatus": "OK",
      "message": "cep OK: 0 active queries", "tags": ["local"], "lastRunTs": 1720000000000 }
  ]
}
```

It is exposed on all three controller tiers (`AgentExecutor`, `GlobalExecutor`, and
`RegionalExecutor`, which delegates to the global handler). If the executor hasn't started, it
returns `aggregate: "UNKNOWN"` with an empty `checks` list.

## Liveness and the mesh rollup

Each node actively pings its parent (agent → region, region → global) every few seconds and stamps
the last successful pong; `link:parent` reads that timestamp. Health also **rides the existing
ping/pong** with no extra messages — `MeshHealthPing` stamps a child's rolled-up health on its ping
and the parent's health on the pong, so a parent learns its subtree's health and a child learns its
parent's. A node advertises `aggregate("local","mesh")` — deliberately **excluding** `link`, so a
node's own parent-link liveness is never conflated with its health when propagating upward.

## Health drives state

Exactly one thing crosses from health into the state machine: when **`link:parent`** reaches
`CRITICAL`, `HealthMinaBridge` fires the corresponding loss event (`regionalControllerLost` for an
agent, `globalControllerLost` for a region), which triggers recovery/re-discovery. Every other check
is pure observability — `plugins`, `subtree`, and `link:quality` are truthful-but-non-failover by
design. MINA remains the state authority; the bridge only feeds it a clean, grace-protected loss
signal.

---

# The full health-check catalog

Format per check: **name** (tags) — what it inspects — the statuses it can return and the exact
condition for each.

## Controller — `local` checks

Registered by `LocalHealthChecks` with `tags=[local]`; they inherit the executor defaults
(interval 5 s / grace 20 s / sticky 60 s).

### `broker` (tags: `local`)
The embedded ActiveMQ broker (regional and global controllers run one; a plain agent does not).

- **OK** — no local broker on this role (`"n/a (no local broker on this role)"`), or broker healthy
  and its manager active (`"broker healthy"`).
- **TEMPORARILY_UNAVAILABLE** — broker not yet started (`"broker not yet started"`), or broker
  manager not active (`"broker manager not active"`).
- **CRITICAL** — broker present but `!isHealthy()` (`"broker not started/healthy"`).
- **HEALTH_CHECK_ERROR** — the check threw.

### `dataplane` (tags: `local`)
This node's own JMS messaging-plane connection (the "fault" URI to its broker).

- **OK** — fault URI active (`"messaging plane active"`).
- **TEMPORARILY_UNAVAILABLE** — active client not ready, or fault URI not active. (Grace promotes a
  sustained outage to CRITICAL.)
- **HEALTH_CHECK_ERROR** — the check threw.

### `db` (tags: `local`)
The controller state store (Derby), probed with a benign read of this node's own agent record.

- **OK** — DB handle present pre-registration, or `getANode(agent)` returns without throwing
  (`"db reachable"`) — any result proves the DB is up.
- **TEMPORARILY_UNAVAILABLE** — DB handle not ready.
- **CRITICAL** — the read threw (Derby unreachable) (`"db query failed: …"`).

### `disk` (tags: `local`)
Free usable space under the Cresco data dir. Floor = `health_disk_floor_mb` (default **100 MB**).

- **CRITICAL** — free < floor (`"low disk: <free>MB free < floor <floor>MB (<dir>)"`).
- **WARN** — free < floor × 2 (`"disk approaching floor: …"`).
- **OK** — otherwise (`"<free>MB free (<dir>)"`).
- **HEALTH_CHECK_ERROR** — the check threw.

### `memory` (tags: `local`)
JVM heap pressure = `(total − free) / max × 100`. Thresholds `health_mem_warn_pct` (default **85**),
`health_mem_crit_pct` (default **95**). Message base: `"heap <pct>% used (<used>MB/<max>MB)"`.

- **CRITICAL** — pct ≥ crit.
- **WARN** — pct ≥ warn.
- **OK** — otherwise.
- **HEALTH_CHECK_ERROR** — the check threw.

### `plugins` (tags: `local`)
Aggregates every hosted plugin: each plugin's `status_code` is mapped to a status and the worst
wins. A sick plugin is a signal only; it does not by itself drive a state transition.

- **TEMPORARILY_UNAVAILABLE** — plugin admin not ready.
- **OK** — no plugins loaded, or all plugins OK.
- **worst-of-mapped** — `"<n> plugin(s): name[id]=code(STATUS), …"`. Code mapping: `10`/`8` → OK;
  `3`/`40` (init / WATCHDOG stale) → TEMPORARILY_UNAVAILABLE; `90`/`91`/`92` (shutdown/verify/disable
  timeouts) → WARN; `7`/`9`/`50`/`80` (could-not-start / install / WATCHDOG lost / failed) → CRITICAL;
  `41` (missing status) → HEALTH_CHECK_ERROR; unknown → WARN.
- **HEALTH_CHECK_ERROR** — the check threw.

### `cep` (tags: `local`)
The embedded Siddhi Complex-Event-Processing engine (runs in-process in the controller's
DataPlaneService; the standalone cep plugin was removed).

- **TEMPORARILY_UNAVAILABLE** — dataplane not ready (`"dataplane not ready"`), or Siddhi still
  initializing (`"cep engine initializing"`).
- **OK** — ready (`"cep OK: <n> active query"` / `"… queries"`, from the active-query count).
- **HEALTH_CHECK_ERROR** — the check threw.

## Controller — mesh & link checks

Registered on every controller; a leaf agent has no children and reports OK.

### `link:parent` (tags: `link`, `link:parent`)
Parent-controller reachability — the **only** check that drives a state transition (a `CRITICAL` here
fires `regionalControllerLost`/`globalControllerLost` via `HealthMinaBridge`). Interval
`health_link_interval_sec` (default **5 s**), grace `health_link_grace_sec` (default **10 s**). The
verdict is computed **locally** from the last-pong age (staleness threshold `health_link_stale_ms`,
else `max(2×ping interval, 10 s)`); only the ping/pong crosses the wire.

- **OK** — pong age < stale threshold (`"<label> link ok (pong age <age>ms)"`); or no parent link on
  this role (global/standalone: `"no parent link (<mode>)"`); or a region→global link that uses the
  broker-bridge signal instead of a ping (`"global link via broker bridge …"`).
- **TEMPORARILY_UNAVAILABLE** — pong stale (`"<label> link stale (pong age …)"`), or state/watcher
  not ready. (Grace, default 10 s, promotes to CRITICAL → the loss event fires.)
- **HEALTH_CHECK_ERROR** — the check threw.

### `link:quality` (tags: `link`, `link:quality`)
Parent-edge **quality** from continuous [link metrics](metrics.md#controller-network-link-metrics-netlink).
Deliberately never CRITICAL — quality never drives failover. Interval `health_link_interval_sec`
(5 s).

- **OK** — metrics not ready, or no RTT samples yet (`"no samples yet for <path>"`), or all
  thresholds clear (`"parent link ok [<path>] srtt=… jitter=… sendLat=…"`).
- **WARN** — any threshold exceeded (`"parent link degraded [<path>]: <why>"`), where `why` lists
  whichever fired: `rttHi` > `link_quality_rtt_warn_ms` (default **50**), `jitter` >
  `link_quality_jitter_warn_ms` (default **25**), `sendLat` > `link_quality_sendlat_warn_ms`
  (default **25**), `backlog` > `link_quality_backlog_warn` (default **1000**).
- **HEALTH_CHECK_ERROR** — the check threw.

### `subtree` (tags: `mesh`)
Rolls up children's advertised health (worst-wins). Interval `health_subtree_interval_sec` (default
**10 s**). A child older than `health_subtree_stale_ms` (default **30 s**) is aged out, not reported
CRITICAL — this is not a liveness check and never drives a transition.

- **OK** — no children, no fresh children, or all fresh children OK (`"<n> child(ren) ok"`).
- **worst-of-fresh** — `"<n> child(ren), worst=<status> <path> (<detail>)"` (can be WARN /
  TEMPORARILY_UNAVAILABLE / CRITICAL / HEALTH_CHECK_ERROR depending on the worst fresh child).
- **HEALTH_CHECK_ERROR** — the check threw.

## Plugin checks

Each plugin registers its own check (best-effort; a missing Health-Check API bundle never breaks the
plugin) with `tags=[local]`. They share one guard pattern: **TEMPORARILY_UNAVAILABLE** while starting
up (plugin null / not active / subsystem not ready), **OK** with a descriptive message when active,
**WARN** on any exception. They never return CRITICAL directly — a sustained TEMPORARILY_UNAVAILABLE
(a plugin that never comes up) is promoted to CRITICAL by the executor's grace window.

| Check | OK message | TEMPORARILY_UNAVAILABLE when | WARN when |
|---|---|---|---|
| **`executor`** | `executor OK: <active> running / <configured> configured runner(s)` | plugin/​runnerEngine not ready | exception |
| **`filerepo`** | `filerepo OK: <n> file(s) cataloged, repo dir=<dir>` | plugin/​repoEngine not ready | repo dir not writable, or exception |
| **`repo`** | `repo OK: <n> plugin jar(s)` | plugin/​executor not ready | exception |
| **`sysinfo`** | `sysinfo OK: host telemetry available` | plugin/​builder not ready | exception |
| **`wsapi`** | `wsapi OK: wss listening on port <port>` | plugin not active | exception |
| **`stunnel`** | `stunnel OK: <n> configured tunnel(s)` | plugin/​socketController not ready | exception |

The `wsapi` check is registered only **after** the Netty server binds, so "active" implies
"listening". A clean single-node global therefore settles to a full `[OK]` summary across all
checks — broker, dataplane, db, disk, memory, plugins, cep, link:parent, link:quality, subtree, and
the six plugin checks.

## The state machine

The controller runs a state machine over the [controller modes](overview.md#nodes-and-the-hierarchy)
(`STANDALONE` / `AGENT` / `REGION` / `REGION_GLOBAL` / `GLOBAL`). Node, plugin, and edge status
(`ACTIVE`, `STALE`, `LOST`, …) is persisted per-controller in **Derby**; regional and global
watchdogs age out stale nodes. `StatusAdapter` maps legacy node/plugin status codes into Felix HC
statuses. The health subsystem and the state machine together give the fabric its self-healing
behavior. Design detail: the health-check design doc via [Design Docs](../reference/design-docs.md).
