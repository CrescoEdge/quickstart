# Cresco — Tenant Isolation (Glocal) — Design

**Date:** 2026-07-03
**Status:** **L1+L2 SHIPPED + PROVEN (2026-07-03), flag `tenant_namespacing` (default off).** Both controlled
channels are namespaced end-to-end and role-based authorization is live. Proven by
`run/tests/tenant_namespacing_test.sh` **12/12** (federation forms + zero self-denial under namespacing →
MsgEvent wildcard-infra-inbox + producer tenant-qualification work; TENANT confined to `T.<tenant>.*`;
SUPERUSER cross-tenant god view) with **no regression** (`tenant_isolation_mesh_test` 10/10,
`security_foundation` 15/15, `regional_ca_mtls` 10/10). L3 (dedicated per-tenant bridges) + L4 (payload
encryption) remain optional hardening tiers; core isolation already holds via client-edge ACL at every broker.

**What shipped:**
- `library/security/TenantNamespace` — pure `T.<tenant>.<raw>` qualify/wildcard/parse helper.
- `TenantPolicy` — role-aware (`SUPERUSER`/`INTERNAL`/`TENANT`) + namespaced-prefix rule (own `T.<tenant>.`
  subtree only; closes the flat-name same-region write hole).
- `CrescoAuthorizationBroker` — `roleOf()` from `broker_superuser_tenants` (default `cresco-system`).
- `AgentProducer` — stamps origin tenant (`cresco_tenant` param) + qualifies the forward destination.
- `ControllerSMHandler.initIOChannels` — infra (vm://) consumes the `T.*.` **wildcard** inbox; leaf agents
  consume their own `T.<tenant>.` inbox.
- `DataPlaneServiceImpl.getTopicName` — dataplane topics qualified `T.<tenant>.<agent|region|global>.event`.

This doc recommends the target model and the minimal grounded changes.
**Siblings:** [`distributed-identity-trust-design.md`](distributed-identity-trust-design.md) (owns
identity/trust/authz — this doc is its §5 "tenant namespacing" expanded and revised),
[`optimal-global-routing-plan.md`](optimal-global-routing-plan.md) (W2 = this doc's L1+L3; W6 = L5; W9 = L4),
[`metrics-unification-design.md`](metrics-unification-design.md).

---

## 0. Framing: "glocal" = one global invariant, enforced locally at every hop

The design goal in one sentence: **define a single global isolation invariant, and make it enforceable
from purely *local* information at every broker — including every bridge — so no broker needs global
coordination to uphold it.** That is what "glocal" means here: think globally (one invariant over the
whole federated mesh), act locally (each broker enforces it on the traffic it touches, using only what it
can see).

The mechanism that makes local enforcement of a global invariant possible is **carrying the tenant in two
places at once**: (1) *cryptographically*, in the connection identity (already done — `O=tenant` in the
cert DN, mTLS-bound), and (2) *syntactically*, in the destination name (a mandatory tenant prefix — not
done). With both, any broker can check "does this connection's tenant match this destination's tenant?"
with a local string comparison — no topology map, no controller round-trip. **Namespacing is not cosmetic;
it is the thing that turns a global rule into a local check.**

---

## 1. What's in place vs. the gap (code-grounded)

**Shipped and proven (keep):**
- Tenant bound into the cert DN; mTLS derives the authoritative principal
  ([CrescoAuthorizationBroker.java:141-167](../code/controller/src/main/java/io/cresco/agent/controller/communication/CrescoAuthorizationBroker.java)).
- Regional-CA trust distribution (O(regions), not O(n²)).
- `TenantPolicy` ACL evaluated at addConsumer/addProducer/send.

**The three real gaps — why isolation is "not implemented" for the mesh:**

1. **Destinations are not tenant-namespaced, so the strong boundary is dormant.** The primary rule —
   allow `destination == tenant || destination.startsWith(tenant + ".")`
   ([TenantPolicy.java:70-72](../code/controller/src/main/java/io/cresco/agent/controller/communication/TenantPolicy.java#L70)) —
   only matches destinations that *happen* to be tenant-prefixed. Nothing **mandates** that, and the
   fabric's own destinations (`region_agent` inboxes, `global.event`) are not prefixed. The boundary
   exists in code but has almost nothing to match.

2. **A concrete cross-tenant *write* hole in the flat scheme.** Because inboxes are `region_agent` with no
   tenant, the "same-region-write" rule
   ([TenantPolicy.java:82-85](../code/controller/src/main/java/io/cresco/agent/controller/communication/TenantPolicy.java#L82))
   lets **tenant B send into tenant A's inbox whenever they share a region** (`R_agentA` starts with
   `R_`, so WRITE is allowed). Confidentiality of A's inbox holds (B can't read), but *delivery integrity*
   does not — B can inject. Namespacing closes this: the peer rule becomes intra-tenant.

3. **The shared dataplane topics are a cross-tenant channel, and bridges bypass enforcement entirely.**
   `agent/region/global.event` are allow-listed for *all* tenants
   ([TenantPolicy.java:62-63](../code/controller/src/main/java/io/cresco/agent/controller/communication/TenantPolicy.java#L62),
   `DataPlaneServiceImpl` flat topic names). The comment's justification ("isolated by physical per-region
   broker separation") **fails on a global broker**, where multiple regions/tenants converge. And every
   broker-to-broker bridge is *blanket-exempt* —
   `ctx.isNetworkConnection() → return true`
   ([CrescoAuthorizationBroker.java:102](../code/controller/src/main/java/io/cresco/agent/controller/communication/CrescoAuthorizationBroker.java#L102)) —
   so inter-broker hops carry **all tenants with zero tenant enforcement.**

**Conclusion:** enforcement today exists only at the *client edge of a single broker*. It is neither
end-to-end nor cross-bridge. That is exactly the piece to build.

---

## 2. Isolation has three axes — name them, or you'll only build one

| Axis | "A tenant cannot …" | Covered by |
|---|---|---|
| **Confidentiality** | …read another tenant's data | L1 namespace + L2 ACL (read-deny) + L4 encryption |
| **Delivery integrity** | …inject into / misdeliver to another tenant | L1 namespace + L2 ACL (write-deny) + **L3 bridge scoping** |
| **Performance** | …starve/degrade another tenant (noisy neighbor) | **L5 per-tenant QoS (H-DWRR)** |

An ACL-only design covers confidentiality and part of delivery integrity, *on one broker*. Real isolation
across the mesh needs all three axes, at every hop.

---

## 3. The layered model (each layer independently switchable)

### L0 — Identity (done). Tenant cryptographically bound in the cert DN; the anchor everything keys off.

### L1 — Namespace (the keystone). Make the tenant a **mandatory** destination prefix, everywhere.
Every destination — inboxes **and** dataplane topics — carries its tenant:
```
Inbox / queue:   t.<tenant>.<class>.<region>.<agent>
Dataplane topic: t.<tenant>.<class>.event        (per-tenant, replacing the shared global.event)
```
`<class>` ∈ {live, ctrl, tele, bulk} — the `MsgQoS` tier, placed *before* region so a bridge or scheduler
can filter by tenant **and** class with one prefix. Use `.` as the hierarchy separator so a whole tenant
is one wildcard subtree: `t.<tenant>.>`. This single change:
- makes L2/L3 a local prefix rule (the whole point of "glocal");
- removes the shared-topic cross-tenant channel (gap #3) — `global.event` becomes N per-tenant topics;
- closes the same-region write hole (gap #2) — inboxes are now tenant-scoped;
- gives L5 (per-tenant QoS) and the router (per-tenant/per-class paths) a ready-made key.

### L2 — Local ACL (mostly done; simplify around the prefix).
`TenantPolicy` collapses to: *a connection authenticated as tenant T may only touch `t.<T>.…`* (plus
advisories, handled in L1/§5). The flat-name special cases (`region_agent` own-inbox, same-region peer
read/write asymmetry) become **intra-tenant** sub-rules under the prefix, and the cross-tenant write hole
disappears. Net: fewer rules, stronger guarantee.

### L3 — Bridge-scoped tenant forwarding (THE fix for gap #3).
Replace the blanket `isNetworkConnection → exempt` with a **per-bridge allowed-tenant set**. Two levers,
both already available in Cresco:
- **Forwarding filter (primary):** each bridge connector forwards *only* its allowed tenant namespaces via
  `setDynamicallyIncludedDestinations(t.<tenant>.>)` / `setExcludedDestinations(...)` — the *exact*
  mechanism `buildConnector` already uses for dataplane sharding
  ([ActiveBroker.java:572-586](../code/controller/src/main/java/io/cresco/agent/controller/communication/ActiveBroker.java#L572)).
  A message for a non-allowed tenant prefix is never bridged.
- **Receiving-side enforcement (defense in depth):** the bridge connection *does* have an identity (the
  peer broker's cert, chaining to a region CA). Give bridges a `BridgeTenantPolicy` (allowed-tenant set for
  that edge) instead of a free pass, so even if a mislabeled message crosses, the receiving broker's L2
  prefix check drops it.

**Shared prefix-filtered bridges (O(E)) are the default**; **dedicated per-tenant bridges (O(T·E)) are an
opt-in high-assurance tier** — mechanically just multiple connectors per host, which `bridgeGroups` /
`addBridgeConnections` already support.

### L4 — End-to-end confidentiality across bridges (opt-in).
Transport mTLS encrypts each hop *in flight*; the residual threat is **payload at rest inside a
compromised/inspected transit broker**. For tenants that can't accept that: envelope-encrypt the payload
with a tenant key (AES-256-GCM), keep routing metadata (namespaced dst + segment path) clear in JMS
headers, decrypt at egress. Reuse the shipped `MessageSigner` pattern / recipient leaf public key. Opt-in,
because it's costly and most tenants are satisfied by L1–L3 + hop TLS.

### L5 — Per-tenant performance isolation (the "QoS everywhere on the tenant level" requirement).
Hierarchical Deficit-Weighted Round Robin keyed by the L1 tenant prefix (L1 inter-tenant fair share, L2
intra-tenant `MsgQoS` priority), enforced at the contention points (ingress rate-limit + shared-bridge
egress) by extending the existing `CrescoAuthorizationBroker` filter chain. This is routing-plan W6; it is
**part of the isolation model, not an add-on** — without it, one tenant's bulk flood violates isolation
axis 3.

---

## 4. The glocal invariant (state it, enforce it, test it)

> **Global invariant:** a message stamped tenant *T* exists *only* on destinations prefixed `t.<T>.`, and
> is accessible *only* to connections cryptographically authenticated as *T*.
>
> **Local enforcement (every broker, every op — produce, consume, send, *and bridge-forward*):** compare
> the connection's cert-bound tenant to the destination prefix; allow iff they match (plus explicit
> advisory/shared carve-outs). No global state consulted.

Because tenant is both cert-bound and name-carried, the check is purely local, so it **composes**: if every
hop upholds it, the mesh upholds it. Defense in depth: a message that somehow reaches the wrong broker is
dropped by that broker's L2 check even if a bridge misforwarded it.

---

## 5. Concrete, minimal changes (grounded)

1. **Mandate the prefix at ingress.** `MsgRouter` ingress stamps/validates `t.<tenant>.<class>.…` from the
   connection's bound identity; a publish whose destination tenant ≠ authenticated tenant is dropped at
   ingress (not just denied at the broker).
2. **Per-tenant dataplane topics.** `DataPlaneServiceImpl` topic names become `t.<tenant>.<class>.event`;
   remove `global.event` (and the sharded `global.event.N`) from `broker_shared_destinations` — the shared
   carve-out ([TenantPolicy.java:62](../code/controller/src/main/java/io/cresco/agent/controller/communication/TenantPolicy.java#L62))
   should shrink to genuinely tenant-agnostic control only.
3. **Bridge policy.** New `BridgeTenantPolicy` + per-connector `setDynamicallyIncludedDestinations`; replace
   the `isNetworkConnection` blanket exemption
   ([CrescoAuthorizationBroker.java:99-111](../code/controller/src/main/java/io/cresco/agent/controller/communication/CrescoAuthorizationBroker.java#L99))
   with the allowed-tenant-set check.
4. **Advisory leak (subtle).** `ActiveMQ.Advisory.*` is allow-listed
   ([TenantPolicy.java:59-61](../code/controller/src/main/java/io/cresco/agent/controller/communication/TenantPolicy.java#L59));
   advisories can leak *destination existence* across tenants on a shared broker. Decide: scope advisories
   per tenant subtree, suppress cross-tenant advisory propagation on bridges, or accept + document.

---

## 6. Cross-global — the "global" half of glocal

A global controller must only accept/forward tenant namespaces it is *federated for*. Reuse the trust-bundle
gossip already designed for CA distribution
([distributed-identity-trust-design.md](distributed-identity-trust-design.md) §3.2): alongside the region-CA
bundle, a global advertises an **allowed-tenant set per federation edge**. A global↔global bridge then
applies L3 with that set, so a tenant present under G1 is only reachable under G2 if the G1↔G2 edge is
authorized for it. Tenant reachability becomes a policy artifact distributed the same way trust is — no new
distribution channel.

---

## 7. Migration (additive, flag-gated, staged — same discipline as the security fabric)

- **Flag `tenant_namespacing`**, default off → fabric unchanged.
- **Dual-namespace transition shim:** during rollout, brokers accept both flat and namespaced destinations;
  a translation layer maps legacy `region_agent` → `t.<tenant>.<region>.<agent>` using the connection's
  bound tenant, so un-migrated clients keep working. Flip enforcement on per region once both ends speak
  namespaced.
- **Default tenant:** existing single-tenant deployments map to `t.default.…` — nothing breaks.
- **`vm://` stays exempt** for local rights, but the controller must now stamp the correct tenant on
  outbound so its own messages land in the right namespace.
- Stage region-by-region; keep the full bring-up/recovery + `tenant_isolation_mesh_test.sh` +
  `regional_ca_mtls_test.sh` suites green with the flag both off and on.

---

## 8. Decisions to make (recommendations first)

1. **Namespace scheme.** *Recommend* `t.<tenant>.<class>.<region>.<agent>`, `.`-hierarchical, class before
   region. Confirm delimiter and that IDs never contain it.
2. **Tenant ID form.** *Recommend an opaque stable ID* (UUID/hash), not the human `O=tenant` string, so
   tenant names don't leak in destination strings visible to shared-broker operators; controllers hold the
   map. (Trade: harder debugging.)
3. **Bridge model.** *Recommend* shared prefix-filtered (O(E)) default + dedicated per-tenant opt-in.
4. **Advisory handling.** *Recommend* scope/suppress cross-tenant advisories (close the existence leak).
5. **Encryption tier + key mgmt.** *Recommend* opt-in; per-tenant key via recipient leaf pubkey to avoid a
   new KMS dependency initially.
6. **Cross-global tenant policy.** Allowed-tenant-per-edge in the trust bundle (§6).

---

## 9. Relationship to the routing plan

L1 (namespace) + L3 (bridge scoping) **are** routing-plan W2 and are its prerequisite — the router forwards
per-tenant prefixes on chosen bridges, so isolation and routing share the same substrate. L5 = W6, L4 = W9.
**Build order:** L1 → L2 → L3 first (they close the actual leaks and unblock W2 routing), L5 alongside the
QoS/fairness work, L4 as an opt-in tier. Do L1–L3 before any redundant-path routing carries multi-tenant
traffic, or the new paths widen the very leak we're closing.

---

## 10. Configuration & operations reference (shipped)

All tenant-isolation behavior is additive and flag-gated; the defaults reproduce the pre-existing fabric
exactly. There are two independent switches — the **security fabric** (identity/mTLS/authz, shipped
earlier) and **namespacing** (this work). Namespacing is only meaningful with the security fabric on.

| Config (`-D` or config file) | Default | Effect |
|---|---|---|
| `tenant_namespacing` | `false` | Master switch for this work. When on: MsgEvent inbox queues and dataplane topics are qualified `T.<tenant>.<raw>`; producers stamp+qualify by message tenant; infra (vm://) consumes the `T.*.` wildcard inbox. Off ⇒ raw names, byte-for-byte unchanged. |
| `tenant_id` | `default` | This node's tenant. Bound into the leaf cert DN (`O=<tenant>`) and used as the local tenant for stamping/qualification. In a single-tenant deployment leave as `default`. |
| `broker_security_enabled` | `false` | Installs `CrescoAuthorizationBroker` (the ACL). Required for isolation to be *enforced*; namespacing without it only *names* destinations, it does not deny. |
| `broker_require_client_auth` | `false` | Broker demands a client cert (mTLS) and binds the principal from the validated DN — makes tenant identity non-spoofable. Strongly recommended whenever `broker_security_enabled` is on. |
| `broker_superuser_tenants` | `cresco-system` | Comma list of tenants whose **network** clients get the cross-tenant `SUPERUSER` role (god view). The local controller (`vm://`) and broker bridges are superuser-equivalent regardless. |
| `broker_shared_destinations` | `agent.event,region.event,global.event` | Tenant-agnostic destinations allowed for everyone. Under namespacing the real dataplane topics are `T.<tenant>.*.event`, so these flat names are unused; kept for back-compat. |
| `security_regional_ca` | `false` | Regional-CA enrollment (O(regions) trust). Recommended at scale so every node presents a CA-signed, identity-bearing leaf. |
| `broker_security_log_allow` | `false` | Verbose: log every ALLOW decision (debugging ACLs). DENY is always logged. |

**Recommended enablement order (staged, per region, never blind):**
1. Turn on `security_regional_ca` fabric-wide; confirm every node enrolls and bridges under mTLS
   (`regional_ca_mtls_test`).
2. Turn on `broker_security_enabled` + `broker_require_client_auth`; confirm zero self-denial and that
   cross-tenant is denied (`tenant_isolation_mesh_test`).
3. Turn on `tenant_namespacing`; confirm the fabric still forms and control plane flows with zero
   self-denial (`tenant_namespacing_test`). Roll it region-by-region.
4. Provision `broker_superuser_tenants` only for genuine fabric-admin/god-view clients.

**Failure modes & what they look like:**
- A `DENY <READ|WRITE> '<dest>' for <identity> : <reason>` in a broker log is the ACL working. On a
  *legit* flow it means a namespacing mismatch — usually a producer stamping the wrong tenant or an infra
  node not using the `T.*.` wildcard inbox.
- Messages silently not delivered under namespacing (no DENY) ⇒ producer qualified with tenant X but the
  consumer subscribes tenant Y (a genuine cross-tenant send — correct isolation), *or* an infra node was
  mis-detected as a leaf and got a tenant-specific inbox instead of the wildcard.

---

## 11. End-to-end message-flow walkthrough (namespacing on)

Trace a request from **agent1 (tenantA, region-a)** to **agent2 (tenantA, region-c)**, relayed
region-a → global → region-c. Local tenant of each node in parentheses.

```
agent1 (tenantA)                 region-a ctrl (tenantA)      global ctrl (global)         region-c ctrl (tenantA)      agent2 (tenantA)
   |  produce, dst=region-c_agent2                                                                                          |
   |  AgentProducer: no cresco_tenant param -> stamp tenantA; qualify -> T.tenantA.region-c_agent2                          |
   |  send to its regional ctrl inbox (T.tenantA.region-a_ctrlA)                                                            |
   |------------------------------->|                                                                                       |
   |            (network client, mTLS tenantA; ACL: writing own T.tenantA.* -> ALLOW)                                       |
   |                                | MsgRouter relays; AgentProducer sees cresco_tenant=tenantA (preserved)                |
   |                                | qualify raw fwd dst with tenantA -> T.tenantA.<global>_<globalAgent>                   |
   |                                |=========bridge========>|                                                              |
   |                                |   (bridge exempt; demand-forwards T.tenantA.* on demand)                              |
   |                                |                        | global ctrl on vm:// consumes T.*.<global>_<globalAgent>   |
   |                                |                        | (wildcard infra inbox -> receives any tenant stamp)        |
   |                                |                        | relays; qualify with preserved tenantA                     |
   |                                |                        |=========bridge========>|                                     |
   |                                |                        |         (demand-forwards T.tenantA.*)                       |
   |                                |                        |                        | region-c ctrl vm:// T.*. inbox     |
   |                                |                        |                        | relays to leaf; qualify tenantA    |
   |                                |                        |                        |----------------------------------->|
   |                                |                        |                        |  agent2 network client, mTLS       |
   |                                |                        |                        |  consumes T.tenantA.region-c_agent2|
   |                                |                        |                        |  (ACL: own tenant -> ALLOW)        |
```

Key invariants this shows: **(a)** the tenant is stamped once at origin and *preserved* across every relay,
so every hop qualifies to the identical queue name; **(b)** infra controllers (vm://) use the `T.*.`
**wildcard** inbox so they relay any tenant regardless of the stamp; **(c)** the leaf's own regional
controller is always same-tenant, so the last hop qualifies correctly; **(d)** a cross-tenant send would
qualify to `T.tenantB.…`, which agent2 (tenantA) never consumes *and* which agent1's own ACL would deny —
isolation is enforced twice over.

---

## 12. Security properties & threat model

**Guaranteed (with `broker_security_enabled` + `broker_require_client_auth` + `tenant_namespacing`):**
- *Confidentiality (client-to-client):* a `TENANT` client cannot **subscribe** to any destination outside
  its own `T.<tenant>.` subtree at **any** broker in the mesh — enforced locally at every broker from the
  cert-bound tenant, so it holds across bridges and across globals.
- *Delivery integrity:* a `TENANT` client cannot **publish** into another tenant's subtree; the flat-name
  same-region write hole (tenant B injecting into tenant A's `region_agent` inbox) is closed because inboxes
  are now `T.<tenant>.`-scoped.
- *Non-spoofable identity:* under mTLS the tenant is derived from the validated client-cert DN and overrides
  any self-asserted username.

**Explicitly NOT guaranteed by L1+L2 (documented, tiered mitigations):**
- *A malicious/compromised BROKER* is trusted infrastructure (it holds a region-CA identity and relays all
  tenants); it can read/inject across tenants. Mitigation: L3 dedicated per-tenant bridges (blast radius)
  and L4 payload encryption (confidentiality from the broker itself). Out of scope for L1+L2.
- *Payload confidentiality at rest inside a transit broker.* mTLS encrypts each hop in flight; a payload
  sitting in a transit broker's store is readable by that broker. Mitigation: L4 envelope encryption (opt-in).
- *Advisory destination-existence leak.* `ActiveMQ.Advisory.*` is allow-listed; a tenant can learn other
  tenants' destination *names* (not contents) via advisories. Mitigation: scope/suppress advisories (future).
- *Metadata:* routing headers (tenant, src/dst region/agent) travel in clear so brokers can route; only
  payloads are candidates for L4 encryption.

**Trust boundary:** tenants trust the Cresco infrastructure (controllers/brokers) to relay faithfully;
they do **not** trust each other. That is the standard shared-fabric multi-tenant model and matches how
NATS accounts / Solace VPNs draw the line.

---

## 13. Verification (shipped tests)

| Test | Result | What it proves |
|---|---|---|
| `run/tests/tenant_namespacing_test.sh` (new) | **12/12** | Federated 1-global/2-tenant-region mesh forms under `tenant_namespacing=true`; **zero self-denial** on every node's control plane (⇒ MsgEvent `T.*.` wildcard inbox + producer tenant-qualification work end-to-end); `TENANT` confined to own `T.<tenant>.*`, denied cross-tenant and `T.*.` wildcard; `SUPERUSER` (`cresco-system`) gets cross-tenant god view. |
| `run/tests/tenant_isolation_mesh_test.sh` | 10/10 | **No regression** with namespacing off (default): legacy `<tenant>.stream` ACL isolation intact across the federated mesh. |
| `run/tests/security_foundation_test.sh` | 15/15 | Identity/sign/verify/chain-trust primitives intact. |
| `run/tests/regional_ca_mtls_test.sh` | 10/10 | Regional-CA enrollment + fabric-wide mutual-TLS federation intact. |

Run all four after any change to the auth or namespacing path; each must stay green with the flags both
**off** (default) and **on**.

---

## 14. Implementation map (where each layer lives)

| Layer | File(s) | Symbol |
|---|---|---|
| L1 namespace helper | `library/.../security/TenantNamespace.java` | `qualify` / `wildcard` / `tenantOf` / `isNamespaced` |
| L1 MsgEvent producer | `controller/.../communication/AgentProducer.java` | `sendMessage` — stamp `cresco_tenant` + qualify `dstPath` |
| L1 MsgEvent inbox | `controller/.../statemachine/ControllerSMHandler.java` | `initIOChannels` — `T.*.` wildcard (vm://) vs `T.<tenant>.` (network) |
| L1 dataplane | `controller/.../data/DataPlaneServiceImpl.java` | `getTopicName` — qualify topics |
| L2 policy | `controller/.../communication/TenantPolicy.java` | `Role` enum + namespaced-prefix rule in `check` |
| L2 role resolution | `controller/.../communication/CrescoAuthorizationBroker.java` | `roleOf` from `broker_superuser_tenants` |

---

## 15. Limitations & next work

- **L3 — dedicated per-tenant bridges** (blast-radius / physical isolation). Core isolation already holds
  via client-edge ACL; L3 is a high-assurance tier. The mechanism is a per-connector
  `setDynamicallyIncludedDestinations(T.<tenant>.>)` (the same lever `ActiveBroker.buildConnector` already
  uses for shard partitioning).
- **L4 — payload envelope encryption** (opt-in; protects against a compromised transit broker).
- **Advisory scoping** — close the destination-existence leak (§12).
- **Infra cross-tenant dataplane aggregation** — with topics qualified by local tenant, a global "god
  view" of all tenants' dataplane streams needs a `T.*.` wildcard *consumer* on infra (produce stays
  tenant-scoped). Not built; per-tenant monitoring is arguably the more correct default under isolation.
- **App-destination enforcement** — application destinations that opt out of the `T.` prefix still fall to
  the legacy tenant rules; a strict mode could require every non-shared destination to be namespaced.
