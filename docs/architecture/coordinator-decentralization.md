# Coordinator Decentralization (Region-First, Multi-Global)

Cresco no longer requires a single, static global controller on the critical path. A region is a **first-class,
autonomous node** that operates and peers with *no global present*, and the "global" role has become an
**elected, sharded, optional coordinator service** that any capable node can host — with **more than one
coordinator** able to coexist. This page documents the shipped capability; the full design and the
distributed-systems rationale are in the
[Coordinator Decentralization plan](../design/coordinator-decentralization-plan.md).

!!! info "All flag-gated, default-off"
    Every behaviour here is opt-in; with the flags at their defaults the fabric behaves exactly as the prior
    single-static-global build. See [Configuration → Coordinator decentralization](../reference/configuration.md#coordinator-decentralization-region-first-multi-global).

## The idea: decompose the coordinator by duty

The old global sat on the critical path for seven duties. Most are **coordination-free** and were devolved to
the region tier; only two genuinely need agreement and are confined to a consensus group.

| Duty | Where it runs now | Consistency |
|---|---|---|
| Membership / directory | every node, from the pushed [RouteView](dynamic-routing.md) | eventual (CRDT/gossip) |
| Workload placement | region-local (single-writer); escalate cross-region | local; strong only for global invariants |
| Cross-domain liveness verdict | coordinator consensus group | **strong (quorum)** |
| Discovery / aggregation | scope-local, yield-over-harvest | eventual |
| Trust issuance | each region is its own CA; bilateral peer federation | none / eventual |
| Network-wide optimization | coordinator consensus group | **strong (quorum)** |
| Rendezvous | gossip / direct peering | eventual |

## Region-first autonomy

With `global_optional=true`, a region boots, starts its discovery engine and peer maintenance **immediately**,
and makes a *bounded* attempt to join a global instead of blocking forever. If no global is present it settles
into `REGION` as a valid steady state: it serves its agents, forms **direct region↔region bridges**
(`regional_peers` + self-organizing inference), and routes on the monotone latency metric — all with no global.

> Proven: two regions with no global log `operating REGION-FIRST`, never join a global, form a direct bridge,
> and exchange traffic with receiver-stamped `transit-hops=[]` (direct, no global hop).

## Multiple globals — coordinator registry, consensus, epoch, quorum

A coordinator is a **role, not a fixed tier**. Any `role=global` node advertises itself in its link-state, so
every node enumerates the live coordinator set straight from the shared `RouteView` — no scalar "the global"
pointer remains (`CoordinatorRegistry`).

- **Election** is a pure function of the shared view (deterministic — no vote needed): policy `identity`
  (lowest coordinator path) or `centroid` (k-center: the most central coordinator over the learned latency
  graph, so the leader sits where control-plane latency is lowest).
- **Consensus** (`CoordinatorConsensus`) adds what strong duties need: a monotonic **epoch** that bumps on
  every leadership change and **fences** stale (partitioned old-leader) messages, and a **majority quorum**
  over a *stable* membership (`coordinator_expected` / high-water) so a lone survivor cannot self-commit.
- Coordinators heartbeat over the data-plane push bus (no new connection); strong-duty commits (liveness
  verdict, global optimization) require f+1 of 2f+1 acks.

> Proven with three globals: they coexist and agree
> (`coordinators=[g1,g2,g3] leader=g1 epoch=1`); killing the leader elects the next with **epoch 1→2**;
> killing a second leaves a lone survivor at `hasQuorum=false` that refuses to commit (no split-brain); and on
> heal the set **reconverges to three**.

**How many coordinators?** Size each strong-duty shard to `2f+1` for a target fault tolerance `f` (3 tolerates
1, 5 tolerates 2), shard by duty+namespace, and place the members by k-center over the RouteView graph.

## Accurate failure detection (φ-accrual + SWIM)

Peer liveness uses a **φ-accrual** detector (continuous suspicion from heartbeat inter-arrival statistics)
rather than a fixed timeout. Before concluding a peer is gone, **SWIM** indirect probing asks other peers to
reach it; if any can, the fault is localized to the local link and the verdict is suppressed.

> Proven: cutting one direct link drove `phi=12.0`; an indirect probe via a third region reached the peer, so
> the suspicion was **suppressed** — no false "lost".

## Bilateral peer trust (no subordination)

With `security_peer_federation=true`, two **independently-rooted** regions cross-trust as **equals**: each
adds the other's region CA to its truststore and preserves its own identity, instead of one being re-enrolled
under a common issuer. This removes the need for a global as trust broker for region↔region mTLS.

> Proven: two independently-rooted regions, mTLS + client-auth on, **no global**, both log
> `cross-trusted peer … as an EQUAL … no subordination` and exchange secure direct pings.

## Behaviour under partition

Region-local operation and single-writer placement stay available (yield 100%); cross-domain **commits block**
without a quorum (consistency over availability, no split-brain); membership/directory reads serve reduced
harvest; on heal, the coordinator set and directory reconverge automatically.

## Configuration

`global_optional`, `global_connect_attempts`, `failure_phi_suspect`, `failure_phi_dead`, `failure_swim_k`,
`coordinator_expected`, `coordinator_election_policy`, `coordinator_lease_sec`, `security_peer_federation` —
see the [Configuration reference](../reference/configuration.md#coordinator-decentralization-region-first-multi-global).

## See also

- [Coordinator Decentralization plan](../design/coordinator-decentralization-plan.md) — full design, duty
  taxonomy, and the distributed-systems rationale (CALM/quorum/CPP/SPIFFE/Sobrinho)
- [Dynamic Cost-Aware Routing](dynamic-routing.md) — the push bus and per-node path computation this builds on
- [Messaging & Routing](messaging.md) · [Security & Identity](security.md)
