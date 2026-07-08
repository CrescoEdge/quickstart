# Cresco Health, State & Inventory — Design

Status: **IMPLEMENTED + SHIPPED** (health subsystem built on Felix HC — `controller/health/`:
`CrescoHealthExecutor`, `LocalHealthChecks`, `MeshHealth`, `link:parent`/`link:child` + broker/db/
dataplane/disk/memory/plugin checks; Felix HC api-2.0.4 jar staged; plugins implement `HealthCheck`).
The §3 state-predicate bugs are fixed in code. This doc is the design of record for that shipped work.
· Scope: **Health subsystem first** (measurements and benchmark separation are tracked separately —
measurements later shipped too, see [`metrics-unification-design.md`](metrics-unification-design.md)).
Author: design pass, 2026-07-01.

Decisions locked with the user going in:
- **Full Felix Health Checks** (`org.apache.felix.hc`) as the health engine — not a thin facade;
  plugins implement `org.apache.felix.hc.api.HealthCheck`.
- **State stays in MINA; health moves to HC.** MINA remains the **sole state authority** (keep the
  FSM and the `ControllerMode` states as-is — no Role×Phase enum restructure). HC is the **health
  authority**. The two are bridged by **events**: HC produces verdicts, and only a *post-grace*
  verdict fires an existing MINA transition. See §7.
- **Design doc before code.** This is that doc. **Start with health.**

---

## 1. Goal

Move all health/liveness logic out of the state layer and onto **one health model** built on Felix
HC, leaving MINA to do clean state transitions fed by clean events, such that:

1. Every health verdict (local subsystem, parent link, child liveness, plugin) speaks **one status
   vocabulary** and is **queryable**, not reactive-only.
2. Transient volatility (a missed ping, a GC pause, a flapping link) **cannot** cause a spurious
   `FAILED`/full-re-init. Anti-flap is a standard primitive (HC grace), applied uniformly, so a flap
   is absorbed **before** any MINA event fires.
3. What is computed **in-JVM** vs what must be **communicated** is explicit and minimal.
4. **Health is de-tangled from state.** MINA stops being fed by ad-hoc watcher retry/timer logic;
   the derived-state predicate bugs (§3) are fixed directly. "Degraded" becomes an HC *health
   status*, not a control state.

Non-goals for this tranche: the measurement/metrics unification and the benchmark isolation (separate
efforts). This doc only touches health where it overlaps them — e.g. retiring the dead
`executeKPI`-as-health idea.

---

## 2. Where health lives today

| Mechanism | File | What it does | Anti-flap |
|---|---|---|---|
| `AgentHealthWatcher` | `controller/agentcontroller/AgentHealthWatcher.java` | agent → region: DB watchdog + active RPC ping | **none** — 1 miss ⇒ `regionalControllerLost` |
| `RegionHealthWatcher` | `controller/regionalcontroller/RegionHealthWatcher.java` | region → global ping + agent status tracking | retries(2)+jitter+3-miss tolerance (recent fix) |
| `GlobalHealthWatcher` | `controller/globalcontroller/GlobalHealthWatcher.java` | global → regions: DB-staleness only | none, hardcoded 45s |
| Node status ladder | `db/NodeStatusType.java` | `STARTING, ACTIVE, PENDINGSTALE, STALE, PENDINGLOST, LOST, ...` for child nodes | the `PENDING*` states ARE a bespoke grace ladder |
| Plugin status | `controller/agentcontroller/PluginNode.java` | int codes: `10` working, `40` WATCHDOG STALE, `50` WATCHDOG LOST, ... | the STALE→LOST step is a *third* bespoke grace ladder |
| Controller role/lifecycle | `statemachine/ControllerSMHandler.java` (MINA FSM) + `core/ControllerStateImp.java` | ~20-state `ControllerMode` FSM; health failures drive transitions | n/a |
| Health via KPI | `library/plugin/Executor.executeKPI()` | intended plugin-health hook | **dead** — all 8 impls `return null` |

Three observations drive the whole design:

- **The same OK→transient→dead ladder is hand-rolled three times** (`NodeStatusType` PENDING*,
  plugin `40→50`, and the region ping retry/miss counters) — inconsistently. Felix HC has exactly
  one: `OK → TEMPORARILY_UNAVAILABLE →(grace)→ CRITICAL`.
- **Robustness is applied unevenly.** Region got retry+tolerance; Agent did not, so an agent still
  drops on a single missed ping — the same class of bug we just fixed one level up.
- **`executeKPI` as a health hook is dead.** Nothing uses it. Health has no contract today.

---

## 3. State-layer predicate bugs — RESOLVED (independent of the model)

`ControllerMode` (in `library/agent/ControllerMode.java`) is a flat cross-product of role
`{STANDALONE, AGENT, REGION, GLOBAL}` × phase `{INIT, ACTIVE, FAILED, SHUTDOWN}` — ~20 values. Meaning
is reconstructed with hand-maintained OR-chains and `toString().startsWith(...)`, which had broken as
below; these predicates are now **fixed in code** (`ControllerStateImp`):

- `ControllerStateImp.isFailed()` no longer includes `REGION_GLOBAL` (the **healthy active** state) —
  it enumerates only the `*_FAILED` modes, so a region+global controller no longer reports
  **`isActive()==true` AND `isFailed()==true`** at once. **Resolved.**
- `ControllerStateImp.isActive()` now **includes `REGION`**, so a region-only controller correctly
  reports active. **Resolved.**
- `isRegionalController()` / `isGlobalController()` used `startsWith("REGION")` / `startsWith("GLOBAL")`
  on the enum name — stringly-typed role logic (tidied alongside the above).

**The `ControllerMode` model is kept (MINA owns it — §7) and the predicates were fixed directly**:
the correct set is enumerated explicitly. These were targeted correctness fixes, not a model rewrite.

---

## 4. Design principle: separate the *verdict* from the *signal*

> **All health verdict logic is centralized in-JVM (Felix HC over locally-held data). The only thing
> communicated over the fabric is the raw liveness signal that keeps that local data fresh.**

Every check — even "is my child agent alive?" — becomes a **local** HC check reading a
locally-held value. The messaging layer's only job is to keep those local values current
(outbound pings, inbound pongs/heartbeats stamping timestamps).

| Concern | Centralized in-JVM (HC check reads local state) | Communicated (fabric) |
|---|---|---|
| Broker up / not flow-blocked | `broker` check | — |
| DB reachable / disk headroom / JVM heap | `db`, `disk`, `memory` checks | — |
| Dataplane (JMS) connected | `dataplane` check | — |
| Each plugin's status | `plugin:<id>` check (reads `PluginNode.status_code`) | — |
| **Parent liveness** | `link:parent` reads `lastPongAt` | outbound ping / inbound pong |
| **Child liveness** | `link:child:<id>` reads `lastHeartbeatAt[id]` | inbound heartbeats |
| Node role + lifecycle **state** | **MINA FSM** (unchanged) | — |
| Full node/fabric snapshot | InventoryPrinter (renders local state) | heartbeats populate the child map |

Consequence: the three `*HealthWatcher` classes collapse to **transport only** — send ping, receive
pong, stamp a timestamp. All retry/jitter/threshold/flap logic leaves them and becomes HC config.

---

## 5. The one health vocabulary

Adopt `org.apache.felix.hc.api.Result.Status` as the single ladder everywhere:

`OK` · `WARN` (functional, trending bad) · `TEMPORARILY_UNAVAILABLE` (not functional now, expected to
self-recover; executor auto-promotes to `CRITICAL` after a grace window) · `CRITICAL` (not functional,
do not use) · `HEALTH_CHECK_ERROR` (status uncomputable).

The three existing ladders map onto it directly — this is the migration table, not a rewrite:

**`NodeStatusType` → `Result.Status`**

| NodeStatusType | Result.Status |
|---|---|
| `STARTING` | `TEMPORARILY_UNAVAILABLE` |
| `ACTIVE` | `OK` |
| `PENDINGSTALE` | `TEMPORARILY_UNAVAILABLE` |
| `STALE` | `WARN` |
| `PENDINGLOST` | `TEMPORARILY_UNAVAILABLE` |
| `LOST` / `FAILED` | `CRITICAL` |
| `ERROR` | `HEALTH_CHECK_ERROR` |
| `DISABLED` | not registered / excluded from aggregate |

**Plugin `status_code` → `Result.Status`**

| status_code | meaning | Result.Status |
|---|---|---|
| `3` | agentcontroller init | `TEMPORARILY_UNAVAILABLE` |
| `10` | started and working | `OK` |
| `40` | WATCHDOG STALE | `TEMPORARILY_UNAVAILABLE` (→ WARN if sticky) |
| `50` | WATCHDOG LOST | `CRITICAL` |
| `7,9,80` | could-not-start / bundle-fail | `CRITICAL` |
| `8` | disabled | excluded |
| `90,91,92` | shutdown timeouts | `WARN` |
| `41` | missing status param | `HEALTH_CHECK_ERROR` |

`NodeStatusType` and the int codes need not be deleted immediately — they become **projections** of
the HC verdict (kept for DB/back-compat), written *from* the check result rather than driving logic.

---

## 6. Health check catalog

All checks implement `org.apache.felix.hc.api.HealthCheck` and are registered as OSGi services with
`hc.name`, `hc.tags`, and (for scheduled ones) `hc.async.intervalInSec`.

### 6.1 Local checks — tag `local` (pure in-JVM, no messaging)

| Check | `hc.name` | Reads | CRITICAL when |
|---|---|---|---|
| Broker | `broker` | `ActiveBrokerManager` / broker service state | broker not started / connector down |
| DataPlane | `dataplane` | JMS connection in `DataPlaneServiceImpl` | JMS connection closed |
| DB | `db` | `DBEngine` ping | Derby unreachable |
| Disk | `disk` | KahaDB + `cresco-data` dir free space | below floor (configurable) |
| Memory | `memory` | JVM heap | above ceiling |
| Plugin | `plugin:<id>` | `PluginNode.status_code` per plugin | code ∈ {7,9,50,80} |

### 6.2 Link checks — tag `link` (verdict local, signal communicated)

- `link:parent` — reads `lastPongAt` maintained by the (now transport-only) parent watcher.
  `now - lastPongAt < interval` ⇒ `OK`; within grace ⇒ `TEMPORARILY_UNAVAILABLE`; beyond grace ⇒
  `CRITICAL` (auto-promoted by the executor). Uses `hc.keepNonOkResultsStickyForSec` so a flap that
  self-heals still surfaces in the record.
- `link:child:<id>` — on region/global, one check per child reading `lastHeartbeatAt[id]`. Same
  semantics. This is how a parent's view of children becomes local checks over communicated data.

### 6.3 Aggregates and how they reach MINA

- `execute(tags=["local"])` → **local health** → surfaced as node health status (queryable via
  Inventory/JMX). Hard local failures (broker/db/dataplane `CRITICAL`) may fire a MINA fail event
  (see O3); non-fatal ones (one plugin) are reported as `DEGRADED` health **without** a state change.
- `execute(tags=["link"])` → **link health** → `link:parent` `CRITICAL`-after-grace **fires the
  existing MINA event** `regionalControllerLost` (agent) / `globalControllerLost` (region). MINA
  handles it exactly as today. Because grace lives in HC, MINA only ever sees a *real* loss.
- `execute(tags=["local","link"])` (OR) → the node's overall health, exposed via Inventory + JMX.

Grace/interval come from config, defaulted to match today's effective ~5s ping timeout × tolerance so
recovery latency is unchanged (see Open Question O2).

---

## 7. State stays in MINA; health moves to HC — the bridge

### 7.1 Separation of authority

- **MINA is the state authority.** Keep `ControllerSMHandler` (the `@State`/`@Transition` FSM), the
  `ControllerMode` enum, and the transitions. MINA does a good job of state; we do not restructure it.
- **HC is the health authority.** All liveness/verdict logic lives in HC checks (§6). HC owns no
  state; it produces verdicts.
- **The bridge is events, one-directional (HC → MINA).** A health verdict that crosses a threshold
  fires an **existing** MINA event. HC replaces the *watchers* as the event source; MINA remains the
  *state owner and actuator*. The anti-flap grace lives entirely in HC, so a transient blip is
  absorbed **before** any MINA event fires — this is the fix for "one missed ping nukes the node,"
  achieved **without touching MINA's states**.

```
 fabric signal            in-JVM verdict (HC)                 state (MINA)
 ─────────────            ───────────────────                 ────────────
 pong  ─────► lastPongAt ─► link:parent check ─(CRITICAL       fire regionalControllerLost
                            after grace)──────────────────────► / globalControllerLost  ─► FAILED ─► recover
 heartbeat ─► lastHb[x] ──► link:child:x check ─► child health projection (NodeStatusType in DB)
 (none)      broker/db ───► local checks ───────► node health status;  hard CRITICAL ─► (O3) fail event
```

### 7.2 What changes in the state layer (minimal)

- **No enum restructure.** `ControllerMode` and the MINA state set are unchanged.
- **Fix the §3 predicate bugs** in `ControllerStateImp` (`isActive`/`isFailed`/`isRegional…`) — a small
  correctness patch; the public `ControllerState` interface is **unchanged** (no back-compat shim
  needed, unlike a model rewrite).
- **Watchers stop calling into state directly with hand-rolled logic.** Today the watchers run their
  own timers/retries and call `regionalControllerLost()`. After this change, the transport-only
  watchers just stamp timestamps; the **HC link check** is what fires the MINA event. Net: the same
  MINA transition, but sourced from a single, grace-protected, uniform place.
- **"Degraded" is a health status, not a MINA state.** A node with one sick plugin stays in `AGENT`
  (MINA) while its health reports `DEGRADED` (HC). Load-shedding or alerting can react to the health
  status without a state transition.

### 7.3 Why this is the "smarter way" (and why MINA stays)

MINA is used in exactly **2 files** (`ControllerSMHandler`, `ControllerEngine`), declared only in
`controller/pom.xml` — isolated and working. The complexity that was flagged earlier came from
**health logic being tangled into the state layer** (per-watcher timers, retry counters, `stateInit`
calls near transitions), not from MINA. Removing health to HC leaves MINA as a clean FSM fed by clean
events — which is exactly the simplification wanted, with **no engine change and minimal state-layer
risk**. The MINA-vs-EnumMap question is therefore **closed: keep MINA.**

---

## 8. Inventory Printer — the read surface (not a state store)

Clarification on intent: `InventoryPrinter.print(PrintWriter, Format, isZip)` is an **on-demand
renderer** (TEXT/JSON/HTML), not a place that "maintains" state. State is held by MINA (control state)
and the HC executor's result cache (health); Inventory **dumps** it.

Register printers (`felix.inventory.printer.*` service props):

| Printer | Renders |
|---|---|
| `node` | role/state (from MINA), identity, config summary |
| `plugins` | plugin table + per-plugin health |
| `fabric` (region/global only) | children + their `link:child` health |
| `health` | latest HC results (all tags) as JSON |

Exposed via **JMX** and through **wsapi** (the existing websocket API). We do **not** enable the Felix
Web Console servlet — it's an unneeded HTTP admin/security surface that overlaps wsapi. This retires
the ad-hoc `getControllerInfoMap()` / `stateUpdateTask` export path.

---

## 9. Concurrency & volatility — what HC absorbs, what stays custom

**HC absorbs** the health-execution concurrency you currently hand-roll:
- Per-watcher `AtomicBoolean` "already running" guards → the HC async executor schedules/serializes.
- `AtomicInteger consecutivePingFailures` + retry/jitter → grace window + `TEMPORARILY_UNAVAILABLE`.
- 4 separate `Timer`s (agent/region/global watchers + `stateUpdateTimer`) → HC async executor.
- Reads are **atomic snapshots**: `HealthCheckExecutor` caches results (`hc.resultCacheTtlInMs`), so
  querying the aggregate is a cheap consistent read, never a re-run storm.

**Stays custom (domain state HC/MINA do not own):**
- The child-inventory map (`lastHeartbeatAt`, plugin table) remains an in-JVM `ConcurrentHashMap`,
  updated by inbound messages. HC checks *read* it; Inventory *renders* it.

**Volatility fix — where it comes from now:** the primary fix is structural — **HC grace absorbs flaps
before a MINA event fires**, so the controller no longer transitions (and re-inits + re-persists) on a
transient blip. Because MINA transitions only on real post-grace loss, the existing per-transition DB
write in `ControllerStatePersistance` is no longer a flap-driven write storm; a secondary tidy-up
(avoid persisting transient INIT churn) can follow but is not the main lever.

---

## 10. Felix HC integration specifics

- **Bundle: API only.** Provision `org.apache.felix.healthcheck.api-2.0.4.jar` (exports
  `org.apache.felix.hc.api`, `.api.execution`, `.api.condition`; imports only `org.osgi.framework` →
  resolves trivially). Install it in the agent bootstrap
  (`agent/src/main/java/io/cresco/main/HostApplication.java`, via `installInternalBundleJars`, before
  library/controller). Jar staged into `agent/src/main/resources/`.
- **Why NOT `.core`:** Felix HC **core 2.3.0** is an R8 bundle with mandatory imports absent from
  Cresco's minimal R7 Felix — `org.osgi.service.condition [1.0,2)` and `org.osgi.service.servlet.context
  [2.0,3)` (R8; `osgi.cmpn-7.0.0` is R7), plus `org.slf4j [1.7,2)` which *excludes* Cresco's SLF4J 2.x.
  Provisioning core would force a compendium upgrade in a stripped Felix (high resolution risk, blocks
  testing) for an executor whose behavior we want to own and tune for the mesh anyway.
- **Executor is controller-owned** (`io.cresco.agent.controller.health.CrescoHealthExecutor`):
  discovers `HealthCheck` services by `hc.tags` via a `ServiceTracker`, runs them on a scheduler, and
  applies **grace** (`TEMPORARILY_UNAVAILABLE`→`CRITICAL` after a window), **sticky** non-OK, and a
  **result cache** (atomic snapshot reads). This is where the mesh-tuning lives. Checks are real
  `org.apache.felix.hc.api.HealthCheck` services with `hc.name`/`hc.tags` props, so the Felix HC
  **core bundle remains a drop-in** replacement if Cresco ever moves to an R8 compendium.
- **Surfaces:** JMX + wsapi. **No** webconsole servlet.

---

## 11. Phased implementation

Each phase is independently reviewable; earlier phases are useful even if later ones slip. **P1–P5
have shipped** (`controller/health/` + Felix HC api jar staged); P6 cleanup is the trailing tidy-up.

- **P1 — Foundation. ✅ DONE.** Add HC bundles; `health` package; `Result.Status` adopted;
  `NodeStatusType`/plugin-code adapters (§5). No behavior change yet.
- **P2 — Local checks. ✅ DONE.** `broker/dataplane/db/disk/memory/plugin` checks, tag `local`
  (`BrokerHealthCheck`, `DataPlaneHealthCheck`, `DbHealthCheck`, `MemoryHealthCheck`, `LocalHealthChecks`).
  Live + queryable. Zero messaging. Lowest risk.
- **P3 — Link checks + watcher slimming. ✅ DONE.** Watchers → transport-only (stamp timestamps);
  `link:parent` + `link:child` checks with grace/sticky (`MeshHealth`/`MeshHealthChecks`). **Agent
  robustness fixed here for free** (same check as region).
- **P4 — HC→MINA bridge + predicate fixes. ✅ DONE.** Link-check `CRITICAL`-after-grace fires the
  existing `regionalControllerLost`/`globalControllerLost` MINA events; retire the watchers' direct
  calls; fix the §3 `isActive`/`isFailed` predicates (now fixed in `ControllerStateImp`). **MINA states
  unchanged.**
- **P5 — Inventory. ✅ DONE.** `node/plugins/fabric/health` printers; wsapi + JMX (`HealthInventory`);
  retire `getControllerInfoMap`/`stateUpdateTask`.
- **P6 — Retire custom machinery.** Delete watcher atomics/counters/Timers; `NodeStatusType`/int codes
  become projections; remove dead `executeKPI`-as-health. (Trailing cleanup.)

**Proof harness (built alongside P3):** a **link-flap reproduction** in `run/tests` — drop pongs for
N seconds and assert the node stays in its MINA state (no `regionalControllerLost` fired, no re-init)
until real loss beyond grace. The health analogue of `broker-bench/StarvationRepro`.

---

## 12. Open questions (need decisions before/at each gate)

- **O1 (P1):** Do Cresco plugins implement `org.apache.felix.hc.api.HealthCheck` directly (full Felix,
  as chosen), confirming plugins may import the Felix HC API package? Assumed **yes** per the locked
  decision; flagging because it puts a Felix package on the plugin classpath.
- **O2 (P3):** Grace/interval defaults. Match today's effective detection latency (~5s ping ×
  tolerance) or deliberately widen for edge/volatile links? Affects failover speed.
- **O3 (P4):** Which **local** CRITICALs fire a MINA fail event vs just report `DEGRADED` health?
  Proposal: broker/dataplane/db CRITICAL ⇒ fire a fail/re-init event; a single plugin CRITICAL ⇒
  `DEGRADED` health only (no state change). Needs the specific MINA event(s) for local-subsystem loss
  identified (today there may be no clean one — may add `localSubsystemFailed`).
- **O4 (P5):** Expose Inventory JSON through wsapi to external clients (pycrescolib), or JMX-only?

---

## 13. Risks

- HC executor lifecycle across OSGi bundle refresh (felix-hygiene refreshes Felix bundles) — validate
  the executor survives refresh.
- Recovery-latency regression if grace is mis-tuned (O2) — the flap-repro guards against *spurious*
  failover; a separate test must confirm *real* failover still happens promptly.
- The HC→MINA event bridge must fire the existing events with the same semantics the watchers used, so
  the recovery path (`stateInit`) behaves identically — inventory of who currently calls
  `regionalControllerLost`/`globalControllerLost` required in P4.
