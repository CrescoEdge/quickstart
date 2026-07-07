# Cresco — Decentralizing the Coordinator Role — Implementation Plan

**Date:** 2026-07-06
**Status:** IMPLEMENTED + PROVEN (Phases A–F shipped, flag-gated, default-off). Dependency-ordered — the
fabric is never changed blind (same discipline as
[`distributed-identity-trust-design.md`](distributed-identity-trust-design.md) §5 and
[`optimal-global-routing-plan.md`](optimal-global-routing-plan.md)).

!!! success "Implemented & proven in containerlab — 2026-07-06"
    | Phase | What shipped | Proof (mesh) |
    |---|---|---|
    | **A** region-first boot | `global_optional`: regions come up + peer with NO global; bounded global-join | 2 regions, no global: `operating REGION-FIRST`, `Regional Global Success`=0, direct bridge, ping `transit-hops=[]` |
    | **A/W7** φ-accrual + SWIM | `PhiAccrualFailureDetector`; SWIM indirect probe before any verdict | triangle, cut R1↔R2: phi(R2)=12 → SWIM via R3 → `suspicion SUPPRESSED` (no false LOST) |
    | **C/W3** de-scalarize | `CoordinatorRegistry`: all role=global from RouteView, deterministic leader | 3 globals coexist: `coordinators=[g1,g2,g3] leader=g1 epoch=1` |
    | **D/W5** consensus + epoch | `CoordinatorConsensus`: beats, epoch fencing, majority quorum over stable membership | kill leader → `epoch 1→2`, new leader; kill 2nd → lone survivor `hasQuorum=false` (no split-brain) |
    | **E/W6** election/placement | identity + centroid (k-center over RouteView) policies | identity leader + failover proven; centroid coded |
    | **F/W8** partition/heal | quorum-guard blocks commits; failover-bridge auto-reconnect reconciles | partition g1 (3→2, quorum held) → heal → `reconverged to 3` |
    | **B/W2** bilateral trust | `security_peer_federation`: peers cross-trust as equals, no subordination | peer-CA exchange + preserved identity (security-on topology) |
    | **C/W4** regional scheduling | `rpipelinesubmit`: region-local placed regionally, cross-region escalates | Kandoo local/root decision + escalation to `coordinatorForDuty` |
    Flags: `global_optional`, `failure_phi_suspect/dead`, `failure_swim_k`, `coordinator_expected`,
    `coordinator_election_policy`, `coordinator_lease_sec`, `security_peer_federation`. All default to the
    prior single-static-global behaviour.
**Siblings:** [`optimal-global-routing-plan.md`](optimal-global-routing-plan.md) (routing — phases A/C/D
*shipped*), [`distributed-identity-trust-design.md`](distributed-identity-trust-design.md) (identity/trust —
regional CA *shipped*), [`region-federation-design.md`](region-federation-design.md),
[`link-metrics-design.md`](link-metrics-design.md), [`health-check-design.md`](health-check-design.md).
**Explicitly OUT OF SCOPE:** *a region whose control parent is another region* (region-under-region
nesting). On hold by direction; nothing in this plan requires it (see §9).

---

## 0. The one thing to read first

The global controller is a **single, statically-configured node** on the critical path for a specific,
enumerable set of duties. It has **no peer globals, no election, no state replication** — the global identity
is a scalar `globalRegion`/`globalAgent` pair (`core/ControllerStateImp.java:20-21`) and recovery on loss is
*reconnect to the same global*, never failover (`statemachine/ControllerSMHandler.java:126-143`).

But most of what *made* the global a bottleneck has already moved off it. After the routing work
(`netmetrics/RouteView`, `RouteAdvertiser`, `RouteComputer`, `RegionHealthWatcher.inferConnections`):
metric collection is per-node and **pushed**; every node builds its **own** mesh-wide latency graph and runs
its **own** Dijkstra; **direct region↔region TLS bridges already form and are preferred over via-global**
(`RegionHealthWatcher.maintainPeerConnections/measurePeerRtt`, `ActiveBroker.buildConnector`). The data plane
does not need the global at all.

So this is **not** "rebuild the control plane." It is: **decompose the coordinator by duty; devolve the
coordination-free duties to the regional (local-controller) tier; confine consensus to the two or three duties
that provably need it; and make the coordinator role an elected, sharded, *optional* service rather than a
fixed tier.** The theory (CALM, I-confluence, Ω/◇W, CPP, Gao–Rexford/Sobrinho, SPIFFE-style federation) tells
us *which* duties fall where; the code tells us *exactly where each duty lives today*.

---

## 1. Re-baseline — the seven duties, and where each lives today

From the controller source (verified 2026-07-06):

| Duty | Today (code) | On global critical path? |
|---|---|---|
| **D1 Authoritative membership / system-of-record** | fabric-wide registry assembled only on the global via the `update_mode=REGION` dataplane listener into the single Derby `gdb` (`core/ControllerStatePersistance.java:334-363,411-447`; `db/DBInterfaceImpl.java`) | **Yes** — only the global holds the whole-fabric view |
| **D2 Workload placement / scheduling (CADL)** | `AppScheduler`/`ResourceScheduler` created only in `startGlobalSchedulers()` (`ControllerSMHandler.java:910-931`); `gpipelinesubmit` only in `GlobalExecutor.java:1200-1214`; placement is operator-specified (`location_region`/`location_agent`) and merely *validated* against `gdb` | **Yes** — no regional deploy path exists |
| **D3 Cross-domain liveness authority** | only `GlobalHealthWatcher.GlobalNodeStatusWatchDog` ages a whole *region* to LOST (`globalcontroller/GlobalHealthWatcher.java:55-102`); regions age only their own agents (`RegionHealthWatcher.java:489-565`) | **Yes** — single-node verdict |
| **D4 Cross-domain discovery / aggregation** | `listregions`/cross-region `listagents`/`resourceinventory` fan out from the global reading `gdb` (`GlobalExecutor.java:278-372`) | **Yes** — but structurally region-answerable per scope |
| **D5 Trust anchoring / identity issuance** | per-region CA (`communication/RegionCA.java`), but the only mutual-trust path is **one-directional enrollment under a common issuer** — a non-global issuer **rewrites the joiner's tenant** to subordinate it (`communication/CertificateManager.java:488-517`, esp. `504-509`) | **Effectively yes** — the common issuer is the global |
| **D6 Network-wide optimization** | none beyond per-node Dijkstra; any true global TE/quota arbitration would need a single consistent view | n/a (unbuilt) |
| **D7 Rendezvous / forwarding target** | every `isGlobal()` message forwarded to the one known global (`RegionHealthWatcher.sendRegionalMsg:351-360`, `RegionalExecutor.remoteGlobalSend:162-176`); a region will not start its **inbound discovery engine** until it reaches `REGION_GLOBAL` (`ControllerSMHandler.java:461,496`) | **Yes** — hard rendezvous + boot gate |

**Already decentralized (leverage, do not rebuild):** per-node link metrics (pushed), the mesh-wide
`RouteView` graph, per-node Dijkstra + source-route enforcement, direct region↔region bridges + self-organizing
inference, regional-CA trust that keeps material O(regions). The `RouteView` push bus is the substrate several
workstreams below extend.

---

## 2. Target model — region-first, coordinator-as-elected-sharded-service

Two structural inversions:

1. **Region-first autonomy.** A region is a first-class, fully-operational node with **zero** coordinators
   present: it serves its agents, forms peer bridges, routes on the monotone latency metric, and is
   *discoverable*. Coordinators provide **optional** fabric-wide services, not permission to exist.

2. **Coordinator is a role, not a tier.** "Global" stops being a boot flag on one node and becomes a
   **service role** — hosted by an *elected, sharded set* of coordinator-capable nodes — that provides exactly
   D1/D3/D4/D6 at fabric scope. This is the Kandoo *local-controller / root-controller* split: local
   (regional) controllers absorb everything that depends only on local or single-writer state; the root
   (coordinator) set handles only the rare, genuinely-global operations.

The coordinator set is **sharded by duty + namespace** and **sized for fault tolerance** (§7): each shard that
carries a *strong* duty (D3/D6) is a `2f+1` consensus group; the *coordination-free* duties (D1/D4, and D2 in
the common case) run replicated/eventual and need no quorum.

---

## 3. Corrected duty-decomposition taxonomy (the consistency contract)

This is the theory applied *precisely* — with the corrections from review folded in (single-writer capacity,
convergence ≠ verdict, isotonicity for optimal routing).

| Duty | Consistency class | Coordination? | Mechanism | Sharp caveat |
|---|---|---|---|---|
| **D1** membership registry | eventual / causal | **free** | CRDT directory (OR-Set of nodes) gossiped over the `RouteView`/LSA bus | CRDT gives **replica convergence, not a liveness verdict** — "is X permanently dead" is D3, not D1 |
| **D2** placement | single-writer-local; strong only for global invariants | **free in common case** | region owns its own capacity counter (single writer) → cross-region placements are partitioned & confluent; **escalate to D6 quorum only when a CADL carries a genuinely global invariant** (e.g. "exactly N instances fabric-wide") | a per-domain *capacity limit* is the canonical **non**-I-confluent case — it is confluent **only** because one local controller owns that counter, not because limits are inherently safe |
| **D3** liveness verdict | strong (linearizable) | **required** | `2f+1` consensus commit of a "region LOST" verdict, **fed by** φ-accrual + SWIM indirect-probe evidence | φ-accrual/SWIM decide *when to suspect*; the *commit* still needs quorum + a fencing epoch |
| **D4** discovery/aggregation | eventual | **free** | scope-local answers; fan-out prioritizes **yield over harvest** | a partition returns *available* data at reduced completeness — never blocks |
| **D5** trust issuance/revocation | issuance: none; revocation: eventual | **free** | each region a sovereign root; **bilateral trust-bundle exchange** (SPIFFE-style), short-TTL SVIDs so revocation ≈ TTL, not a CRL race | must **stop the subordinating identity-rewrite** — peers cross-trust as equals, keep own root |
| **D6** global optimization (TE/quota) | strong (total order) | **required** | `2f+1` consensus over a consistent NIB view; epoch-fenced config push | irreducible: two domains optimizing the same bottleneck on local heuristics can drive congestion collapse |
| **D7** rendezvous | eventual | **free** | gossip/anycast bootstrap + monotone-metric routing takes over | bootstrap seed is unavoidable (static seed or anycast) — but it is a *hint*, not a critical-path authority |

**Routing convergence (underpins region-first, D7):** Cresco routes on **additive, positive RTT**, which is
both *monotone* (path weight never decreases as it extends → convergence, no dispute wheels — Sobrinho) **and
isotone** (→ the converged paths are actually *optimal*, not merely stable). This is the property that lets
regions peer and converge on optimal loop-free routes with **no path oracle**. Gao–Rexford policy hierarchy is
*not* needed unless/until Cresco adds local *policy* overrides to the metric.

---

## 4. Deliberate deviations & decisions (each for a concrete reason)

- **C1 — Devolve first, consensus last.** Ship region-first autonomy and duty-devolution (coordination-free
  work, no new consensus machinery) before building the `2f+1` group. Most of the "unnatural constraint"
  dissolves without any consensus at all; consensus is added only for D3/D6.

- **C2 — Reuse the `RouteView` push bus as the gossip/CRDT transport.** The mesh-wide, NON_PERSISTENT,
  selector-filtered dataplane push I already built for link-state is the natural carrier for the D1 membership
  CRDT and coordinator-candidacy advertisements. Do **not** invent a second gossip layer.

- **C3 — Elect on the graph we already have.** Coordinator *placement* is a k-center facility-location problem
  over the network graph; Cresco already assembles that graph live in `RouteView`. Centrality/placement is
  computed from `RouteView` with the existing `RouteComputer` machinery — no new telemetry.

- **C4 — Static seed + dynamic re-placement.** Seed coordinator-capable nodes statically (k-center approx over
  the boot topology) for a bounded worst-case bootstrap, then promote/demote dynamically as the graph and load
  change. Never fully unstructured cold-start (self-stabilization convergence is real but slow).

- **C5 — Peers cross-trust, they do not subordinate.** The `2f+1` and region-region peering both require the
  bilateral-trust fix (W2) — a peer must keep its own root and identity. This is the keystone security change.

- **C6 — Preserve QoS and "Cresco is the router" invariants.** Every relay/forward stays inside the
  mTLS+secret-gated fabric, TTL-bounded; QoS tiers (`MsgQoS`) and the isolated `ControlPlaneSender` are
  untouched; coordinators never become an ActiveMQ-routed shortcut. (Non-negotiables carried from the routing
  work.)

- **C7 — Fence everything global.** Every mutation of fabric-wide state (D1 authoritative writes, D3 verdicts,
  D6 config) carries a monotonically-increasing **epoch/lease token**; `MsgRouter`/executors **reject stale
  epochs**. This is the split-brain guard and must land *with* the first multi-coordinator code, not after.

---

## 5. Workstreams (the concrete build) — goal · integration point · depends · validate

- **W1 — Region-first boot autonomy.**
  *Goal:* a region with no reachable coordinator comes up fully and is discoverable.
  *Integration:* `ControllerSMHandler.stateInit()` cases 8/24 — move `startNetDiscoveryEngine()` to run right
  after `regionInit()` instead of after the `REGION_GLOBAL` while-loop (`:461,496`); introduce
  `global_optional`/`region_first` so "no parent global" is a **steady state**, not `REGION_GLOBAL_FAILED`;
  make `ParentLinkHealthCheck`/`globalControllerLost` (`:126-143`) a no-op when `global_optional`.
  *Depends:* nothing.
  *Validate:* two isolated regions (no global) discover each other, bridge, route, exchange RPC; suite stays
  green with `global_optional=false`.
  *Flag:* `global_optional` (default false).

- **W2 — Bilateral region↔region trust (peer cross-trust).**
  *Goal:* two independently-rooted regions mutually authenticate with **no common issuer**.
  *Integration:* extend the CERTIFY handshake (`netdiscovery/DiscoveryProcessor.java:199-236`) to **exchange
  `RegionCA.caChain()` both directions** and add each peer's CA to the local truststore
  (`CertificateManager`/`ActiveBroker.updateTrustManager`); **stop the non-global tenant rewrite**
  (`CertificateManager.java:504-509`) on a *peer* enrollment so each region keeps its own root+identity;
  short-TTL leaf SVIDs so revocation ≈ TTL.
  *Depends:* nothing (regional CA shipped).
  *Validate:* region A↔B bridge with `broker_require_client_auth=true`, neither enrolled under a common global;
  `CrescoAuthorizationBroker` derives the correct peer identity; cross-tenant still denied.
  *Flag:* `security_peer_federation` (default false). *(This is the Phase 3 "NEXT" item in the trust design.)*

- **W3 — Coordinator as a service role (de-scalarize the global).**
  *Goal:* the code can address *a set of* coordinators and a per-shard leader, not one static node.
  *Integration:* replace scalar `globalRegion`/`globalAgent` (`ControllerStateImp.java:20-21`,
  `getGlobalControllerPath:100-106`) with a coordinator-set + per-duty leader pointer; generalize
  `RegionHealthWatcher.sendRegionalMsg` (`:351-360`) and `RegionalExecutor.remoteGlobalSend` (`:162-176`) to
  resolve "the coordinator for duty X / namespace Y"; implement the real global-vs-regional split in
  `MsgRouter.getRoutePath()` (the hard-coded `GM="0"`, `:764-773`, is the documented rework).
  *Depends:* W1.
  *Validate:* messages for a duty route to that duty's current coordinator; a second coordinator can coexist
  without either being "the" global.
  *Flag:* `coordinator_roles` (default off → legacy single-global behavior).

- **W4 — Duty devolution to the regional (local-controller) tier.**
  *Goal:* coordination-free duties run regionally; the coordinator sees only what's genuinely fabric-wide.
  *Integration:*
    - **D1:** a mesh-wide membership **OR-Set CRDT** gossiped on the `RouteView`/LSA bus (C2); each region is
      the single writer for its own subtree; any node can answer a directory read locally.
    - **D2:** add a `RegionalExecutor` `rpipelinesubmit` + a regional scheduler mirroring
      `AppScheduler`/`ResourceScheduler` that validates against the *regional* view and dispatches `pluginadd`
      in-region; detect cross-region `location_region` at parse time (`AppScheduler.buildNodeMaps:262`) and
      **escalate to a coordinator only then**.
    - **D4:** scope-local `listagents`/`resourceinventory`; fan-out yields over harvests.
  *Depends:* W3 (needs the coordinator abstraction to escalate to).
  *Validate:* a region-local CADL deploys with **no coordinator present**; a cross-region CADL escalates; the
  directory converges across a partition heal.
  *Flag:* `regional_scheduling`, `regional_registry` (default off).

- **W5 — Elected, sharded coordinator group for D3/D6 (the consensus lift).**
  *Goal:* the two strong duties get a correct, fault-tolerant home.
  *Integration:* a `2f+1` consensus group among coordinator-capable nodes (embed a minimal Raft or a vetted
  lib); **D3** = quorum-committed "region LOST" verdict replacing `GlobalHealthWatcher`'s single-node authority
  (`:55-102`), fed by W7; **D6** = consensus over a shared NIB for TE/quota; **epoch/lease fencing (C7)** on
  every fabric-wide mutation, enforced in `MsgRouter`/executors.
  *Depends:* W3, W7.
  *Validate:* kill the leader → a new leader commits within lease+election bound; a partitioned old leader's
  stale-epoch writes are rejected; no double-verdict on a region.
  *Flag:* `coordinator_consensus` (default off).

- **W6 — Dynamic election + placement (CPP over `RouteView`).**
  *Goal:* coordinators sit where they minimize control-plane latency and survive failures — chosen by the
  network, not by hand.
  *Integration:* compute k-center/k-median placement over the live `RouteView` graph (reuse `RouteComputer`);
  static seed at boot (C4) then promote/demote coordinator-capability as the graph/load shift; leader election
  via Ω/◇W realized by the φ-accrual+SWIM layer from W7.
  *Depends:* W5.
  *Validate:* on topology change, the coordinator set migrates toward the new centroid; worst-case region→coordinator
  latency stays within the k-center bound; churn is damped (hysteresis, as in routing).
  *Flag:* `coordinator_dynamic_placement` (default off).

- **W7 — Failure detection upgrade (φ-accrual + SWIM).**
  *Goal:* accurate suspicion under a latency-heterogeneous, partition-prone mesh; no spurious verdicts.
  *Integration:* replace fixed-timeout aging (`RegionHealthWatcher.RegionalNodeStatusWatchDog:489-565`,
  `GlobalHealthWatcher`, `ParentLinkHealthCheck`) with a **φ-accrual** continuous suspicion level + **SWIM**
  indirect probing (ask k random peers before escalating), so an asymmetric A↔B partition does not become a
  global "B is dead."
  *Depends:* W1 (region-first health semantics).
  *Validate:* asymmetric partition → suspicion localized, no false LOST; recovery time vs. fixed-timeout
  baseline; feeds a correct D3 commit in W5.
  *Flag:* `failure_detector_accrual` (default off → legacy timeout).

- **W8 — Partition behavior + reconciliation (hardening).**
  *Goal:* define and prove the harvest/yield contract end-to-end.
  *Integration:* local + single-writer-local placement yield 100% under partition; cross-domain **modifications
  (D6) block on quorum**; D1/D4 reads serve reduced-harvest; on heal, CRDT merge (D1) converges with no manual
  step; coordinator group re-forms under `2f+1`.
  *Depends:* W4, W5.
  *Validate:* partition the coordinator set → regions keep operating (region-first), global mutations refuse,
  heal converges automatically; extend the containerlab suite with a partition/heal scenario.
  *Flag:* n/a (behavior of the above flags under partition).

---

## 6. Phased delivery (dependency-ordered, not a schedule)

| Phase | Theme | Workstreams | Exit gate |
|---|---|---|---|
| **A — Autonomy & signal** | region-first boot + good failure detection (no consensus yet) | W1, W7 | Two globalless regions form, route, and RPC; asymmetric partition yields no false LOST. **High value, low risk — dissolves most of the "unnatural constraint."** |
| **B — Peer trust** | bilateral region↔region cross-trust | W2 | Two independently-rooted regions mutually authenticate with mTLS, no common global; cross-tenant still denied |
| **C — Role & devolution** | de-scalarize the coordinator; regional registry + regional scheduling | W3, W4 | Region-local CADL deploys with no coordinator; directory is a converging CRDT; cross-region work escalates cleanly |
| **D — Consensus core** | `2f+1` group for D3/D6 + epoch fencing | W5 (+C7) | Leader failover within bound; stale-epoch writes rejected; single authoritative region-LOST verdict |
| **E — Self-placement** | dynamic election + CPP placement over `RouteView` | W6 | Coordinator set migrates to the latency centroid on topology change; damped churn |
| **F — Partition hardening** | harvest/yield contract + reconciliation; TE on top | W8 (+ D6 use-cases) | Coordinator-set partition → region-first survives, global mutations refuse, heal auto-converges |

Phases **A–B** are the near-term high-value work and require **no new consensus machinery**. **C** is the
structural refactor. **D–F** are the genuinely-new distributed-systems engineering, each independently
shippable behind its flag.

---

## 7. How many coordinators, and where — the sizing rule

- **Coordination-free duties (D1, D4, and D2 common case):** replicate freely — *every* coordinator-capable
  node can serve them; there is no quorum and no "how many" constraint beyond desired read availability.
- **Strong duties (D3, D6):** each duty+namespace shard is a **`2f+1` consensus group** to tolerate `f`
  simultaneous coordinator failures and requires a **reachable majority** to commit (the Ω/consensus
  requirement `f < n/2`). This is the principled answer to "how many globals": choose the fault-tolerance
  target `f`, size each strong shard to `2f+1` (3 tolerates 1, 5 tolerates 2), and **shard by duty+namespace**
  so a leader failure's blast radius is one shard, not the fabric.
- **Placement:** the `2f+1` members of each shard are placed by **k-center** (bound worst-case region→coordinator
  latency, for failure-detection timeouts and SLAs) or **k-median** (minimize average) over the live
  `RouteView` graph — statically seeded, dynamically re-placed (W6).
- **Region-first floor:** a region with **zero** reachable coordinators still operates locally and peers
  directly; it simply cannot participate in D3/D6 commits until a majority is reachable again.

---

## 8. Risks & mitigations

- **Split-brain on partition of the coordinator set** → epoch/lease fencing (C7) landed *with* W5, not after;
  data plane rejects stale-epoch writes; majority-quorum required to commit.
- **False global verdicts under asymmetric partition** → φ-accrual + SWIM indirect probing (W7) before any D3
  commit; SWIM localizes A↔B failures.
- **New consensus in the hot path** → consensus is confined to D3/D6 (rare, off the data path); everything on
  the common path stays coordination-free (C1). Benchmark-gate against the routing suite.
- **Trust regression when peers cross-trust** → `security_peer_federation` flag-gated, staged; `vm://` and
  existing regional-CA paths untouched; extend `tenant_isolation_mesh_test.sh` to peer-federated bridges.
- **Election/placement churn** → static seed + hysteresis on promote/demote (reuse the routing anti-flap
  discipline); damping validated in W6.
- **Doing too much** → build the CRDT/eventual mechanisms for D1/D4/D2, not consensus, wherever the taxonomy
  (§3) permits; add strong coordination only for D3/D6.

---

## 9. Out of scope (explicitly)

- **Region-under-region control nesting** (a region whose control parent is another region) — on hold by
  direction. Nothing here needs it: region↔region **peering** (data + relay + trust) is a *lateral* peer
  relationship, not a control hierarchy; the coordinator set is a *role over* regions, not a region *under* a
  region. If nesting is later desired, it is an independent design.
- **Payload envelope encryption, per-tenant H-DWRR fairness, joint routing+capacity** — tracked in
  [`optimal-global-routing-plan.md`](optimal-global-routing-plan.md) phases B/E/F; orthogonal to coordinator
  decentralization.

---

## 10. Validation & success metrics

**Reuse:** the containerlab mesh + independent-proof method (receiver-side hop-stamps + physics + causal
netem intervention) already used for routing; `tenant_isolation_mesh_test.sh`, `regional_ca_mtls_test.sh`,
the bring-up/recovery suite (green with each flag **off** and, separately, **on**).
**New:** globalless two-region bring-up (W1); peer-federated mTLS bridge (W2); region-local CADL deploy +
cross-region escalation (W4); leader-failover + stale-epoch-rejection (W5); asymmetric-partition no-false-LOST
(W7); coordinator-set partition → region-first survival + auto-reconcile (W8); coordinator migration to
centroid on topology change (W6).
**Metrics:** control-plane availability during coordinator loss (target: local duties unaffected); D3 verdict
correctness (zero false LOST under asymmetric partition); leader-failover time vs. lease bound; worst-case
region→coordinator latency vs. k-center bound; directory convergence time after heal.

---

## 11. One-line summary

The global is a static single point on the critical path for seven duties; the data plane already needs none
of them. **Decompose by duty; devolve the coordination-free ones (D1/D2/D4/D5/D7) to autonomous region-first
local controllers over the push bus already built; confine `2f+1` consensus to the two duties that provably
need total order (D3 liveness verdict, D6 global optimization); make the coordinator an elected, duty-sharded,
CPP-placed, epoch-fenced *service role* rather than a fixed tier** — building deliberately *less* consensus
than a fully-general solution, exactly where the theory says none is required.
