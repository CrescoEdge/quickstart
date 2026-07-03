# Region↔Global federation on one host — root cause & design

Evidence-based findings from a deep read of the controller source (file:line cited inline).
Goal: let a **regional controller (case 24)** federate to a **global** when both run on one host,
and fix the conflated self-detection so two agents on one host work in general.

## TL;DR

There are **two independent problems**. The self-guard (`isLocal`) the symptom pointed at is
*mostly a red herring* for the PONG failure; the real blocker is message **routing through a
duplex broker bridge** whose reverse leg doesn't deliver same-host.

1. **PONG failure (the actual case-24 blocker) — structural routing, not the self-guard.**
   A region routes *all* global-bound messages (including the health PING) through its **own**
   `vm://localhost` broker and relies on a single duplex ActiveMQ network bridge
   (`ActiveBroker.AddNetworkConnector`, `setDuplex(true)` — ActiveBroker.java:405-420) to
   demand-forward them to the global. The global never creates a return bridge (candidate
   brokers are enqueued only region→global at ControllerSMHandler.java:796). The PING reaches
   the global (forward leg works → registration succeeds), but the **PONG must travel back over
   the duplex bridge's reverse leg**, which is not established/maintained same-host → `sendRPC`
   returns null → `RegionHealthWatcher.ActivePingTask` logs "No PONG … GLOBAL CONTROLLER LOST"
   (RegionHealthWatcher.java:391-395) and loops.
   - A plain **agent (case 6)** avoids this entirely: `isAgent()` passes the remote broker IP into
     `initIOChannels`, so its producer/consumer attach **directly** to the global broker via
     `failover:(nio+ssl://HOST:32010)` (ControllerSMHandler.java:619-630, 1009-1013). No bridge,
     no reverse leg → works same-host.

2. **`isLocal(ip)` self-detection — conflates two purposes; one is dead code, one is a real
   same-host drop, neither is what breaks case 24 (which uses static-TCP discovery).**
   - **P1 discovery-dedup** at `ActiveBrokerManager.java:60-63` is a **no-op today** — the block
     only logs the "REMOVED BLOCKED BROKER CONNECTION FOR LOCALHOST" warning; nothing is skipped
     (the real skip was commented out). Self-dedup actually happens by identity via the
     `brokeredAgents` map key = `getDiscoveredPath()` (region_agent).
   - **Real same-host drop**: `UDPDiscoveryEngine.java:258` — a node won't answer a UDP broadcast
     whose source IP is one of its own NIC IPs, so **same-host UDP discovery is silently dropped**.
     Case 24 doesn't hit this because it uses *static TCP* discovery (`exchangeKeyWithBroker`,
     port 32005), which has no IP self-filter (identity/cert-gated) — that's why registration works.
   - **P2 connection-method** `isLocalBroker(addr)` (ControllerSMHandler.java:1022) only matches the
     literals `"localhost"`/`"[::]"` (the region's own broker sentinel). It is **not IP-based and
     not broken** — a discovered same-host peer arrives as a real IP and correctly uses TCP.

3. **Secondary `getRegionHealthWatcher()` NPE.** PONG-timeout recovery nulls the watcher
   (`globalControllerLost`→`isRegionalGlobalShutdown`→`setRegionHealthWatcher(null)`,
   ControllerSMHandler.java:843-844) while the watcher's own daemon timers / message routing still
   dereference it (RegionHealthWatcher.java:308; MsgRouter.java:30) → NPE during the recovery window.

## The identity primitive (for any self/peer check)

`agentPath = region_id + "_" + agent_id` (`ControllerStateImp.getAgentPath()`:105-107) is the stable,
unique-per-instance identity. The discovery response already carries the peer's via
`DiscoveryNode.getDiscoveredPath()` (DiscoveryNode.java:95-97) — **same format**, available at every
self-check call site. Per-agent X.509 certs exist (alias = agentPath) but the CN is randomized and the
cert is null until the CERTIFY exchange, so use a cert-fingerprint only as optional hardening, not the
primary key. Discovery secrets/validators are **fabric-shared** — useless for telling two agents apart.

## Proposed fixes (ranked)

**Fix A — make region global-bound RPCs go direct to the global broker (fixes the PONG; highest value).**
Mirror the agent path: give the region a producer/consumer attached to the global broker
(`failover:(nio+ssl://global:32010)`) for global-destined messages, instead of depending on the duplex
bridge's reverse leg. Touch points: `MsgRouter.forwardToRemoteGlobal/forwardToRemoteRegion`
(MsgRouter.java:46-75) + the region's IO-channel setup (ControllerSMHandler.regionInit/initIOChannels).
This removes same-host dependence on bridge reverse-demand-forwarding.
*(Alternative A′: keep the bridge but ensure the global creates/sustains the reverse leg, or verify
duplex reverse demand-forwarding for the region's RX queue — more ActiveMQ-internal, less certain.)*

**Fix B — replace IP self-detection with identity (fixes same-host UDP discovery + removes the latent trap).**
- `UDPDiscoveryEngine.java:258`: gate the responder skip on
  `discoveryNode.getDiscoveredPath().equals(cstate.getAgentPath())` (identity), not on
  `intAddr.containsKey(remoteAddress)` (IP). Lets two same-host agents discover via UDP while still
  not answering *itself*.
- `ActiveBrokerManager.java:60-63`: implement the real self-skip by identity
  (`getDiscoveredPath().equals(cstate.getAgentPath())`) instead of the dead IP warn.
- Leave `isLocalBroker` as-is (it's identity-correct via the sentinel); do **not** widen it to match IPs.

**Fix C — null-safe health-watcher teardown (fixes the NPE).**
Guard `getRegionHealthWatcher()` dereferences and/or cancel the watcher's timer tasks before nulling it
in `isRegionalGlobalShutdown()`.

## Why this matters for the test plan

On this host (VPN owns the default route; only `127.0.0.1` is same-host-reachable), **Fix A is what makes
a same-host regional controller viable** — without a second routable host or Docker. Until then, the
verified working multi-node topology is **global + plain agent (case 6) over loopback**.
