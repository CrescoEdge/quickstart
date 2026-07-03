# Client Libraries

Cresco ships two external client SDKs for driving a mesh from outside the OSGi
runtime:

- **[Java — `clientlib`](java.md)** — the Java SDK (`io.cresco:clientlib`).
- **[Python — `pycrescolib`](python.md)** — the Python SDK.

Both connect to a running Cresco agent that hosts the **`wsapi`** plugin over a
secure WebSocket (`wss://host:8282`) and drive the fabric — controlling agents,
deploying plugins, submitting pipelines, streaming the data plane, and tailing
logs.

## The shared model

Every interaction follows the same shape regardless of language:

1. **Connect once.** Construct a client with the target host, port, and the
   `wsapi` service key, then `connect()`. The client opens a single
   authenticated control connection to the global controller's `wsapi`
   endpoint. Identity (region, agent, plugin) is read from the negotiated TLS
   peer certificate.
2. **Use submodule handles.** The connected client exposes a fixed set of
   submodules, each grouping a family of operations:

    | Submodule | Purpose |
    |---|---|
    | `api` | Local and global identity lookups (region/agent/plugin names). |
    | `admin` | Node lifecycle on a target agent (stop/restart controller, restart framework, kill JVM). |
    | `agents` | Per-agent operations: plugin add/remove/list/status, uploads, logs, CEP. |
    | `globalcontroller` | Mesh-wide inventory and application ops: regions, agents, resources, pipelines, repositories, metric/capability inventories. |
    | `messaging` | Low-level MsgEvent routing (RPC and fire-and-forget) to controllers, agents, and plugins. |

3. **Open streams as needed.** Data-plane and log-streamer channels are
   *separate* WebSocket connections opened on demand (`get_dataplane(...)`,
   `get_logstreamer(...)`). They are tracked by name on the client, delivered to
   a callback, and auto-reconnect if the socket drops.

4. **Close.** `close()` tears down the control connection and every managed
   stream.

### RPC vs. fire-and-forget

Messaging calls carry an `is_rpc` flag. When `true`, the call blocks for the
single reply and returns it; when `false`, the message is sent without waiting.
The `wsapi` protocol is one-outstanding-RPC-per-connection with no correlation
id, so each RPC completes before the next is issued.

## Feature parity

!!! note "Standardized clients"
    The Python and Java clients are kept **feature- and name-identical**: the
    same submodules (`api`, `admin`, `agents`, `globalcontroller`, `messaging`),
    the same method names (snake_case in both languages), the same behavior, and
    the same worked examples. Anything you can do in one client, you can do in
    the other with the same call.

Parity is enforced by test: both clients ship a self-contained suite that
asserts they emit **identical wire messages for identical API calls**, verified
against the same generated golden corpus. A green run on both proves the two
clients produce byte-for-byte equivalent results.

## SSL verification

Cresco agents present self-signed certificates by default, so client SSL
certificate verification is **disabled by default**. Both clients accept a
`verify_ssl` constructor flag to enable it when the mesh is fronted by a
trusted certificate authority.

## Where to go next

- **[Java (clientlib)](java.md)** — installation, connection, full interface
  reference, and worked examples.
- **[Python (pycrescolib)](python.md)** — the equivalent Python SDK.
- **[MsgEvent](../api/msgevent.md)** — the message envelope and routing model
  the `messaging` submodule builds on.
- **[Plugin Actions](../api/plugin-actions.md)** — the action catalog exposed by
  controllers and plugins.
