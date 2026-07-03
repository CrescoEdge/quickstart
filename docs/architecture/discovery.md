# Discovery

Before a node can join the mesh it must **find** a parent (region/global) and prove it is **allowed** to
join. Cresco does this with a discovery protocol gated by a pre-shared secret, implemented under the
controller's `netdiscovery` package.

## How a node finds a parent

| Mechanism | When |
|-----------|------|
| **UDP broadcast/multicast** (`UDPDiscoveryEngine`) | dynamic discovery on a local network — a joining node broadcasts a `DISCOVER`, listening controllers respond |
| **Static/TCP** (`TCPDiscoveryEngine`, `TCPDiscoveryStatic`) | directed discovery to a known host:port (e.g. `global_controller_host`) — used across networks where broadcast doesn't reach |

Discovery exchanges `DiscoveryNode` records carrying the peer's region/agent identity, IP/port, agent
count, a validator, and a measured latency. Discovery runs per tier: `AGENT`, `REGION`, `GLOBAL`,
`NETWORK`.

## The join secret

A node proves it may join by demonstrating knowledge of the **tier join secret**. There is one secret per
tier:

| Config | Gate |
|--------|------|
| `discovery_secret_agent` | joining as an agent |
| `discovery_secret_region` | joining as a region |
| `discovery_secret_global` | joining as a global |

The joining node encrypts a known challenge with the secret; the responder decrypts it with its configured
secret and only proceeds if the challenge matches. A node that doesn't hold the right secret cannot join.

!!! danger "Set your secrets"
    If `discovery_secret_*` are unset a node generates random values and nobody can join it. For a real
    fabric, set fixed shared values (as `run/cresco.env` does). See [Configuration](../getting-started/configuration.md).

## Certification (establishing trust at join)

Discovery is also where nodes exchange the cryptographic material that later secures the broker
connections. Two modes (see [Security & Identity](security.md)):

- **Direct trust (default):** peers exchange leaf certificates during discovery and add each other to
  their trust stores.
- **Regional-CA (`security_regional_ca=true`):** the responder (a region/global acting as an issuing
  authority) signs the joiner a leaf certificate under its CA and returns the chain; the joiner installs
  it and trusts the issuing CA. This collapses trust material from O(nodes²) to O(regions).

The join secret is the enrollment gate for both — you must pass the secret challenge before any certificate
is issued or trusted.

## After discovery

Once a parent is found and trust is established, the node opens its [messaging](messaging.md) channels
(inbox queue + producers), and — if it is a controller — starts its broker and the **federation bridge**
to its parent. Liveness [health](health.md) checks then keep the parent link monitored.
