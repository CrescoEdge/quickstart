# MsgEvent

`MsgEvent` is the single message envelope that carries **everything** on the Cresco fabric — control
commands, discovery, telemetry, logs, RPC requests and their replies. Every plugin action is invoked by
sending a `MsgEvent` and (for RPC) reading the `MsgEvent` that comes back. There is no other on-wire message
type: a client reaching Cresco over the WebSocket bus, a plugin talking to its controller, and a controller
federating with the global tier all speak `MsgEvent`.

Source: `code/library/src/main/java/io/cresco/library/messaging/MsgEvent.java`.

For how these messages are actually moved between agents, regions, and the global tier, see
[Messaging & Routing](../architecture/messaging.md).

## Anatomy

A `MsgEvent` has three parts:

1. **A typed header** — the message `Type`, the source identity (`src_region`/`src_agent`/`src_plugin`),
   the destination identity (`dst_region`/`dst_agent`/`dst_plugin`), and two routing flags (`isRegional`,
   `isGlobal`).
2. **Two string→string maps** — `params` (the application payload, including the `action` key and its
   arguments) and `msgparams` (transport-internal metadata, e.g. the outgoing-files flag).
3. **An optional file list** — paths of files to ship alongside the message.

```
┌───────────────────────── MsgEvent ─────────────────────────┐
│ header:  msgType (Type)                                     │
│          src_region / src_agent / src_plugin                │
│          dst_region / dst_agent / dst_plugin                │
│          isRegional  isGlobal                               │
│ params:      { "action": "listagents", "action_region": …, │
│                "status": "10", ... }   ← app payload / RPC  │
│ msgparams:   { "HAS_OUTGOING_FILES": "true", ... } ← transport│
│ fileList:    [ "/path/one.jar", ... ]                       │
└─────────────────────────────────────────────────────────────┘
```

An identity field being `null` is meaningful: a `dst_plugin` of `null` addresses the **agent controller**
itself rather than a plugin on that agent.

## Type enum

`MsgEvent.Type` selects the executor entry point on the receiving bundle
(`executeEXEC`, `executeCONFIG`, …). The `(type, action)` pair is the dispatch key.

| Type | Purpose |
|------|---------|
| `CONFIG` | State-changing commands: add/remove plugins, configure tunnels, enable/disable regions, kill/restart. |
| `DISCOVER` | Broadcast/agent discovery handshakes used to bootstrap the mesh. |
| `ERROR` | Error notification / failed-delivery signalling. |
| `EXEC` | Read/query and RPC actions: list agents, fetch metrics, ping, capability inventory. |
| `GC` | Garbage-collection / cleanup signalling. |
| `INFO` | Informational notices between nodes. |
| `KPI` | Key performance indicator / telemetry samples flowing toward the collectors. |
| `LOG` | Log records forwarded across the fabric. |
| `WATCHDOG` | Periodic liveness heartbeats used by the health subsystem. |

By convention **`EXEC`** carries read/query actions and **`CONFIG`** carries mutating actions; each plugin
action's required `Type` is documented in [Plugin Actions](plugin-actions.md).

## Public API

### Construction

| Constructor | Notes |
|-------------|-------|
| `MsgEvent()` | Empty envelope (used by (de)serializers). |
| `MsgEvent(Type, src_region, src_agent, src_plugin, dst_region, dst_agent, dst_plugin, isRegional, isGlobal)` | Full header. Initializes empty `params`/`msgparams`. The canonical way to build a routed message. |
| `MsgEvent(Type, msgRegion, msgAgent, msgPlugin, msgBody)` | Convenience constructor that sets the **source** identity; the legacy `msgBody` argument has no accessor. |

In practice application code rarely calls these constructors directly — the plugin `PluginBuilder` and the
client libraries expose helpers (e.g. `global_controller_msgevent`, `plugin_msgevent`) that build a
correctly-addressed envelope for you. See [Plugin Actions](plugin-actions.md#invoking-an-action).

### Header accessors

| Method | Returns / Effect |
|--------|------------------|
| `getMsgType()` / `setMsgType(Type)` | The message `Type`. |
| `getSrcRegion()` / `getSrcAgent()` / `getSrcPlugin()` | Source identity components. |
| `getDstRegion()` / `getDstAgent()` / `getDstPlugin()` | Destination identity components. |
| `setSrc(region, agent, plugin)` | Set source identity (writes `src_*` params). |
| `setDst(region, agent, plugin)` | Set destination identity (writes `dst_*` params). |
| `isRegional()` | `true` if the message is scoped to a region controller. |
| `isGlobal()` | `true` if the message is scoped to the global controller. |
| `printHeader()` | Human-readable one-line dump of type + src + dst. |

### Routing helpers

| Method | Effect |
|--------|--------|
| `setForwardDst(region, agent, plugin)` | Overwrite the destination header in place (used by relays when forwarding). |
| `getForwardDst()` | The next-hop key: `region_agent`, or just `region`, or `null` — how a relay decides where to send next. |
| `dstIsLocal(localRegion, localAgent, localPlugin)` | `true` when this node is the message's final destination (agent match, and plugin match or both `null`). |
| `setReturn()` | **Swap source and destination** so the message becomes a reply headed back to its origin. The cornerstone of the RPC pattern — a handler mutates `params`, calls `setReturn()`, and hands the envelope back. |

### Params (application payload)

| Method | Effect |
|--------|--------|
| `getParams()` / `setParams(Map)` | The whole payload map. |
| `getParam(key)` / `setParam(key, value)` | Read / write one string param. |
| `removeParam(key)` | Delete a param. |
| `paramsContains(key)` | Membership test. |

The `action` param names the plugin action to run; everything else in `params` is either an action
argument or, on the reply, a result value.

### Binary and compressed params

Large or structured values ride as Base64 (optionally gzip-compressed) strings so they fit in the
string→string map. Compressed returns are marked *compressed* in the action tables on the
[Plugin Actions](plugin-actions.md) page.

| Method | Effect |
|--------|--------|
| `setDataParam(key, byte[])` | Store raw bytes as Base64. |
| `getDataParam(key)` | Decode a `setDataParam`/binary value back to `byte[]`. |
| `setCompressedParam(key, String)` | Store a string gzip-compressed + Base64. |
| `getCompressedParam(key)` | Decompress + decode back to the original string. Returns `null` if absent/undecodable. |
| `setCompressedDataParam(key, byte[])` | Store bytes gzip-compressed + Base64. |
| `stringCompress(String)` / `dataCompress(byte[])` | Low-level gzip helpers returning the compressed `byte[]`. |

!!! note "Reading compressed returns from a client"
    A `status` param of `10` conventionally means success and `9` means failure. Structured returns
    (agent lists, metrics, capability documents) are usually compressed — read them with
    `getCompressedParam()` in Java, or `decompress_param()` + `json_deserialize()` in
    [pycrescolib](../clients/python.md).

### File transfer

| Method | Effect |
|--------|--------|
| `addFile(path)` | Attach a file to ship with the message (sets the `HAS_OUTGOING_FILES` flag). |
| `addFiles(List<String>)` | Attach several files. |
| `getFileList()` | The attached paths. |
| `hasFiles()` | `true` if outgoing files are attached. |
| `clearFileList()` | Detach all files and clear the flag. |

## Addressing and routing

`MsgEvent` addressing is **hierarchical**: `region → agent → plugin`. Routing is driven by the destination
header plus the two scope flags:

- **`isGlobal`** — deliver to the **global controller** (the fabric-wide brain).
- **`isRegional`** — deliver to the **region controller** for `dst_region`.
- Neither flag set — deliver to the agent named by `dst_region`/`dst_agent`, and, if `dst_plugin` is set,
  to that plugin's executor; if `dst_plugin` is `null`, to the agent controller itself.

Each hop uses `getForwardDst()` to pick the next relay and `dstIsLocal(...)` to decide whether it has
arrived. A reply is produced by calling `setReturn()`, which flips the source and destination so the
envelope naturally routes back to the caller. The full routing model — brokers, bridges, and the
global/regional/agent tiers — is described in [Messaging & Routing](../architecture/messaging.md).

## Example: an RPC request and reply

An RPC is a request `MsgEvent` sent with "wait for reply" semantics; the handler mutates the same envelope,
calls `setReturn()`, and the reply routes home.

### Request — `EXEC listagents` to the global controller

```json
{
  "type": "EXEC",
  "isGlobal": true,
  "src_region": "edge-region",
  "src_agent": "edge-agent-001",
  "src_plugin": "plugin/0",
  "dst_region": "global-region",
  "dst_agent": "global-controller",
  "dst_plugin": null,
  "params": {
    "action": "listagents",
    "action_region": "edge-region"
  }
}
```

### Reply — after the handler ran `setReturn()`

Source and destination are now swapped, and the result rides in `params`. `status` `10` means success;
`agentslist` is a compressed JSON payload (shown here decompressed).

```json
{
  "type": "EXEC",
  "isGlobal": true,
  "src_region": "global-region",
  "src_agent": "global-controller",
  "src_plugin": null,
  "dst_region": "edge-region",
  "dst_agent": "edge-agent-001",
  "dst_plugin": "plugin/0",
  "params": {
    "action": "listagents",
    "status": "10",
    "agentslist": {
      "agents": [
        { "agent_id": "global-controller", "region_id": "global-region", "plugins": "4" },
        { "agent_id": "edge-controller",   "region_id": "edge-region",   "plugins": "2" },
        { "agent_id": "edge-agent-001",    "region_id": "edge-region",   "plugins": "2" }
      ]
    }
  }
}
```

## See also

- [Plugin Actions (Capability Inventory)](plugin-actions.md) — the full catalog of actions a `MsgEvent`
  can carry, per plugin, with params and returns.
- [Messaging & Routing](../architecture/messaging.md) — how envelopes traverse the fabric.
- [Java client (clientlib)](../clients/java.md) · [Python client (pycrescolib)](../clients/python.md) —
  helpers that build and send `MsgEvent`s for you.
