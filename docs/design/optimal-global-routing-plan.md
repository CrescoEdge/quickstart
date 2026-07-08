# Cresco — Optimal, Tenant-Aware, QoS-Enforced Global Routing — Implementation Plan

**Date:** 2026-07-03 · **Updated:** 2026-07-06
**Status:** PARTIALLY IMPLEMENTED. Phases **A** (redundant mesh + activate cost routing), **C** (route
computation + per-class-capable path math), and **D** (source-routing data plane) have **shipped**, taking
the *distributed-cost-first* path of deviation **D1** (each node builds a mesh-wide link-state view from
**pushed** advertisements and runs Dijkstra locally, rather than a central FIB push) and adding a
**self-organizing link-inference** capability beyond the original plan. See the shipped-feature reference:
[Dynamic Cost-Aware Routing](../architecture/dynamic-routing.md). Phase **B**'s L1 tenant namespacing has
since **shipped + proven** (flag `tenant_namespacing`, `tenant_namespacing_test.sh` 12/12 — see
[`tenant-isolation-design.md`](tenant-isolation-design.md)); only its L3 bridge-scoping (per-bridge
allowed-tenant forwarding) + multi-tenant carriage over the redundant paths remain. Phases **E**
(per-tenant fairness) and **F** (joint routing/capacity) remain. Re-baselines an external architecture
report against Cresco's actual in-tree state and sequences the real remaining work.
**No calendar/effort estimates here** — phases are a *dependency order*, not a schedule.

!!! success "Implemented 2026-07-06"
    W1 (redundant mesh + `net_cost_routing` selection), W3 (route computation — distributed variant D1, via
    pushed link-state `RouteView` + `RouteComputer` Dijkstra), W4 (source-routing data plane —
    `net_source_routing`, `srcroute` waypoint stacks, `MsgRouter.advanceSourceRoute`), and W8's hysteresis
    (`PathTable`, `net_route_hysteresis_ms`) are live and proven in the containerlab mesh. **New beyond the
    plan:** self-organizing inference — a region learns an un-configured peer's address from link-state and
    forms a direct link when it is faster (`RegionHealthWatcher.inferConnections`).
**Siblings:** [`link-metrics-design.md`](link-metrics-design.md) (metrics/cost/tuning — *shipped*),
[`distributed-identity-trust-design.md`](distributed-identity-trust-design.md) (identity/trust/tenant
authz — *shipped through Phase 2*), [`region-federation-design.md`](region-federation-design.md),
[`broker-performance.md`](broker-performance.md), [`health-check-design.md`](health-check-design.md).

---

## 0. The one thing to read first

The external report proposes a hybrid SDN + segment-routing control plane with cryptographic tenant
isolation, multi-topology per-class routing (LARAC/widest-path), hop-by-hop H-DWRR fair queuing, and
joint routing/capacity optimization. **The architecture is sound. Its sequencing is wrong for Cresco**,
because it assumes a near-greenfield build. In fact **the entire measurement/cost/actuation substrate
and the entire security/trust/tenant-authz substrate are already implemented and proven live in this
tree.** The remaining work is a fraction of what the report describes, in a different order, with several
places where we should deliberately build *less* than the report specifies.

This plan (a) marks what is done, (b) states the deviations and why, (c) sequences only the remaining
work against real classes and validation gates.

---

## 1. Re-baseline — research building blocks vs. Cresco reality

| Research building block | Cresco status | Where (code / doc) |
|---|---|---|
| Out-of-band link telemetry (RTT/jitter/loss/backlog) | **DONE + PROVEN** | `controller/netmetrics/LinkMetrics`, `LinkMetricsRegistry`; RTT harvested from existing ping RPC (agent→region **and** region→global bridge edge). `link-metrics-design.md` §3–4 |
| EWMA metric smoothing | **DONE** | `LinkMetrics` uses Jacobson/Karels SRTT + RTTVAR (stronger than the report's flat α=0.15) |
| Backlog / congestion signal | **DONE** | `ActiveBroker.getBrokerPendingBacklog()` (JMX `DestinationViewMBean.QueueSize`), polled by AutoTuner |
| Central metric aggregation at global | **PARTIAL** | Global already aggregates edge health (`GlobalExecutor`, `DBInterfaceImpl.getEdgeHealthStatus`); LinkMetrics are **not** yet streamed up as a topology graph |
| Composite link cost model | **DONE (gated)** | `LinkMetrics.cost()`, `LinkMetricsRegistry.costOf/lowestCostEdge`; `MsgRouter` per-hop cost annotation behind `net_cost_routing` |
| Dynamic capacity (spin up/down bridges) | **DONE** | `ActiveBroker.addBridgeConnections/removeBridgeConnections`, driven by `AutoTuner` — **in-process, not the JMX MBean API the report assumes** |
| Automated tuning control loop | **DONE (gated)** | `AutoTuner` (buffers/block/connections to BDP + saturation), `NetTuningProfile`, gated by `net_autotune` |
| Cryptographic tenant identity (O=tenant in cert DN) | **DONE + PROVEN** | `CrescoIdentity`, `CertificateManager` identity-bearing leaves |
| mTLS binding of identity at broker | **DONE + PROVEN** | `broker_require_client_auth`; `CrescoAuthorizationBroker.addConnection` derives principal from validated cert DN (anti-spoof proven) |
| Regional-CA trust distribution (O(regions), not O(n²)) | **DONE + PROVEN** | `RegionCA`, `CertificateManager.issueEnrollment/installEnrollment`; `security_regional_ca`; `run/tests/regional_ca_mtls_test.sh` 10/10 |
| Per-destination tenant ACL enforcement | **DONE + PROVEN** | `TenantPolicy` + `CrescoAuthorizationBroker` (BrokerFilter on addConsumer/addProducer/send), `broker_security_enabled`; `tenant_isolation_mesh_test.sh` 10/10 |
| Selective message signing primitive | **DONE** | `MessageSigner` (sign/verify payload digest) |
| Loop safety in a mesh | **DONE** | `BrokerPath` + TTL (`MsgRouter.getTTL`), native bridge loop suppression |
| **Tenant destination namespacing** (`T.<tenant>.<raw>`) | **DONE + PROVEN (L1); L3 bridge-scoping TODO** | Shipped flag `tenant_namespacing` (`TenantNamespace`/`TenantPolicy`, `tenant_namespacing_test.sh` 12/12 — `distributed-identity-trust-design.md` §5, `tenant-isolation-design.md`); *the keystone for both routing and isolation.* Per-bridge tenant-scoped forwarding (L3) still to build. |
| **Redundant federation paths** (distinct alternate routes, not parallel sockets to one peer) | **TODO** | Topology is still a strict tree; `addBridgeConnections` makes parallel sockets to the *same* peer, not alternate paths |
| **Central route computation + routing-table push** (SDN) | **TODO** | No global topology graph / path solver / pushed FIB |
| **Segment / source-routing data plane** | **TODO** | `MsgRouter` has a hop trail + per-hop cost, not a source-computed segment stack |
| **Multi-topology per-class routing** (4 logical topologies) | **TODO** | Single composite cost today; no per-class Dijkstra/widest-path/LARAC |
| **Per-tenant fair scheduling** (H-DWRR) | **TODO** | QoS is global 4-tier (`MsgQoS` + isolated `ControlPlaneSender`); no per-tenant fairness |
| **Payload envelope encryption** (per-tenant AES-256-GCM) | **TODO (primitive exists)** | mTLS protects in-flight; `MessageSigner` exists; per-tenant confidentiality-at-rest-in-transit is unbuilt |
| **Joint routing + capacity (MCF/BwE, tenant quotas)** | **TODO** | AutoTuner does local saturation scaling; no global MCF / bandwidth-function starvation control |
| **Path-switch stability** (hysteresis, hold-down, flowlet) | **TODO** | No path switching exists yet, so nothing to damp — needed once selection is active |

**Takeaway:** the report's Phase 1 (telemetry) is done, its Phase 2 metric/control half is done, and its
Phase 3 security foundation (mTLS + regional CA + tenant authz) is done. What remains is **namespacing,
redundant topology, central computation, source routing, per-class routing, per-tenant fairness,
encryption tier, and joint capacity** — plus the stability controls that only matter once path selection
is live.

> **⚠ Metric-source volatility (read with §4/W3).** Metrics are *actively being centralized* onto the
> unified Micrometer `MeasurementEngine` + `getmetrics`/`getmetricinventory` contract (see
> [`metrics-unification-design.md`](metrics-unification-design.md)); the `netlink`/LinkMetrics group
> already lives there. **So every specific metric reference in this plan — class location
> (`controller/netmetrics/…`), field name (`link.<path>.rtt_ms`), poller (`getBrokerPendingBacklog`) — is
> a moving target and will be re-bound as unification lands.** This plan therefore commits only to metric
> *signals by role* — **RTT, jitter, loss, throughput, send-latency, backlog, NIC ceiling** — and to the
> unified metrics surface as the binding point. The architecture is metric-source-agnostic by design; the
> route-compute and cost inputs read whatever the unified registry exposes for those roles at build time.
> **General planning proceeds now; the concrete metric bindings are (re-)verified at each phase's start,
> not frozen here.**

---

## 2. The gating reality: there is no redundant path yet

`net_cost_routing` already annotates per-hop cost, and `lowestCostEdge` already exists — but **min-cost
selection has nothing to choose because the mesh is a strict tree** (agent→region→global, one path).
`link-metrics-design.md` §5 states this plainly: connection actuation is "validated for correctness;
needs a multi-region mesh to validate throughput."

**Therefore the #1 unblocker is standing up a redundant multi-region mesh** (alternate region↔region /
region↔global bridges forming real cycles). Nothing in routing — cost selection, per-class topologies,
segment routing, capacity optimization — can be validated until redundant paths physically exist. This
comes first, and it simultaneously validates the cost-routing layer that is already built but dormant.

---

## 3. Target architecture and deliberate deviations from the report

Adopt the report's end state: **SDN control (global) + source/segment routing data plane + per-tenant
namespacing + per-tenant fair scheduling + joint routing/capacity + stability damping.** Deviate on
sequencing and scope as follows, each for a concrete engineering reason:

- **D1 — Distributed-local cost routing first; central SDN layered on second.** The local cost model is
  *already built* (`net_cost_routing`); on a redundant mesh it yields correct min-cost next-hop with zero
  new control-plane machinery. Ship that, then add central computation for global-optimal / multi-constraint
  cases. Bonus: local cost routing becomes the **graceful-degradation fallback** when the central
  controller is unreachable or its pushed table is stale — turning the report's SDN single-point-of-failure
  into a non-issue. *(This is the main structural deviation from the report, which leads with central SDN.)*

- **D2 — Right-size the path math.** Start with per-class **Dijkstra (latency/loss)** and **widest-path
  (bulk)** — cheap, well-understood, and enough for the common case. Defer **LARAC / multi-commodity-flow
  / BwE bandwidth functions** until a measured need exists (a hard jitter SLA on control traffic, or proven
  multi-tenant link contention). Do not build an NP-complete MCP solver before there is redundancy to run
  it over.

- **D3 — Per-tenant scheduling at contention points, not every hop.** The report puts H-DWRR in *every*
  transit broker's dispatch loop. Contention happens at **ingress (rate-limit)** and **shared bridge egress
  (fair dispatch)** — schedule there first. Implement it by **extending the existing
  `CrescoAuthorizationBroker` BrokerFilter chain** (it already intercepts `send`), not a separate plugin
  wired at every node. Add transit-hop scheduling only if measurements show mid-mesh contention.

- **D4 — Provision bridges via the in-process API we already have.** Use
  `addBridgeConnections/removeBridgeConnections` (AutoTuner already drives them), not the JMX MBean API the
  report suggests. Same-process, already tested, no remote-JMX surface to secure.

- **D5 — Payload encryption is a tiered opt-in, not a blocking per-tenant KMS.** Inter-broker hops are
  already TLS-encrypted in flight (nio+ssl) and identity-bound (regional CA). The residual threat envelope
  encryption addresses is **payload-at-rest inside a compromised/inspected transit broker**. Make AES-256-GCM
  envelope encryption an **opt-in tier for high-assurance tenants**, reusing the shipped `MessageSigner`
  pattern and the recipient's leaf public key; for most tenants, tenant-namespaced bridge forwarding +
  removing the bridge ACL exemption (W10) is sufficient. Do not gate the routing program on standing up a
  per-tenant symmetric KMS.

- **D6 — Multipath breaks JMS ordering; default to pinning a flow to one path.** Splitting one logical
  stream across parallel paths reorders messages, which order-sensitive JMS consumers do not tolerate.
  Default: **pin a flow (src→dst destination) to a single chosen path**; only switch on an idle gap
  (**flowlet switching**, gap ≥ ~2×RTT). Validate against consumers that assume per-destination order
  before enabling any per-message splitting.

- **D7 — Keep BrokerPath + TTL even under source routing.** Source-computed segment stacks are acyclic by
  construction, but transit brokers must still validate the segment and honor `BrokerPath`/TTL as
  defense-in-depth against a malformed or stale header.

---

## 4. Workstreams (the real remaining build)

Each: goal · primary integration point · depends-on · validation.

- **W1 — Redundant federation topology + activate cost routing.** Stand up alternate region↔region /
  region↔global bridges (real cycles); enable `net_cost_routing` selection. *Integration:* `ActiveBroker`
  bridge setup, `MsgRouter` (activate `lowestCostEdge` selection). *Depends:* multi-region test env.
  *Validate:* failover time on link degrade; path matches oracle min-cost.

- **W2 — Tenant destination namespacing + bridge tenant filtering.** **L1 namespacing shipped + proven**
  (`T.<tenant>.<raw>` via `TenantNamespace`/`TenantPolicy`; `AgentProducer` stamps+qualifies; ingress drop
  of mismatched-tenant publishes; `tenant_namespacing_test.sh` 12/12). **Remaining (L3):** bridge forwards
  only its tenant namespace (closes the bridge-ACL-exemption leak for the common case). *Integration:*
  `TenantPolicy`, `CrescoAuthorizationBroker`, `MsgRouter` ingress, `ActiveBroker` bridge
  included/excluded destinations. *Depends:* nothing (security substrate + L1 namespacing shipped).
  *Validate:* extend `tenant_isolation_mesh_test.sh` to multi-hop bridged paths. *Flag:* `tenant_namespacing`.

- **W3 — Central route computation (global as route reflector) + FIB push.** Global builds the topology
  graph from per-edge link signals, computes routes, and pushes a routing table to each regional
  `MsgRouter` cache. **Do not invent a new metric-push:** the metrics-unification work is already building
  the region/global metric fan-out (`getmetricinventory` scope=region|global, pull-based on demand — see
  [`metrics-unification-design.md`](metrics-unification-design.md) §3–4). The route-compute service reads
  the link signals (RTT/jitter/loss/throughput/backlog roles) from that unified inventory at whatever
  location/schema they land, rather than binding to today's `netlink` field names. *Integration:*
  route-compute service consuming `getmetricinventory`; CONFIG-message push of the FIB (reuse the
  `nettuning` broadcast pattern). *Depends:* W1; metrics-unification region/global fan-out (Phase 2 there).
  *Validate:* pushed table converges; stale-table fallback to local cost (D1). *Note:* pull-on-demand
  aggregation may need a faster cadence for routing than for observability — confirm the inventory refresh
  interval meets the control loop's needs, or add a lightweight route-scoped pull.

- **W4 — Segment / source-routing data plane.** Ingress encodes the computed path as a segment stack in a
  JMS header (`JMS_Cresco_SegmentPath`); transit brokers pop-and-forward statelessly (validate + TTL/BrokerPath).
  *Integration:* `MsgRouter.route()` (extend `forwardDst` + `routepath` mechanics). *Depends:* W3.
  *Validate:* loop-freedom under induced churn; transit statelessness.

- **W5 — Multi-topology per-class routing.** Four logical topologies over one graph: liveness=min-RTT
  Dijkstra, control=jitter-bounded (LARAC *when justified*, else latency Dijkstra), telemetry=min-loss
  (−ln(1−p) additive), bulk=widest-path. *Integration:* route-compute service (W3), keyed by `MsgQoS.Tier`.
  *Depends:* W3. *Validate:* each class takes its class-optimal path in a rigged topology.

- **W6 — Per-tenant QoS fair scheduling (H-DWRR at contention points).** Two-level scheduler (L1 inter-tenant
  weighted fair share, L2 intra-tenant priority class) at ingress rate-limit + bridge egress. *Integration:*
  extend `CrescoAuthorizationBroker` filter chain; reuse `MsgQoS` tiers for L2. *Depends:* W2 (namespaced
  destinations carry tenant+class). *Validate:* extend `StarvationRepro` — one tenant's bulk flood cannot
  degrade another tenant's control-plane RTT; Jain fairness index across tenants.

- **W7 — Joint routing + capacity provisioning + starvation control.** Global loop: detect link utilization
  ≥ threshold → expand capacity (`addBridgeConnections`, bounded by tenant quota) or reroute low-priority
  class; BwE-style bandwidth functions + per-tenant quotas guarantee baselines before any tenant expands.
  *Integration:* global controller + `AutoTuner` (extend from local to global-informed). *Depends:* W3, W5,
  W6. *Validate:* under overload, guaranteed baselines honored; no tenant monopolizes shared bridges.

- **W8 — Path-switch stability.** Cost hysteresis (switch only if alternate < current / 1.5), hold-down
  timer (~30s per flow), flowlet switching (D6). *Integration:* `MsgRouter` selection + route-compute.
  *Depends:* W1 (something to switch). *Validate:* induced flapping link → bounded switch rate, no
  reordering to order-sensitive consumers.

- **W9 — Payload encryption tier (opt-in).** AES-256-GCM envelope, per-tenant key, routing metadata stays
  clear in JMS headers; decrypt at egress. *Integration:* ingress/egress `MsgRouter`, reuse `MessageSigner`
  + leaf public keys. *Depends:* W2. *Validate:* transit-broker store inspection yields ciphertext only.

- **W10 — Security hardening (adjacent, do alongside).** (a) Replace the weak discovery-secret crypto
  (AES-ECB / SHA-1 KDF / hardcoded salt in `DiscoveryCrypto`) with AES-GCM + a proper KDF. (b) Decide the
  bridge ACL exemption policy — W2 namespacing lets bridges enforce a tenant-prefix rule instead of blanket
  trust. *Validate:* join handshake still passes; bridges federate under the new rule.

---

## 5. Phased delivery (dependency-ordered, not a schedule)

Dependency-ordered, each flag-gated and default-off so the fabric is never changed blind (the same
discipline `distributed-identity-trust-design.md` §5 used for the security fabric).

| Phase | Theme | Workstreams | Exit criteria / gate |
|---|---|---|---|
| **A — Unblock & prove** | Redundant mesh + activate the *already-built* cost routing | W1 | Redundant multi-region mesh forms; `net_cost_routing` selects min-cost path; failover on link degrade measured. **Validates the dormant shipped layer.** |
| **B — Isolation substrate** | Tenant namespacing + bridge filtering; security hardening; encryption tier | W2, W10, (W9 opt-in) | Cross-tenant denied on **multi-hop bridged** paths; discovery crypto hardened; `tenant_namespacing` on in a 2-tenant mesh |
| **C — Central brain** | Route computation + push; per-class topologies (simple first); stability | W3, W5 (Dijkstra/widest-path), W8 | Global computes + pushes routes; each `MsgQoS` class takes its class-optimal path; stale-table falls back to local cost; no route flap |
| **D — Source routing** | Segment-stack data plane | W4 | Transit brokers stateless pop-and-forward; loop-free under churn |
| **E — Fairness** | Per-tenant H-DWRR at contention points | W6 | `StarvationRepro` multi-tenant: bulk flood by tenant X cannot starve tenant Y's control RTT; fairness index within target |
| **F — Traffic engineering** | Joint routing+capacity, quotas, BwE; LARAC/MCF *if justified* | W7, W5 (LARAC/MCF) | Under overload, per-tenant baselines guaranteed before expansion; no shared-bridge monopoly |

Phases A–B are the high-value near term (activate what's built + close the multi-hop isolation gap).
C–F are the genuinely new control-plane engineering, each independently shippable behind its flag.

---

## 6. Decisions needed (I've flagged recommendations; these change plan shape)

1. **Route-computation primary model.** *Recommend D1:* distributed-local cost routing first (built,
   lowest effort, doubles as SPOF fallback), central SDN layered on in Phase C. Alternative: central-SDN-first
   per the report (more upfront work, no fallback until it lands).
2. **Namespacing scheme (final).** Confirm `T<tenant>.<priorityClass>.<region>.<agent>` (matches the identity
   doc). Blocks W2 and everything downstream.
3. **Payload encryption.** *Recommend D5:* opt-in tier. Alternative: mandatory envelope on all tenant traffic
   (higher assurance, real throughput cost, needs per-tenant KMS).
4. **H-DWRR scope.** *Recommend D3:* contention points (ingress + bridge egress) first. Alternative: every
   transit hop per the report (more isolation, more dispatch-loop overhead).
5. **Multi-region test environment.** W1 (and thus everything) needs a redundant multi-region mesh to
   validate. Is there hardware/CI for this, or do we validate on same-host multi-global first
   (per the `multi-global-mesh` learnings)?

---

## 7. Risks & mitigations

- **No redundant path today blocks all routing work** → Phase A first; it also finally validates the built
  cost layer.
- **Central controller SPOF / stale FIB** → local cost routing (built) is the always-on fallback (D1).
- **Message reordering under multipath** → pin-flow default + flowlet switching + order-sensitive-consumer
  validation before enabling splits (D6).
- **Scheduler overhead in the dispatch hot path** → benchmark-gate every scheduler change against
  `broker-bench` and `StarvationRepro`; keep the fast path unchanged when `broker_security_enabled` /
  scheduling flags are off.
- **Tenant namespacing is fabric-breaking** (like security Phase 2 was) → flag-gated `tenant_namespacing`,
  staged region-by-region, `vm://` always exempt.
- **Over-engineering (LARAC/MCF/BwE) ahead of need** → gate on measured contention/SLA (D2); the simple
  per-class algorithms cover the common case.

---

## 8. Validation & success metrics

**Reuse:** `broker-bench`, `StarvationRepro`, `run/tests/tenant_isolation_mesh_test.sh`,
`run/tests/regional_ca_mtls_test.sh`, the full bring-up/recovery suite (must stay green with each new flag
**off** and, separately, **on**).
**New:** redundant-path failover test; per-class path-selection test (rigged topology); multi-tenant
fairness test (extend `StarvationRepro`); path-flap stability test.
**Metrics:** path optimality vs. offline oracle; failover time on link degrade; per-tenant Jain fairness
index on a shared bridge; control-plane ping p50/p99 under cross-tenant bulk flood (compare to the
`f157677` control-plane-priority baseline in `broker-performance.md`); **zero** cross-tenant leakage on
multi-hop bridged paths.

---

## 9. One-line summary

Cresco already has the measurement, cost, dynamic-capacity, mTLS, regional-CA, and tenant-authz substrate
the report treats as to-be-built. The real program is: **stand up redundant paths and switch on the
cost routing that already exists (A)**, **namespace tenants end-to-end across bridges (B)**, **add a
central route-compute brain with a local-cost fallback and per-class paths (C)**, **make transit stateless
via segment routing (D)**, **enforce per-tenant fairness at the contention points (E)**, and **jointly
optimize routing with capacity under per-tenant quotas (F)** — building deliberately *less* math and
fewer moving parts than the report until measurements justify more.
