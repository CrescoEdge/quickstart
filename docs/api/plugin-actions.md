# Plugin Actions (Capability Inventory)

Every Cresco capability — from listing agents to configuring a tunnel — is a **message action**: a
[`MsgEvent`](msgevent.md) of a given `Type` carrying an `action` parameter. A plugin (or controller tier)
handles the action and, for RPC, returns a reply `MsgEvent` whose params carry the result. Because an
external client reaches Cresco only over the WebSocket message bus (the [`wsapi`](../plugins/wsapi.md)
plugin), **actions are the entire callable surface** — there are no direct method calls into the fabric.

This page is the canonical human reference for every action across the fabric — **89 actions** over 7
namespaces (the three controller tiers plus the `sysinfo`, `stunnel`, `wsapi`, and `repo` plugins). The same
information is produced, machine-readable, by the capability inventory described below.

## The capability model

Actions are **self-describing**. Each handler method is annotated so the fabric can emit, for every action,
an LLM-ready tool definition (`name` / `description` / `input_schema`) plus a Cresco *binding* telling a
caller how to build and route the `MsgEvent`.

### `@CrescoAction` and friends

The annotations live in `io.cresco.library.capability`
(`code/library/src/main/java/io/cresco/library/capability/`):

| Annotation | Placed on | Key fields |
|------------|-----------|------------|
| `@CrescoCapabilities` | the `Executor` class (bundle/tier) | `namespace` (tool-name prefix), `summary`, default `target`, default `routingParams` |
| `@CrescoAction` | each handler method | `name` (the `action` value), `type` (MsgEvent Type, default `EXEC`), `summary` (WHAT), `why` (WHEN/WHY), `params()`, `returns()`, optional `target`/`routingParams` overrides |
| `@CrescoParam` | nested in `params()` | `name` (the param key), `type` (JSON-Schema type), `required`, `description`, `compressed` |
| `@CrescoReturn` | nested in `returns()` | `name` (reply param key), `type`, `description`, `compressed` |

The `(type, name)` pair is the dispatch key: an `EXEC` message with `action=name` routes to that method.
`summary` + `why` become the tool description; `params()` becomes the `input_schema`. A drift test asserts
that every `switch` case in an executor has a matching `@CrescoAction`, so **the inventory can never
silently disagree with the code**.

Bundle symbolic name and **version come from the OSGi manifest** via `FrameworkUtil.getBundle(clazz)` —
never hardcoded. A bundle answers the standard `getcapabilities` EXEC in one line via
`CapabilityResponder.respond(incoming, this)`, which reflects its annotations into a `CapabilityDocument`
JSON.

### `getcapabilityinventory`

Every bundle exposes `getcapabilities` (its own document). The controller additionally exposes
**`getcapabilityinventory`**, which aggregates:

- the three controller-tier executors' own actions (scanned statically), plus
- each local plugin's `getcapabilities` (over RPC), plus
- optionally, the OSGi surface (`Export-Package` headers and registered service interfaces).

With `action_scope=global` it fans out to every region and agent and returns **one fabric-wide tool
catalog**. `action_scope` may be `node`, `region`, or `global`; `action_include_plugins` and
`action_include_osgi` toggle the plugin and OSGi halves. The OSGi surface is *informational metadata only* —
a tool-runner walks the message actions, never the OSGi services.

Tool names are formed as `cresco_<namespace>_<action>` (e.g. `cresco_global_listagents`). In the tables
below the `<namespace>` prefix is dropped; the section heading is the namespace.

## Invoking an action

Invoking an action is always the same three steps:

1. **Build a `MsgEvent`** of the action's `Type`, addressed to the action's `target`, with
   `params["action"]` set to the action name and the action's arguments added as params.
2. **Send it** (as RPC if you want the reply) from a client library or from another plugin.
3. **Read the reply** — the result rides in the reply's params; large/structured values are
   gzip+Base64 *compressed* (marked in the Returns column).

The routing `target` determines which helper builds the envelope:

| `target` | Reached with (client helper) | Handled by |
|----------|------------------------------|------------|
| `global`   | `global_controller_msgevent(is_rpc, type, payload)` | GlobalExecutor (global controller) |
| `regional` | region-controller message (usually via the global fan-out) | RegionalExecutor (region controller) |
| `agent`    | `global_agent_msgevent(is_rpc, type, payload, region, agent)` | AgentExecutor (that agent) |
| `plugin`   | `global_plugin_msgevent(is_rpc, type, payload, region, agent, pluginid)` | that plugin's Executor |

`plugin`- and `agent`-target actions require **routing identity** params (`region`, `agent`, and for
plugins `pluginid`) so the caller can address the right node; these are listed in the Key-params column.

### Example — `cresco_global_listagents`

An `EXEC` action on the `global` target. It takes an optional `action_region` filter and returns a
compressed `agentslist`.

=== "Python (pycrescolib)"

    ```python
    reply = client.messaging.global_controller_msgevent(
        True,                      # is_rpc — wait for the reply
        "EXEC",                    # MsgEvent Type
        {"action": "listagents",   # the action
         "action_region": "edge-region"},
    )
    agents = json_deserialize(decompress_param(reply["agentslist"]))
    for a in agents["agents"]:
        print(a["agent_id"], a["region_id"], a["plugins"])
    ```

=== "Reply MsgEvent params"

    ```json
    {
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
    ```

A `status` of `10` means success and `9` means failure. See [MsgEvent](msgevent.md) for the envelope,
`setReturn()` reply mechanics, and compressed-param handling.

!!! note "Global vs regional are not duplicates"
    Several actions (`getmetricinventory`, `getcapabilities`, `ping`, …) exist on both the **global** and
    **regional** tiers with the same name. They differ by **scope**: the global tier aggregates across
    every region it sees (a fabric-wide report); the regional tier answers only for its own region. Treat
    the tier as part of the action's identity.

---

## Action reference

Every action below is reachable as a `MsgEvent`. Columns: **Action** (bare action name; tool name is
`cresco_<namespace>_<action>`), **Type** (the MsgEvent `Type`), **Description**, **Key params** (routing
identity + notable arguments), and **Returns** (principal reply params; *compressed* where noted).

### global tier

Routed to the **global controller** (`global_controller_msgevent`). This is the fabric-wide control API:
registry, discovery, resource inventory, and global application (pipeline) scheduling. Reports at this tier
**aggregate across every region** the global sees. Namespace `global` — **32 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `addplugin` | CONFIG | Schedule an iNode (plugin instance) for deployment. | `inode_id`, `resource_id`, `configparams` | `status_code`, `status_desc` |
| `getcapabilities` | EXEC | Return the global controller's self-describing capability document. | none | `capabilities` |
| `getcapabilityinventory` | EXEC | Fabric-wide capability inventory: tiers + plugins + OSGi as one tool catalog. | `action_scope`, `action_include_plugins`, `action_include_osgi` | `capabilityinventory` |
| `getenvstatus` | EXEC | Count nodes matching an environment index key=value. | `environment_id`, `environment_value` | `count` |
| `getgpipeline` | EXEC | Get a pipeline definition by id. | `action_pipelineid` | `gpipeline` |
| `getgpipelineexport` | EXEC | Get a pipeline in export (portable) format. | `action_pipelineid` | `gpipeline` |
| `getgpipelinestatus` | EXEC | Get status of one pipeline or all pipelines. | `action_pipeline` | `pipelineinfo` |
| `getinodestatus` | EXEC | Get an iNode (plugin instance) status map from the DB. | `inode_id` | `inodemap` |
| `getisassignmentinfo` | EXEC | Check assignment info for an iNode/resource. | `action_inodeid`, `action_resourceid` | `isassignmentinfo`, `isassignmentresourceinfo` |
| `getmetricinventory` | EXEC | Unified metric inventory: controller + plugins + resource summary. | `action_scope`, `action_include_plugins`, `action_include_resource` | `metricinventory` |
| `gpipelineremove` | CONFIG | Remove/undeploy a global pipeline by id. | `action_pipelineid` | `success` |
| `gpipelinesubmit` | CONFIG | Submit a global application pipeline for scheduling. | `action_gpipeline`, `action_tenantid` | `gpipeline_id` |
| `listagents` | EXEC | List agents in the fabric, optionally filtered by region. | `action_region` | `agentslist` *(compressed)* |
| `listplugins` | EXEC | List plugin instances, optionally filtered by region/agent. | `action_region`, `action_agent` | `pluginslist` |
| `listpluginsbytype` | EXEC | List plugin instances matching a key=value. | `action_plugintype_id`, `action_plugintype_value` | `pluginsbytypelist` |
| `listpluginsrepo` | EXEC | List all plugins available across the fabric's repos. | none | `listpluginsrepo` |
| `listregions` | EXEC | List all regions plus live inter-broker bridge info. | none | `regionslist` |
| `listrepoinstances` | EXEC | List all repo plugin instances in the fabric. | none | `listrepoinstances` |
| `netresourceinfo` | EXEC | Get network-wide resource totals. | none | `netresourceinfo` |
| `ping` | EXEC | Liveness ping; replies pong and exchanges mesh health. | none | `action` (pong) |
| `plugindownload` | CONFIG | Download a plugin JAR from a URL into the repo (cache-checked). | `pluginurl`, `agentcontroller`, `forceplugindownload` | `hasplugin` |
| `plugininfo` | EXEC | Get metadata for a specific plugin instance. | `action_region`, `action_agent`, `action_plugin` | `plugininfo` |
| `plugininventory` | EXEC | List plugins in the local repo jar directory (name=version). | none | `pluginlist` (CSV) |
| `pluginkpi` | EXEC | Get metrics attached to a specific plugin instance. | `action_region`, `action_agent`, `action_plugin` | `pluginkpi` |
| `region_disable` | CONFIG | Unregister a region from the global registry. | none | `is_unregistered` |
| `region_enable` | CONFIG | Register a region in the global registry. | none | `is_registered` |
| `regionalimport` | CONFIG | Register/import a regional node into the global DB. | none | (status only) |
| `removeplugin` | CONFIG | Schedule an iNode (plugin instance) for removal. | `inode_id`, `resource_id` | `status_code`, `status_desc` |
| `resourceinfo` | EXEC | Aggregated cpu/mem/disk for a region/agent or whole fabric. | `action_region`, `action_agent` | `resourceinfo` |
| `resourceinventory` | EXEC | Get fabric resource totals. | none | `resourceinventory` |
| `savetorepo` | CONFIG | Save/replicate a plugin JAR to all repo instances. | `configparams`, `jardata` | `is_saved` |
| `setinodestatus` | CONFIG | Set the status of an iNode (plugin instance) in the DB. | `action_inodeid`, `action_statuscode`, `action_statusdesc` | `success` |

### agent tier

Routed to a specific **agent controller** (`global_agent_msgevent`, requiring `region` + `agent`). This is
the per-node control API: plugin lifecycle on that agent, logs, files, controller/JVM lifecycle, and
dataplane log streaming. Namespace `agent` — **26 actions**. All actions take routing identity
`region`, `agent`.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `cepadd` | CONFIG | Add a CEP rule to the dataplane. | `region`, `agent`, `cepparams` | `cepid` |
| `controllerupdate` | CONFIG | Stage a controller JAR for the next restart. | `region`, `agent`, `jar_file_path` | (status only) |
| `getagentinfo` | CONFIG | Return the agent's data-directory path. | `region`, `agent` | `agent-data` |
| `getbroadcastdiscovery` | EXEC | Return this agent's network discovery list (slow ~10s scan). | `region`, `agent` | `broadcast_discovery` |
| `getcapabilities` | EXEC | Return the agent controller's self-describing capability document. | `region`, `agent` | `capabilities` |
| `getcapabilityinventory` | EXEC | Return this node's capability inventory (node scope). | `region`, `agent`, `action_include_plugins`, `action_include_osgi` | `capabilityinventory` |
| `getcontrollerstatus` | EXEC | Return this agent's controller state code. | `region`, `agent` | `controller_status` |
| `getfiledata` | EXEC | Stream a chunk of a file on this agent (seek + read). | `region`, `agent`, `filepath`, `skiplength`, `partsize` | `payload` *(compressed)* |
| `getfileinfo` | EXEC | Return metadata (md5, size) for a file on this agent. | `region`, `agent`, `filepath` | `md5`, `size` |
| `getislogdp` | CONFIG | Query whether dataplane log streaming is enabled for a session. | `region`, `agent`, `session_id` | `islogdp` |
| `getlog` | EXEC | Fetch this agent's main log (inline or file). | `region`, `agent`, `action_inmessage` | `log` *(compressed)* |
| `getmetricinventory` | EXEC | Return this node's unified metric inventory (node scope). | `region`, `agent`, `action_include_plugins`, `action_include_resource` | `metricinventory` |
| `iscontrolleractive` | EXEC | Return whether this agent's controller is active. | `region`, `agent` | `is_controller_active` |
| `killjvm` | CONFIG | Kill the agent JVM (async, **destructive**). | `region`, `agent` | (status only) |
| `listagents` | EXEC | List agents in this agent's region. | `region`, `agent`, `action_region` | `agentslist` *(compressed)* |
| `pluginadd` | CONFIG | Add/start a plugin bundle on this agent. | `region`, `agent`, `configparams`, `edges` | `status_code`, `pluginid` |
| `pluginlist` | CONFIG | List all plugins on this agent. | `region`, `agent` | `plugin_list` *(compressed)* |
| `pluginremove` | CONFIG | Stop/remove a plugin on this agent. | `region`, `agent`, `pluginid` | `status_code` |
| `pluginrepopull` | CONFIG | Validate a set of plugins against the repo. | `region`, `agent`, `configparams` | `is_updated` |
| `pluginstatus` | CONFIG | Get the status of one plugin on this agent. | `region`, `agent`, `pluginid` | `plugin_status` *(compressed)* |
| `pluginupload` | CONFIG | Upload a plugin JAR and register it. | `region`, `agent`, `configparams`, `jardata` | `is_updated` |
| `restartcontroller` | CONFIG | Restart this agent's controller (async, **destructive**). | `region`, `agent` | (status only) |
| `restartframework` | CONFIG | Restart the OSGi framework (async, **destructive**). | `region`, `agent` | (status only) |
| `setlogdp` | CONFIG | Enable/disable dataplane log streaming for a session. | `region`, `agent`, `session_id`, `setlogdp` | `status_code` |
| `setloglevel` | CONFIG | Set log level for a class/session. | `region`, `agent`, `session_id`, `loglevel`, `baseclassname` | `status_code` |
| `stopcontroller` | CONFIG | Stop this agent's controller (async, **destructive**). | `region`, `agent` | (status only) |

### regional tier

Routed to a **region controller** (`regional`; usually reached via the global fan-out). The per-region
control API: agent registry for that region, and region-scoped capability/metric/liveness reports.
Namespace `regional` — **6 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `agent_disable` | CONFIG | Unregister an agent from this region. | `region`, `agent` | `is_unregistered` |
| `agent_enable` | CONFIG | Register an agent in this region. | `region`, `agent` | `is_registered` |
| `getcapabilities` | EXEC | Return the regional controller's self-describing capability document. | `region`, `agent` | `capabilities` |
| `getcapabilityinventory` | EXEC | Return this region node's capability inventory. | `region`, `agent` | `capabilityinventory` |
| `getmetricinventory` | EXEC | Return this region node's unified metric inventory. | `region`, `agent` | `metricinventory` |
| `ping` | EXEC | Liveness ping; replies pong, exchanges mesh health. | `region`, `agent` | `action` (pong) |

### sysinfo plugin

Host telemetry plugin ([`sysinfo`](../plugins/sysinfo.md)) — `target=plugin`, so actions take routing
identity `region`, `agent`, `pluginid`. Namespace `sysinfo` — **4 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `getbenchmark` | EXEC | Run and return a SciMark2 CPU benchmark (compressed JSON). | `region`, `agent`, `pluginid` | benchmark *(compressed)* |
| `getcapabilities` | EXEC | Return this plugin's self-describing capability document. | `region`, `agent`, `pluginid` | `capabilities` |
| `getmetrics` | EXEC | Return live metrics (cpu/mem/disk/net/sensors/power/gpu). | `region`, `agent`, `pluginid` | `metrics` |
| `getsysinfo` | EXEC | Full host OS/CPU/memory/disk/fs/network snapshot. | `region`, `agent`, `pluginid` | `perf` *(compressed)* |

### stunnel plugin

Secure TCP tunneling plugin ([`stunnel`](../plugins/stunnel.md)) — `target=plugin` (`region`, `agent`,
`pluginid`). A tunnel has a **source** (listener) side and a **destination** side, configured
independently. Namespace `stunnel` — **12 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `configsrctunnel` | CONFIG | Create the source (listener) side of a TCP tunnel. | `region`, `agent`, `pluginid` | `status` |
| `configdsttunnel` | CONFIG | Configure the destination side (agent nearest the target host:port). | `region`, `agent`, `pluginid`, `action_tunnel_config` | `status` |
| `configdstsession` | CONFIG | Open a destination-side client session for an existing dst tunnel. | `region`, `agent`, `pluginid`, `action_session_config` | `status` |
| `getcapabilities` | EXEC | Return this plugin's self-describing capability document. | `region`, `agent`, `pluginid` | `capabilities` |
| `getmetrics` | EXEC | Return live tunnel metrics as MeasurementEngine gauges JSON. | `region`, `agent`, `pluginid` | `metrics` |
| `gettunnelconfig` | EXEC | Get the full configuration of one tunnel. | `region`, `agent`, `pluginid`, `action_stunnel_id` | `status` |
| `gettunnelstatus` | EXEC | Get live status of one tunnel (ACTIVE/RECOVERING/DOWN/UNKNOWN). | `region`, `agent`, `pluginid`, `action_stunnel_id` | `status` |
| `listtunnels` | EXEC | List all active tunnels on this node with live status. | `region`, `agent`, `pluginid` | `tunnels` |
| `nettuning` | CONFIG | Apply fabric-wide network tuning (buffer/block sizes) to live tunnels. | `region`, `agent`, `pluginid` | `status` |
| `removesrctunnel` | CONFIG | Tear down the source (listener) side of a tunnel. | `region`, `agent`, `pluginid`, `action_stunnel_id` | `status` |
| `removedsttunnel` | CONFIG | Tear down the destination side of a tunnel. | `region`, `agent`, `pluginid`, `action_stunnel_id` | `status` |
| `tunnelhealthcheck` | EXEC | Check whether a tunnel config exists on this node. | `region`, `agent`, `pluginid`, `action_stunnel_id` | `status` |

### wsapi plugin

WebSocket API / dataplane ingress plugin ([`wsapi`](../plugins/wsapi.md)) — the bus external clients
connect to. `target=plugin` (`region`, `agent`, `pluginid`). Namespace `wsapi` — **4 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `getcapabilities` | EXEC | Return this plugin's self-describing capability document. | `region`, `agent`, `pluginid` | `capabilities` |
| `getmetrics` | EXEC | Return wsapi dataplane metrics (connections, bytes, messages). | `region`, `agent`, `pluginid` | `metrics` |
| `globalinfo` | EXEC | Return the global controller's region and agent identity. | `region`, `agent`, `pluginid` | `global_region`, `global_agent` |
| `nettuning` | CONFIG | Apply fabric-wide network tuning to the live wss server. | `region`, `agent`, `pluginid` | `status` |

### repo plugin

Plugin-artifact repository ([`repo`](../plugins/repo.md)) — stores and serves plugin JARs. `target=plugin`
(`region`, `agent`, `pluginid`). Namespace `repo` — **5 actions**.

| Action | Type | Description | Key params | Returns |
|---|---|---|---|---|
| `getcapabilities` | EXEC | Return this plugin's self-describing capability document. | `region`, `agent`, `pluginid` | `capabilities` |
| `getjar` | EXEC | Retrieve a plugin JAR's bytes by name + md5. | `region`, `agent`, `pluginid`, `action_pluginname`, `action_pluginmd5` | `jardata` |
| `getmetrics` | EXEC | Return repo inventory metrics (plugin count, bytes on disk). | `region`, `agent`, `pluginid` | `metrics` |
| `putjar` | EXEC | Store a plugin JAR into the repo (writes disk, verifies md5). | `region`, `agent`, `pluginid`, `pluginname`, `md5`, `jarfile`, `version`, `jardata` | `uploaded`, `md5-confirm` |
| `repolist` | EXEC | List every plugin JAR plus the repo's server identity. | `region`, `agent`, `pluginid` | `repolist` *(compressed)* |

---

## Notes

- **Every plugin/tier exposes `getcapabilities`** (its own document) and the controller tiers additionally
  expose `getcapabilityinventory` (the aggregate). Every plugin also exposes `getmetrics`; see
  [Metrics & Measurements](../architecture/metrics.md).
- **Destructive actions** (`killjvm`, `stopcontroller`, `restartcontroller`, `restartframework`, plugin
  removal, tunnel teardown) mutate node state — invoke deliberately.
- The exhaustive, auto-generated per-action detail (full parameter descriptions, chain-of-events, and
  captured live responses) lives in `docs/cresco-tools-reference.md` and the capability design doc
  `docs/capability-inventory-design.md`; **this page is the canonical human reference** distilled from them.

## See also

- [MsgEvent](msgevent.md) — the envelope every action travels in.
- [Messaging & Routing](../architecture/messaging.md) — how an action reaches its `target`.
- [Library API](library-api.md) — the plugin authoring API (`Executor`, `PluginBuilder`).
- [Java client (clientlib)](../clients/java.md) · [Python client (pycrescolib)](../clients/python.md) —
  building and sending action messages, including `get_capability_inventory()`.
