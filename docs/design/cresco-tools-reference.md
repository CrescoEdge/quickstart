# Cresco Tools — Exhaustive Reference

> Generated from a live 3-node mesh by `run/tests/tool_suite.py` on 2026-07-03. 89 tools; 49 exercised live with real captured responses; the remainder (mutating/destructive/artifact-dependent) are documented with request shape + chain of events.

## 1. What a "tool" is

Every Cresco capability is a **message action** — a `MsgEvent` of a given `Type` (`EXEC`, `CONFIG`, …)
carrying an `action` parameter. An external client reaches Cresco only over the **WebSocket message bus**
(the `wsapi` plugin), so **every tool is a MsgEvent action** — there are no direct method calls. Each tool
here has a self-describing entry in the fabric's capability inventory (`getcapabilityinventory`), which is
directly consumable as an LLM tool definition (`name` / `description` / `input_schema`) plus a
`cresco_binding` block telling a runner how to build and route the MsgEvent.

**Tool name:** `cresco_<namespace>_<action>` (e.g. `cresco_global_listagents`).

## 2. Routing tiers (the `target`) and how a tool call becomes a MsgEvent

| target | reached with | handled by |
|---|---|---|
| `global`   | `global_controller_msgevent(is_rpc, type, payload)` | GlobalExecutor on the global controller |
| `agent`    | `global_agent_msgevent(is_rpc, type, payload, region, agent)` | AgentExecutor on that agent |
| `regional` | (region controller) — usually via the global fan-out | RegionalExecutor on a region controller |
| `plugin`   | `global_plugin_msgevent(is_rpc, type, payload, region, agent, pluginid)` | that plugin's Executor |

A tool call `{name, input}` becomes a MsgEvent by: taking the `cresco_binding` (`msg_type`, `target`,
`action`), pulling the routing identity (`region`/`agent`/`pluginid`) out of `input`, and putting the
rest of `input` as action params. The reply is a MsgEvent whose params carry the result.

## 3. Scope: global vs regional are NOT duplicates

Several actions exist on both the **global** and **regional** tiers with the same name. They differ by
**scope**: the **global** tier aggregates across **every region it sees** (a fabric-wide report); the
**regional** tier reports a **single region**. E.g. `listregions`/`getmetricinventory scope=global` cover
the whole mesh, while a region controller answers only for its own region. Treat the tier as part of the
tool's identity.

## 4. Replies and compressed params

RPC replies are `MsgEvent` params. Large/structured values are **gzip+base64 compressed** (marked
*compressed* in the Returns tables) — read them with `decompress_param()` + `json_deserialize()` (Python)
or `getCompressedParam()` (Java). A `status` of `10` conventionally means success; `9` failure.

## 5. How the examples were captured

`run/tests/tool_suite.sh` brings up a global + region + agent mesh (with the sysinfo/stunnel/wsapi/repo
plugins auto-loaded on the global), then `tool_suite.py` invokes every read/safe tool via the MCP
tool-runner and records the real request + response to `/tmp/tool-results.json`. Destructive tools
(`killjvm`, `stopcontroller`, …) and artifact-dependent tools (needing a real JAR or pipeline) are
documented from their descriptor + source, not invoked live.

---

## 6. Tools by namespace

- **global** — 32 tools
- **agent** — 26 tools
- **regional** — 6 tools
- **sysinfo** — 4 tools
- **stunnel** — 12 tools
- **wsapi** — 4 tools
- **repo** — 5 tools


---

## global tools

**Global tier** — routed to the **global controller** (`global_controller_msgevent`). These are the fabric-wide control API: registry, discovery, resource inventory, and global application (pipeline) scheduling. **Global-tier reports aggregate across every region the global sees** — e.g. `listregions`/`listagents`/`getmetricinventory scope=global` return the whole fabric, not one region.

### `cresco_global_addplugin`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `addplugin` · **Category:** artifact

Schedule an iNode (plugin instance) for deployment.

**Chain of events / interacts with:** Schedules an iNode: getGDB().addINode() with status 0 (scheduled). The global scheduler (PollAddPipeline/ResourceScheduler) later emits a CONFIG pluginadd to the chosen agent, which drives AgentExecutor → PluginAdmin.addPlugin() (OSGi bundle install+start).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `inode_id` | string | yes |  |
| `resource_id` | string | yes |  |
| `configparams` | object | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |
| `status_desc` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_addplugin",
  "input": {
    "inode_id": "\u2026",
    "resource_id": "\u2026",
    "configparams": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "addplugin"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getcapabilities`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return the global controller's self-describing capability document (its actions as LLM tool specs).

**Chain of events / interacts with:** CapabilityResponder scans GlobalExecutor's @CrescoAction annotations (bundle name+version from the OSGi manifest) → the global tier's own tool document.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_getcapabilities",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getcapabilities"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"global\",\"bundle\":\"io.cresco.controller\",\"version\":\"1.3.0.SNAPSHOT-20260703161448\",\"summary\":\"Global controller: fabric-wide registry, discovery, resource inventory, and global application (pipeline) scheduling. The entry point for cross-region orchestration.\",\"actions\":[{\"namespace\":\"global\",\"action\":\"region_enable\",\"msgType\":\"CONFIG\",\"summary\":\"Register a region in the global registry.\",\"why\":\"Called when a region federates with the global; makes it visible to placement.\",\"target\":\"global\",\"routingParams\":[],\"params\":[],\"returns\":[{\"name\":\"is_registered\",\"type\":\"boolean\",\"requir\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_global_getcapabilityinventory`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getcapabilityinventory` · **Category:** read

Return the fabric-wide capability inventory: controller tiers + every plugin's actions + OSGi surface, as one LLM tool catalog.

**Chain of events / interacts with:** CapabilityInventory: statically scans the three controller-tier executor classes, RPCs each local plugin's getcapabilities, and scans the OSGi surface. scope=global fans out to every region+agent → one fabric-wide LLM tool catalog.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_scope` | string | no | node|region|global |
| `action_include_plugins` | boolean | no |  |
| `action_include_osgi` | boolean | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilityinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_getcapabilityinventory",
  "input": {
    "action_scope": "\u2026",
    "action_include_plugins": "\u2026",
    "action_include_osgi": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getcapabilityinventory"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilityinventory": "{\"node\":\"global-region_global-controller\",\"capabilities_by_source\":{\"global-region_global-controller:io.cresco.agent.controller:agent\":{\"namespace\":\"agent\",\"bundle\":\"io.cresco.controller\",\"version\":\"1.3.0.SNAPSHOT-20260703161448\",\"summary\":\"Agent controller: manages plugins on a single agent node, controller lifecycle, log/file access, CEP rules, and agent-local queries.\",\"actions\":[{\"namespace\":\"agent\",\"action\":\"pluginadd\",\"msgType\":\"CONFIG\",\"summary\":\"Add/start a plugin bundle on this agent.\",\"why\":\"Deploy a plugin locally.\",\"target\":\"agent\",\"routingParams\":[\"region\",\"agent\"],\"params\":[{\"nam\u2026(truncated)",
  "action": "getcapabilityinventory",
  "status": "10"
}
```


### `cresco_global_getenvstatus`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getenvstatus` · **Category:** artifact

Count nodes matching an environment index key=value.

**Chain of events / interacts with:** Counts nodes whose environment index matches key=value — query fabric composition by attribute.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `environment_id` | string | yes |  |
| `environment_value` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `count` | integer | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_getenvstatus",
  "input": {
    "environment_id": "\u2026",
    "environment_value": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getenvstatus"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getgpipeline`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getgpipeline` · **Category:** artifact

Get a pipeline definition by id.

**Chain of events / interacts with:** getGDB().getGPipeline(id) — a pipeline (distributed application) definition.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_pipelineid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `gpipeline` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_getgpipeline",
  "input": {
    "action_pipelineid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getgpipeline"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getgpipelineexport`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getgpipelineexport` · **Category:** artifact

Get a pipeline in export (portable) format.

**Chain of events / interacts with:** getGDB().getGPipelineExport(id) — the pipeline in portable export format for re-deployment elsewhere.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_pipelineid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `gpipeline` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_getgpipelineexport",
  "input": {
    "action_pipelineid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getgpipelineexport"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getgpipelinestatus`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getgpipelinestatus` · **Category:** read

Get status of one pipeline or all pipelines.

**Chain of events / interacts with:** getGDB().getPipelineInfo() — the status of one pipeline (or all pipelines if none specified).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_pipeline` | string | no | pipeline id; omit for all |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `pipelineinfo` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_getgpipelinestatus",
  "input": {
    "action_pipeline": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getgpipelinestatus"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getgpipelinestatus",
  "pipelineinfo": {
    "pipelines": []
  }
}
```


### `cresco_global_getinodestatus`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getinodestatus` · **Category:** artifact

Get an iNode (plugin instance) status map from the DB.

**Chain of events / interacts with:** getGDB().getInodeMap(inode_id) — the status map of one iNode (plugin instance).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `inode_id` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `inodemap` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_getinodestatus",
  "input": {
    "inode_id": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getinodestatus"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getisassignmentinfo`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getisassignmentinfo` · **Category:** artifact

Check assignment info for an iNode/resource.

**Chain of events / interacts with:** getGDB().getIsAssignedInfo(inode,resource) — placement diagnostics for an iNode/resource pair.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_inodeid` | string | yes |  |
| `action_resourceid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `isassignmentinfo` | string | yes |  |
| `isassignmentresourceinfo` | string | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_getisassignmentinfo",
  "input": {
    "action_inodeid": "\u2026",
    "action_resourceid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getisassignmentinfo"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_getmetricinventory`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `getmetricinventory` · **Category:** read

Return the unified metric inventory (controller + all plugins + resource summary).

**Chain of events / interacts with:** Delegates to PerfControllerMonitor.getMetricInventory(scope). scope=global fans out CONCURRENTLY to every region+agent, merging each node's controller Micrometer groups + every plugin's getmetrics + the sysinfo resource summary into one document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_scope` | string | no | node|region|global |
| `action_include_plugins` | boolean | no |  |
| `action_include_resource` | boolean | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metricinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_getmetricinventory",
  "input": {
    "action_scope": "\u2026",
    "action_include_plugins": "\u2026",
    "action_include_resource": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "getmetricinventory"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "metricinventory": "{\"node\":\"global-region_global-controller\",\"metrics_by_source\":{\"global-region_global-controller:io.cresco.agent.controller\":{\"jvm\":[{\"name\":\"jvm.memory.max\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"-1.0\",\"group\":\"jvm\"},{\"name\":\"jvm.memory.used\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"8172848.0\",\"group\":\"jvm\"},{\"name\":\"jvm.classes.loaded\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"12290.0\",\"group\":\"jvm\"},{\"name\":\"jvm.classes.unloaded\",\"count\":\"1.0\",\"type\":\"NODE\",\"class\":\"COUNTER\",\"group\":\"jvm\"},{\"name\":\"jvm.buffer.memory.used\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"8047351.0\",\"group\":\"jvm\"},{\"name\":\"j\u2026(truncated)",
  "action": "getmetricinventory",
  "status": "10"
}
```


### `cresco_global_gpipelineremove`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `gpipelineremove` · **Category:** artifact

Remove/undeploy a global pipeline by id.

**Chain of events / interacts with:** Async undeploy via the PollRemovePipeline executor — tears down every iNode of the pipeline across all agents that host it.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_pipelineid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `success` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_gpipelineremove",
  "input": {
    "action_pipelineid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "gpipelineremove"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_gpipelinesubmit`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `gpipelinesubmit` · **Category:** artifact

Submit a global application pipeline (multi-plugin app) for scheduling.

**Chain of events / interacts with:** The main application-deploy path. GlobalExecutor creates a pipeline record (getGDB().createPipelineRecord) and queues it to AppScheduler, which resolves placement (buildNodeMaps) and fans out addplugin CONFIGs to agents across regions. Returns the assigned pipeline id. Interacts with: global DB, AppScheduler, every target agent's PluginAdmin.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_gpipeline` | object | yes | compressed JSON pipeline (CADL) |
| `action_tenantid` | string | no | tenant id |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `gpipeline_id` | string | no | assigned pipeline id |

**Tool call:**

```json
{
  "name": "cresco_global_gpipelinesubmit",
  "input": {
    "action_gpipeline": "\u2026",
    "action_tenantid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "gpipelinesubmit"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_listagents`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listagents` · **Category:** read

List agents in the fabric, optionally filtered by region.

**Chain of events / interacts with:** getGDB().getAgentList(region) reads agents from the global DB; for a non-local region it RPCs that region. GLOBAL scope = every agent in every region.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_region` | string | no | region filter; omit for all |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `agentslist` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listagents",
  "input": {
    "action_region": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listagents"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "agentslist": {
    "agents": [
      {
        "environment": "Mac OS X",
        "agent_id": "global-controller",
        "status_desc": "{\"mode\":\"GLOBAL\"}",
        "plugins": "4",
        "region_id": "global-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      },
      {
        "environment": "Mac OS X",
        "agent_id": "edge-controller",
        "status_desc": "isRegionGlobal() Success",
        "plugins": "2",
        "region_id": "edge-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      },
      {
        "environment": "Mac OS X",
        "agent_id": "edge-agent-001",
        "status_desc": "{\"mode\":\"AGENT\",\"broker\":\"127.0.0.1\"}",
        "plugins": "2",
        "region_id": "edge-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      }
    ]
  },
  "action": "listagents"
}
```


### `cresco_global_listplugins`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listplugins` · **Category:** read

List plugin instances, optionally filtered by region/agent.

**Chain of events / interacts with:** getGDB().getPluginList(region,agent) reads plugin instances from the global DB, optionally filtered by region/agent.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_region` | string | no |  |
| `action_agent` | string | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `pluginslist` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listplugins",
  "input": {
    "action_region": "\u2026",
    "action_agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listplugins"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_agent": "global-controller",
  "action_region": "global-region",
  "action": "listplugins",
  "pluginslist": {
    "plugins": [
      {
        "jarstatus": "embedded",
        "agent": "global-controller",
        "persistence_code": "20",
        "pluginname": "io.cresco.repo",
        "jarfile": "repo.jar",
        "name": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e",
        "region": "global-region",
        "inode_id": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e",
        "version": "1.3.0.SNAPSHOT-20260703160835",
        "md5": "f95a7af8541064aecdb456b7f956b934"
      },
      {
        "jarstatus": "embedded",
        "agent": "global-controller",
        "persistence_code": "20",
        "pluginname": "io.cresco.wsapi",
        "jarfile": "wsapi.jar",
        "name": "system-5ed6fdf9-5843-447d-b641-e28bb3a63f88",
        "region": "global-region",
        "inode_id": "system-5ed6fdf9-5843-447d-b641-e28bb3a63f88",
        "version": "1.3.0.SNAPSHOT-20260703160837",
        "md5": "b691dc0ab7414c1016c00cf3077d5721"
      },
      {
        "jarstatus": "embedded",
        "agent": "global-controller",
        "persistence_code": "20",
        "pluginname": "io.cresco.stunnel",
        "jarfile": "stunnel.jar",
        "name": "system-9a585863-7e2a-4476-a12b-17086df51b9c",
        "region": "global-region",
        "inode_id": "system-9a585863-7e2a-4476-a12
… (truncated)
```


### `cresco_global_listpluginsbytype`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listpluginsbytype` · **Category:** read

List plugin instances matching a key=value (e.g. pluginname=io.cresco.repo).

**Chain of events / interacts with:** getGDB().getPluginListByType(key,value) — every plugin instance matching e.g. pluginname=io.cresco.repo, across the fabric.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_plugintype_id` | string | yes |  |
| `action_plugintype_value` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `pluginsbytypelist` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listpluginsbytype",
  "input": {
    "action_plugintype_id": "\u2026",
    "action_plugintype_value": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listpluginsbytype"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "pluginsbytypelist": {
    "plugins": [
      {
        "jarstatus": "embedded",
        "persistence_code": "20",
        "pluginname": "io.cresco.sysinfo",
        "jarfile": "sysinfo.jar",
        "inode_id": "system-9eff73cc-30c3-490e-b15f-cedfe5a982e4",
        "version": "1.3.0.SNAPSHOT-20260703160832",
        "md5": "af12a3b09bd249278482d6df0cb3fa77",
        "region": "edge-region",
        "agent": "edge-agent-001",
        "pluginid": "system-9eff73cc-30c3-490e-b15f-cedfe5a982e4"
      },
      {
        "jarstatus": "embedded",
        "persistence_code": "20",
        "pluginname": "io.cresco.sysinfo",
        "jarfile": "sysinfo.jar",
        "inode_id": "system-de9f199b-a232-4587-9b14-105e8a22b162",
        "version": "1.3.0.SNAPSHOT-20260703160832",
        "md5": "af12a3b09bd249278482d6df0cb3fa77",
        "region": "edge-region",
        "agent": "edge-controller",
        "pluginid": "system-de9f199b-a232-4587-9b14-105e8a22b162"
      },
      {
        "jarstatus": "embedded",
        "persistence_code": "20",
        "pluginname": "io.cresco.sysinfo",
        "jarfile": "sysinfo.jar",
        "inode_id": "system-bf2a5471-fa3a-4adb-8146-62ed39350b0e",
        "version": "1.3.0.SNAPSHOT-20260703160832",
        "md5": "af12a3b09bd249278482d6df0cb3fa77",
        "region": "global-region",
        "agent": "global-controller",
        "pluginid": "system-bf
… (truncated)
```


### `cresco_global_listpluginsrepo`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listpluginsrepo` · **Category:** read

List all plugins available across the fabric's repos.

**Chain of events / interacts with:** getGDB().getPluginListRepo() — the union of plugin artifacts available across all repos.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `listpluginsrepo` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listpluginsrepo",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listpluginsrepo"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "listpluginsrepo",
  "listpluginsrepo": {
    "plugins": []
  }
}
```


### `cresco_global_listregions`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listregions` · **Category:** read

List all regions in the fabric plus live inter-broker bridge info.

**Chain of events / interacts with:** GlobalExecutor reads the global DB region list AND queries live inter-broker bridges (getConnectedRegions over the ActiveMQ NetworkConnectors). Reports BOTH registered regions and live bridge topology — this is the fabric-wide view of every region the global sees.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `regionslist` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listregions",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listregions"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "regionslist": {
    "regions": [
      {
        "name": "global-region",
        "type": "local",
        "agents": "1"
      },
      {
        "name": "edge-region",
        "type": "local",
        "agents": "2"
      },
      {
        "bridged_agent": "edge-agent-001",
        "name": "edge-region",
        "type": "bridged",
        "agents": "1"
      },
      {
        "bridged_agent": "edge-controller",
        "name": "edge-region",
        "type": "bridged",
        "agents": "2"
      }
    ]
  },
  "action": "listregions"
}
```


### `cresco_global_listrepoinstances`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `listrepoinstances` · **Category:** read

List all repo plugin instances in the fabric.

**Chain of events / interacts with:** getGDB().getPluginListByType('pluginname','io.cresco.repo') — locates every repo server in the fabric.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `listrepoinstances` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_listrepoinstances",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "listrepoinstances"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "listrepoinstances": {
    "plugins": [
      {
        "jarstatus": "embedded",
        "persistence_code": "20",
        "pluginname": "io.cresco.repo",
        "jarfile": "repo.jar",
        "inode_id": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e",
        "version": "1.3.0.SNAPSHOT-20260703160835",
        "md5": "f95a7af8541064aecdb456b7f956b934",
        "region": "global-region",
        "agent": "global-controller",
        "pluginid": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e"
      }
    ]
  },
  "action": "listrepoinstances"
}
```


### `cresco_global_netresourceinfo`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `netresourceinfo` · **Category:** read

Get network-wide resource totals.

**Chain of events / interacts with:** getGDB().getNetResourceInfo() — network-wide resource totals from the global DB.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `netresourceinfo` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_netresourceinfo",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "netresourceinfo"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "netresourceinfo"
}
```


### `cresco_global_ping`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `ping` · **Category:** read

Liveness ping; replies pong and exchanges mesh health.

**Chain of events / interacts with:** pingReply(): records the region's advertised rolled-up mesh health (MeshHealthPing.recordChild), stamps the global's own health back onto the reply, and returns pong + remote_ts. This is the region→global liveness/RTT link.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `action` | string | no | pong |

**Tool call:**

```json
{
  "name": "cresco_global_ping",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "ping"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "pong",
  "parent_health": "OK",
  "remote_ts": "1783097555593",
  "type": "global_controller"
}
```


### `cresco_global_plugindownload`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `plugindownload` · **Category:** artifact

Download a plugin JAR from a URL into the repo (cache-checked).

**Chain of events / interacts with:** Fetches a plugin JAR from a URL into the local repo (cache-checked unless forced). Interacts with the io.cresco.repo plugin and the HTTP source.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `pluginurl` | string | yes |  |
| `agentcontroller` | string | yes |  |
| `forceplugindownload` | boolean | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `hasplugin` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_plugindownload",
  "input": {
    "pluginurl": "\u2026",
    "agentcontroller": "\u2026",
    "forceplugindownload": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "plugindownload"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_plugininfo`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `plugininfo` · **Category:** read

Get metadata for a specific plugin instance.

**Chain of events / interacts with:** getGDB().getPluginInfo(region,agent,plugin) — one plugin instance's full metadata/config.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_region` | string | yes |  |
| `action_agent` | string | yes |  |
| `action_plugin` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `plugininfo` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_plugininfo",
  "input": {
    "action_region": "\u2026",
    "action_agent": "\u2026",
    "action_plugin": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "plugininfo"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_agent": "global-controller",
  "action_plugin": "system-bf2a5471-fa3a-4adb-8146-62ed39350b0e",
  "action_region": "global-region",
  "action": "plugininfo",
  "plugininfo": {
    "jarstatus": "embedded",
    "persistence_code": "20",
    "pluginname": "io.cresco.sysinfo",
    "jarfile": "sysinfo.jar",
    "inode_id": "system-bf2a5471-fa3a-4adb-8146-62ed39350b0e",
    "version": "1.3.0.SNAPSHOT-20260703160832",
    "md5": "af12a3b09bd249278482d6df0cb3fa77"
  }
}
```


### `cresco_global_plugininventory`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `plugininventory` · **Category:** read

List plugins in the local repo jar directory (name=version).

**Chain of events / interacts with:** Scans the global node's local repo JAR directory (reads manifests) → a name=version list of artifacts held locally.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `pluginlist` | string | no | name=version CSV |

**Tool call:**

```json
{
  "name": "cresco_global_plugininventory",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "plugininventory"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "plugininventory"
}
```


### `cresco_global_pluginkpi`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `pluginkpi` · **Category:** read

Get the metrics attached to a specific plugin instance.

**Chain of events / interacts with:** getPerfControllerMonitor().getIsAttachedMetrics(...) — the metrics attached to a specific plugin instance.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_region` | string | yes |  |
| `action_agent` | string | yes |  |
| `action_plugin` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `pluginkpi` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_pluginkpi",
  "input": {
    "action_region": "\u2026",
    "action_agent": "\u2026",
    "action_plugin": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "pluginkpi"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_agent": "global-controller",
  "action_plugin": "system-bf2a5471-fa3a-4adb-8146-62ed39350b0e",
  "action_region": "global-region",
  "pluginkpi": [
    {
      "name": "controller_group",
      "metrics": "{\"controller\":[{\"totaltime\":\"1365.028045\",\"max\":\"955.453583\",\"mean\":\"13.515129158415842\",\"name\":\"message.transaction.time\",\"count\":\"101\",\"type\":\"NODE\",\"class\":\"TIMER\"}]}"
    }
  ],
  "action": "pluginkpi"
}
```


### `cresco_global_region_disable`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `region_disable` · **Category:** artifact

Unregister a region from the global registry.

**Chain of events / interacts with:** GlobalExecutor → getGDB().removeNode() removes the region from the global DB. Sent on graceful region shutdown so the fabric view stays accurate.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_unregistered` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_region_disable",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "region_disable"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_region_enable`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `region_enable` · **Category:** artifact

Register a region in the global registry.

**Chain of events / interacts with:** GlobalExecutor.executeCONFIG → getGDB().nodeUpdate() registers the region node in the Derby global DB and clears stale region/agent configs. Sent automatically when a region federates; makes the region visible to placement and to listregions.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_registered` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_region_enable",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "region_enable"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_regionalimport`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `regionalimport` · **Category:** artifact

Register/import a regional node into the global DB.

**Chain of events / interacts with:** Internal region onboarding — registers a regional node (mode=REGION) via getGDB().nodeUpdate().

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_global_regionalimport",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "regionalimport"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_removeplugin`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `removeplugin` · **Category:** artifact

Schedule an iNode (plugin instance) for removal.

**Chain of events / interacts with:** Marks an iNode 'scheduled for removal'; the scheduler drives PollRemovePipeline → a CONFIG pluginremove on the agent → PluginAdmin stops+uninstalls the bundle.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `inode_id` | string | yes |  |
| `resource_id` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |
| `status_desc` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_removeplugin",
  "input": {
    "inode_id": "\u2026",
    "resource_id": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "removeplugin"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_resourceinfo`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `resourceinfo` · **Category:** read

Get aggregated resource info (cpu/mem/disk) for a region/agent or the whole fabric.

**Chain of events / interacts with:** getPerfControllerMonitor().getResourceInfo(region,agent) → RPCs each agent's io.cresco.sysinfo getsysinfo and aggregates cpu/mem/disk. GLOBAL scope aggregates across every agent it can reach.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_region` | string | no |  |
| `action_agent` | string | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `resourceinfo` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_global_resourceinfo",
  "input": {
    "action_region": "\u2026",
    "action_agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "resourceinfo"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_agent": "global-controller",
  "action_region": "global-region",
  "action": "resourceinfo",
  "resourceinfo": {
    "agentresourceinfo": [
      {
        "perf": "{\"disk\":[{\"disk-writebytes\":\"1970096304128\",\"disk-writes\":\"123529943\",\"disk-size\":\"1000555581440\",\"disk-model\":\"APPLE SSD AP1024Z\",\"disk-readbytes\":\"2698514079744\",\"disk-name\":\"disk0\",\"disk-transfertime\":\"39997958\",\"disk-reads\":\"156439192\"},{\"disk-writebytes\":\"280150016\",\"disk-writes\":\"26966\",\"disk-size\":\"524288000\",\"disk-model\":\"APPLE SSD AP1024Z\",\"disk-readbytes\":\"119095296\",\"disk-name\":\"disk1\",\"disk-transfertime\":\"0\",\"disk-reads\":\"27771\"},{\"disk-writebytes\":\"1968553652224\",\"disk-writes\":\"123501422\",\"disk-size\":\"994662584320\",\"disk-model\":\"APPLE SSD AP1024Z\",\"disk-readbytes\":\"2698391785472\",\"disk-name\":\"disk3\",\"disk-transfertime\":\"0\",\"disk-reads\":\"156411066\"},{\"disk-writebytes\":\"1262501888\",\"disk-writes\":\"1555\",\"disk-size\":\"5368664064\",\"disk-model\":\"APPLE SSD AP1024Z\",\"disk-readbytes\":\"3121152\",\"disk-name\":\"disk2\",\"disk-transfertime\":\"0\",\"disk-reads\":\"342\"}],\"os\":[{\"sys-manufacturer\":\"Apple\",\"process-count\":\"880\",\"sys-uptime\":\"2860497\",\"sys-family\":\"macOS\",\"sys-os\":\"26.2 build 25C56\"}],\"mem\":[{\"memory-total\":\"38654705664\",\"swap-total\":\"0\",\"sw
… (truncated)
```


### `cresco_global_resourceinventory`

**Tier:** global · **MsgEvent:** EXEC · **Action:** `resourceinventory` · **Category:** read

Get fabric resource totals.

**Chain of events / interacts with:** getGDB().getResourceTotal() — high-level fabric resource totals.

**Parameters:**

_none_

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `resourceinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_resourceinventory",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "EXEC", {"action": "resourceinventory"})
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "resourceinventory"
}
```


### `cresco_global_savetorepo`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `savetorepo` · **Category:** artifact

Save/replicate a plugin JAR to all repo instances.

**Chain of events / interacts with:** Broadcasts the JAR to EVERY repo instance in the fabric via RPC putjar (interacts with all io.cresco.repo plugins) so the artifact is replicated fabric-wide.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `configparams` | object | yes | {pluginname,md5,version} |
| `jardata` | binary | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_saved` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_savetorepo",
  "input": {
    "configparams": "\u2026",
    "jardata": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "savetorepo"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_global_setinodestatus`

**Tier:** global · **MsgEvent:** CONFIG · **Action:** `setinodestatus` · **Category:** artifact

Set the status of an iNode (plugin instance) in the DB.

**Chain of events / interacts with:** Direct DB write: getGDB().setINodeParam() twice to set an iNode's status code + description. Lifecycle bookkeeping for a deployed plugin instance.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `action_inodeid` | string | yes |  |
| `action_statuscode` | string | yes |  |
| `action_statusdesc` | string | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `success` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_global_setinodestatus",
  "input": {
    "action_inodeid": "\u2026",
    "action_statuscode": "\u2026",
    "action_statusdesc": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_controller_msgevent(True, "CONFIG", {"action": "setinodestatus"})
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.



---

## agent tools

**Agent tier** — routed to a specific **agent controller** (`global_agent_msgevent` with region+agent). Agent-local operations: manage plugins on that node, controller lifecycle, logs/files, CEP rules, and node-scoped queries.

### `cresco_agent_cepadd`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `cepadd` · **Category:** artifact

Add a Complex-Event-Processing rule to the dataplane.

**Chain of events / interacts with:** DataPlaneService.createCEP() installs a Siddhi Complex-Event-Processing query over the named dataplane input stream, emitting to an output stream. Returns the cepid.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `cepparams` | object | yes | {input_stream,output_stream,query,...} |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `cepid` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_cepadd",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "cepparams": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "cepadd", "cepparams": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_controllerupdate`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `controllerupdate` · **Category:** destructive

Stage a controller JAR for the next restart.

**Chain of events / interacts with:** Stages a controller JAR for the next restart by writing conf/version.ini; the new JAR is swapped in on restartcontroller.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `jar_file_path` | string | yes |  |

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_agent_controllerupdate",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "jar_file_path": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "controllerupdate", "jar_file_path": ...}, region, agent)
```

**Not exercised live** — destructive (would tear down the node). Effect: see chain above.


### `cresco_agent_getagentinfo`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `getagentinfo` · **Category:** read

Return the agent's data-directory path.

**Chain of events / interacts with:** Returns the agent's cresco_data_location (on-disk data directory) from config.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `agent-data` | string | no | data directory path |

**Tool call:**

```json
{
  "name": "cresco_agent_getagentinfo",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "getagentinfo"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getagentinfo",
  "agent-data": "cresco-data"
}
```


### `cresco_agent_getbroadcastdiscovery`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getbroadcastdiscovery` · **Category:** read

Return this agent's network discovery list.

**Chain of events / interacts with:** Triggers a UDP/TCP broker discovery scan (PerfMonitorNet Broker Search) and returns the discovered list. SLOW (~10s) — the scan blocks; give the call a generous timeout.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `broadcast_discovery` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getbroadcastdiscovery",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getbroadcastdiscovery"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "broadcast_discovery": "data",
  "action": "getbroadcastdiscovery"
}
```


### `cresco_agent_getcapabilities`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return the agent controller's self-describing capability document.

**Chain of events / interacts with:** Scans AgentExecutor's annotations → the agent tier's own tool document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"agent\",\"bundle\":\"io.cresco.controller\",\"version\":\"1.3.0.SNAPSHOT-20260703161448\",\"summary\":\"Agent controller: manages plugins on a single agent node, controller lifecycle, log/file access, CEP rules, and agent-local queries.\",\"actions\":[{\"namespace\":\"agent\",\"action\":\"pluginadd\",\"msgType\":\"CONFIG\",\"summary\":\"Add/start a plugin bundle on this agent.\",\"why\":\"Deploy a plugin locally.\",\"target\":\"agent\",\"routingParams\":[\"region\",\"agent\"],\"params\":[{\"name\":\"configparams\",\"type\":\"object\",\"required\":true,\"description\":\"\",\"compressed\":true},{\"name\":\"edges\",\"type\":\"object\",\"required\":false,\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_agent_getcapabilityinventory`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getcapabilityinventory` · **Category:** read

Return this node's capability inventory (node scope): controller tiers + local plugins + OSGi surface.

**Chain of events / interacts with:** This node's node-scoped capability catalog (controller tiers + local plugins + OSGi). The global/region fan-out calls this on each agent.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `action_include_plugins` | boolean | no |  |
| `action_include_osgi` | boolean | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilityinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getcapabilityinventory",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "action_include_plugins": "\u2026",
    "action_include_osgi": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getcapabilityinventory", "action_include_plugins": ..., "action_include_osgi": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilityinventory": "{\"node\":\"global-region_global-controller\",\"capabilities_by_source\":{\"global-region_global-controller:io.cresco.agent.controller:agent\":{\"namespace\":\"agent\",\"bundle\":\"io.cresco.controller\",\"version\":\"1.3.0.SNAPSHOT-20260703161448\",\"summary\":\"Agent controller: manages plugins on a single agent node, controller lifecycle, log/file access, CEP rules, and agent-local queries.\",\"actions\":[{\"namespace\":\"agent\",\"action\":\"pluginadd\",\"msgType\":\"CONFIG\",\"summary\":\"Add/start a plugin bundle on this agent.\",\"why\":\"Deploy a plugin locally.\",\"target\":\"agent\",\"routingParams\":[\"region\",\"agent\"],\"params\":[{\"nam\u2026(truncated)",
  "action": "getcapabilityinventory",
  "status": "10"
}
```


### `cresco_agent_getcontrollerstatus`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getcontrollerstatus` · **Category:** read

Return this agent's controller state code.

**Chain of events / interacts with:** Reads the ControllerState FSM → the integer state code.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `controller_status` | integer | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getcontrollerstatus",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getcontrollerstatus"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "controller_status": "GLOBAL",
  "action": "getcontrollerstatus"
}
```


### `cresco_agent_getfiledata`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getfiledata` · **Category:** artifact

Stream a chunk of a file on this agent (seek+read).

**Chain of events / interacts with:** Seeks skiplength and reads partsize bytes of a file → a compressed payload chunk. Used to stream large files in parts.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `filepath` | string | yes |  |
| `skiplength` | integer | yes |  |
| `partsize` | integer | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `payload` | binary | yes |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getfiledata",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "filepath": "\u2026",
    "skiplength": "\u2026",
    "partsize": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getfiledata", "filepath": ..., "skiplength": ..., "partsize": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_getfileinfo`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getfileinfo` · **Category:** artifact

Return metadata (md5, size) for a file on this agent.

**Chain of events / interacts with:** Returns md5 + size for a file on the agent (status 10=found / 9=not found) — the first step of a chunked file transfer.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `filepath` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `md5` | string | no |  |
| `size` | integer | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getfileinfo",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "filepath": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getfileinfo", "filepath": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_getislogdp`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `getislogdp` · **Category:** read

Query whether dataplane log streaming is enabled for a session.

**Chain of events / interacts with:** Queries whether data-plane log streaming is enabled for a session (DataPlaneLogger).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `session_id` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `islogdp` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getislogdp",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "session_id": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "getislogdp", "session_id": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "status_code": "7",
  "status_desc": "islogDP Get",
  "islogdp": "false",
  "action": "getislogdp",
  "session_id": "suite-sess"
}
```


### `cresco_agent_getlog`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getlog` · **Category:** read

Fetch this agent's main log (compressed inline or as a file).

**Chain of events / interacts with:** Reads cresco-data/cresco-logs/main.log — compressed inline when action_inmessage=true, else attached as a file for chunked transfer.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `action_inmessage` | boolean | no | true=compress into reply; else attach file |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `log` | binary | yes |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getlog",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "action_inmessage": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getlog", "action_inmessage": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "log": "H4sIAAAAAAAA/81bbW/iSBL+vr+i7sNpiRaMzavhxEmEYWeyIS8XMjs6RShq7AZ8MbbXbSfhND/+qruNMWDnxTHhUDTYZdP19FPV9dL2qHX4I7ShptZaoNW6Ta1ba5cbzTbA2eXvV3C3JJYzgQpMQ8e0Kbj+XCEeMRZUmVHbelaY4XdrSk3RalDSGicwDogfWM4cnqxgAXPbnRJ7+BxQx6Q+MBrwa12YEZvRX9R03Z28uuFP6jPLdaAHUpyuodXZ03BH5tQJumC5iuFTZriKECiG69PJ3dCZWw6dwJljBfDlNH3Ujq69b1RFnjHqP1oGjn6gYaPPffTZHGbJMw4PCq8a6amuDwF+psrTD/mdh8QnFcWHa83RSUJ+n3K3/DsgvG19ko0NefIwMmb109FVd5X/lKZNyBO23Llbig8Grhq5dqwKft7f/9yVv3B4fyhsnzxs/WPDDlwn8F3bxggfReAuaEpdUZXxZf96/O3qtsI1qm21rrW0RkNPw6GX1Vo8vduFT4lZabyOxVWYstE/vvhGeK7wJ2Cxe5l9oAv11EygY67R82g0lAHFJDezDBLQC+LgVVR4xljIE59l4r1WsKpM\u2026(truncated)",
  "action": "getlog",
  "action_inmessage": "true"
}
```


### `cresco_agent_getmetricinventory`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `getmetricinventory` · **Category:** read

Return this node's unified metric inventory (node scope).

**Chain of events / interacts with:** Node-scoped metric inventory (this node only, no fan-out). The global/region getmetricinventory fan-out calls exactly this on each agent.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `action_include_plugins` | boolean | no |  |
| `action_include_resource` | boolean | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metricinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_getmetricinventory",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "action_include_plugins": "\u2026",
    "action_include_resource": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getmetricinventory", "action_include_plugins": ..., "action_include_resource": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "metricinventory": "{\"node\":\"global-region_global-controller\",\"metrics_by_source\":{\"global-region_global-controller:io.cresco.agent.controller\":{\"jvm\":[{\"name\":\"jvm.memory.max\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"-1.0\",\"group\":\"jvm\"},{\"name\":\"jvm.memory.used\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"8046840.0\",\"group\":\"jvm\"},{\"name\":\"jvm.classes.loaded\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"12143.0\",\"group\":\"jvm\"},{\"name\":\"jvm.classes.unloaded\",\"count\":\"1.0\",\"type\":\"NODE\",\"class\":\"COUNTER\",\"group\":\"jvm\"},{\"name\":\"jvm.buffer.memory.used\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"6998775.0\",\"group\":\"jvm\"},{\"name\":\"j\u2026(truncated)",
  "action": "getmetricinventory",
  "status": "10"
}
```


### `cresco_agent_iscontrolleractive`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `iscontrolleractive` · **Category:** read

Return whether this agent's controller is active.

**Chain of events / interacts with:** Boolean readiness check on the controller FSM.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_controller_active` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_iscontrolleractive",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "iscontrolleractive"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "is_controller_active": "true",
  "action": "iscontrolleractive"
}
```


### `cresco_agent_killjvm`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `killjvm` · **Category:** destructive

Kill the agent JVM (async).

**Chain of events / interacts with:** CoreState.killJVM() — async hard stop of the agent JVM process. DESTRUCTIVE.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_agent_killjvm",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "killjvm"}, region, agent)
```

**Not exercised live** — destructive (would tear down the node). Effect: see chain above.


### `cresco_agent_listagents`

**Tier:** agent · **MsgEvent:** EXEC · **Action:** `listagents` · **Category:** read

List agents in this agent's region.

**Chain of events / interacts with:** Lists the agents in THIS agent's region (agent-local view; the global tier lists the whole fabric).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `action_region` | string | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `agentslist` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_agent_listagents",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "action_region": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "listagents", "action_region": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "agentslist": {
    "agents": [
      {
        "environment": "Mac OS X",
        "agent_id": "global-controller",
        "status_desc": "{\"mode\":\"GLOBAL\"}",
        "plugins": "4",
        "region_id": "global-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      },
      {
        "environment": "Mac OS X",
        "agent_id": "edge-controller",
        "status_desc": "isRegionGlobal() Success",
        "plugins": "2",
        "region_id": "edge-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      },
      {
        "environment": "Mac OS X",
        "agent_id": "edge-agent-001",
        "status_desc": "{\"mode\":\"AGENT\",\"broker\":\"127.0.0.1\"}",
        "plugins": "2",
        "region_id": "edge-region",
        "location": "Codys-MacBook-Pro.local",
        "platform": "unknown"
      }
    ]
  },
  "action": "listagents"
}
```


### `cresco_agent_pluginadd`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginadd` · **Category:** artifact

Add/start a plugin bundle on this agent.

**Chain of events / interacts with:** AgentExecutor → ControllerEngine.getPluginAdmin().addPlugin(): installs and starts an OSGi bundle (Felix), whose DS @Component wires a new Executor. Returns the assigned pluginid.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `configparams` | object | yes |  |
| `edges` | object | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |
| `pluginid` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginadd",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "configparams": "\u2026",
    "edges": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginadd", "configparams": ..., "edges": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_pluginlist`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginlist` · **Category:** read

List all plugins on this agent.

**Chain of events / interacts with:** PluginAdmin enumerates the plugins currently loaded on THIS agent.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `plugin_list` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginlist",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginlist"}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "plugin_list": [
    {
      "plugin_id": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e",
      "status_code": "10",
      "persistence_code": "20",
      "status_desc": "Plugin Active",
      "pluginname": "io.cresco.repo",
      "jarfile": "repo.jar",
      "watchdog_period": "0",
      "version": "1.3.0.SNAPSHOT-20260703160835",
      "md5": "f95a7af8541064aecdb456b7f956b934",
      "configparams": "{\"jarstatus\":\"embedded\",\"persistence_code\":\"20\",\"pluginname\":\"io.cresco.repo\",\"jarfile\":\"repo.jar\",\"inode_id\":\"system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e\",\"version\":\"1.3.0.SNAPSHOT-20260703160835\",\"md5\":\"f95a7af8541064aecdb456b7f956b934\"}"
    },
    {
      "plugin_id": "system-9a585863-7e2a-4476-a12b-17086df51b9c",
      "status_code": "10",
      "persistence_code": "20",
      "status_desc": "Plugin Active",
      "pluginname": "io.cresco.stunnel",
      "jarfile": "stunnel.jar",
      "watchdog_period": "0",
      "version": "1.3.0.SNAPSHOT-20260703160840",
      "md5": "f57bf502d10920331b0c2da7ae384b4e",
      "configparams": "{\"jarstatus\":\"embedded\",\"persistence_code\":\"20\",\"pluginname\":\"io.cresco.stunnel\",\"jarfile\":\"stunnel.jar\",\"inode_id\":\"system-9a585863-7e2a-4476-a12b-17086df51b9c\",\"version\":\"1.3.0.SNAPSHOT-20260703160840\",\"md5\":\"f57bf502d10920331b0c2da7ae384b4e\"}"
    },
    {
      "plugin_id": "system-5ed6fd
… (truncated)
```


### `cresco_agent_pluginremove`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginremove` · **Category:** artifact

Stop/remove a plugin on this agent.

**Chain of events / interacts with:** PluginAdmin stops and uninstalls the plugin's OSGi bundle by pluginid.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginremove",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginremove", "pluginid": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_pluginrepopull`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginrepopull` · **Category:** artifact

Validate a set of plugins against the repo.

**Chain of events / interacts with:** Validates a plugin set against the repo (pluginAdmin.remotePluginMap) and pulls any missing artifacts before deploy.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `configparams` | object | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_updated` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginrepopull",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "configparams": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginrepopull", "configparams": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_pluginstatus`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginstatus` · **Category:** read

Get the status of one plugin on this agent.

**Chain of events / interacts with:** PluginAdmin.getPluginStatus(pluginid) — one local plugin's status code/desc.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `plugin_status` | object | yes |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginstatus",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginstatus", "pluginid": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "status_code": "10",
  "status_desc": "Plugin Active",
  "pluginid": "system-bf2a5471-fa3a-4adb-8146-62ed39350b0e",
  "action": "pluginstatus",
  "plugin_status": {
    "status_code": "10",
    "status_desc": "Plugin Active",
    "isactive": "true"
  }
}
```


### `cresco_agent_pluginupload`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `pluginupload` · **Category:** artifact

Upload a plugin JAR and register it.

**Chain of events / interacts with:** Writes an uploaded JAR to the agent's plugin dir and registers it (pluginUpdate), making it available to load.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `configparams` | object | yes |  |
| `jardata` | binary | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_updated` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_pluginupload",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "configparams": "\u2026",
    "jardata": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "pluginupload", "configparams": ..., "jardata": ...}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_agent_restartcontroller`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `restartcontroller` · **Category:** destructive

Restart this agent's controller (async).

**Chain of events / interacts with:** CoreState.restartController() — async controller restart (state reload). DESTRUCTIVE mid-call.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_agent_restartcontroller",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "restartcontroller"}, region, agent)
```

**Not exercised live** — destructive (would tear down the node). Effect: see chain above.


### `cresco_agent_restartframework`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `restartframework` · **Category:** destructive

Restart the OSGi framework (async).

**Chain of events / interacts with:** CoreState.restartFramework() — async full OSGi framework restart. DESTRUCTIVE.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_agent_restartframework",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "restartframework"}, region, agent)
```

**Not exercised live** — destructive (would tear down the node). Effect: see chain above.


### `cresco_agent_setlogdp`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `setlogdp` · **Category:** mutate_safe

Enable/disable dataplane log streaming for a session.

**Chain of events / interacts with:** Enables/disables data-plane log streaming for a session — turns a live log feed on/off.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `session_id` | string | yes |  |
| `setlogdp` | boolean | yes |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_setlogdp",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "session_id": "\u2026",
    "setlogdp": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "setlogdp", "session_id": ..., "setlogdp": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "status_code": "7",
  "status_desc": "logDP Set",
  "action": "setlogdp",
  "session_id": "suite-sess",
  "setlogdp": "false"
}
```


### `cresco_agent_setloglevel`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `setloglevel` · **Category:** mutate_safe

Set the log level for a class/session (Trace/Debug/Info/Warn/Error).

**Chain of events / interacts with:** Sets the SLF4J/logback level for a class (or globally) for a session. If data-plane logging is enabled for the session, matching log records also stream over the dataplane.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `session_id` | string | yes |  |
| `loglevel` | string | yes |  |
| `baseclassname` | string | no |  |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status_code` | string | no |  |

**Tool call:**

```json
{
  "name": "cresco_agent_setloglevel",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "session_id": "\u2026",
    "loglevel": "\u2026",
    "baseclassname": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "setloglevel", "session_id": ..., "loglevel": ..., "baseclassname": ...}, region, agent)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "status_code": "9",
  "status_desc": "LogLevel Could Not Be Set",
  "loglevel": "Info",
  "baseclassname": "io.cresco.sysinfo",
  "action": "setloglevel",
  "session_id": "suite-sess"
}
```


### `cresco_agent_stopcontroller`

**Tier:** agent · **MsgEvent:** CONFIG · **Action:** `stopcontroller` · **Category:** destructive

Stop this agent's controller (async).

**Chain of events / interacts with:** CoreState.stopController() — async graceful controller stop. DESTRUCTIVE: the node leaves the fabric.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_agent_stopcontroller",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "stopcontroller"}, region, agent)
```

**Not exercised live** — destructive (would tear down the node). Effect: see chain above.



---

## regional tools

**Regional tier** — answered by a **region controller** (`RegionalExecutor`). These mirror global/agent actions but are **scoped to a single region**. Same action name as a global tool ≠ duplicate: the global version aggregates all regions, the regional version reports one region. Region-tier inventories are normally reached via the global `getcapabilityinventory`/`getmetricinventory` fan-out (region controllers appear as children).

### `cresco_regional_agent_disable`

**Tier:** regional · **MsgEvent:** CONFIG · **Action:** `agent_disable` · **Category:** artifact

Unregister an agent from this region.

**Chain of events / interacts with:** getGDB().removeNode() unregisters an agent from this region.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_unregistered` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_regional_agent_disable",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "agent_disable"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_regional_agent_enable`

**Tier:** regional · **MsgEvent:** CONFIG · **Action:** `agent_enable` · **Category:** artifact

Register an agent in this region.

**Chain of events / interacts with:** RegionalExecutor.executeCONFIG → getGDB().nodeUpdate() registers an agent in THIS region's registry (region-scoped, vs the global fabric registry).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `is_registered` | boolean | no |  |

**Tool call:**

```json
{
  "name": "cresco_regional_agent_enable",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "CONFIG", {"action": "agent_enable"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_regional_getcapabilities`

**Tier:** regional · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return the regional controller's self-describing capability document.

**Chain of events / interacts with:** The regional tier's own tool document (RegionalExecutor annotations).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_regional_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_regional_getcapabilityinventory`

**Tier:** regional · **MsgEvent:** EXEC · **Action:** `getcapabilityinventory` · **Category:** read

Return this region node's capability inventory (node scope).

**Chain of events / interacts with:** REGION-scoped capability catalog — this region's nodes. The single-region counterpart to the global fabric-wide catalog; reached via the global fan-out (region controllers appear as children).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilityinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_regional_getcapabilityinventory",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getcapabilityinventory"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_regional_getmetricinventory`

**Tier:** regional · **MsgEvent:** EXEC · **Action:** `getmetricinventory` · **Category:** read

Return this region node's unified metric inventory (node scope).

**Chain of events / interacts with:** REGION-scoped metric inventory — this region's node view. Same action as the global one but scoped to one region rather than aggregating the whole fabric (delegated to the region's GlobalExecutor). Primarily reached via the global fan-out, which addresses region controllers as children.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metricinventory` | object | no |  |

**Tool call:**

```json
{
  "name": "cresco_regional_getmetricinventory",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "getmetricinventory"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_regional_ping`

**Tier:** regional · **MsgEvent:** EXEC · **Action:** `ping` · **Category:** read

Liveness ping; replies pong and exchanges mesh health.

**Chain of events / interacts with:** Agent→region liveness: records the agent's advertised health (MeshHealthPing), stamps the region's health back, replies pong. The region-scoped analogue of the region→global ping.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `action` | string | no | pong |

**Tool call:**

```json
{
  "name": "cresco_regional_ping",
  "input": {
    "region": "\u2026",
    "agent": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_agent_msgevent(True, "EXEC", {"action": "ping"}, region, agent)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.



---

## sysinfo tools

**sysinfo plugin** — host telemetry: OS/CPU/memory/disk/network snapshot, CPU benchmark, and unified live metrics. Routed with `global_plugin_msgevent(region, agent, pluginid)`.

### `cresco_sysinfo_getbenchmark`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getbenchmark` · **Category:** read

Run and return a SciMark2 CPU benchmark result as compressed JSON.

**Chain of events / interacts with:** Benchmark runs the SciMark2 CPU suite → a compute score. SLOW (seconds) — run it in isolation with a long timeout.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_sysinfo_getbenchmark",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getbenchmark"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "error": "Cresco rpc response was null"
}
```


### `cresco_sysinfo_getcapabilities`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return this plugin's self-describing capability document (its message actions as LLM tool specs).

**Chain of events / interacts with:** Scans sysinfo's annotations → its tool document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no | CapabilityDocument JSON |

**Tool call:**

```json
{
  "name": "cresco_sysinfo_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"sysinfo\",\"bundle\":\"io.cresco.sysinfo\",\"version\":\"1.3.0.SNAPSHOT-20260703160832\",\"summary\":\"Host system-information plugin: OS/CPU/memory/disk/network snapshot, CPU benchmark, and unified live metrics (incl. sensors/power/gpu).\",\"actions\":[{\"namespace\":\"sysinfo\",\"action\":\"getsysinfo\",\"msgType\":\"EXEC\",\"summary\":\"Return a full host system-information snapshot (OS, CPU, memory, disk, filesystem, network) as compressed JSON.\",\"why\":\"Use for a one-shot host inventory / resource view. For time-series metrics prefer getmetrics.\",\"target\":\"plugin\",\"routingParams\":[\"region\",\"agent\",\"plugin\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_sysinfo_getmetrics`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getmetrics` · **Category:** read

Return this plugin's live metrics (cpu/memory/disk/network/sensors/power/gpu/devices) as MeasurementEngine gauges JSON.

**Chain of events / interacts with:** SysInfoBuilder.getMetricsJson() → MeasurementEngine gauges (cpu/mem/disk/net + sensors/power/gpu/devices) grouped by group. Folded into getmetricinventory.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metrics` | object | no | getAllMetrics() JSON grouped by metric group |

**Tool call:**

```json
{
  "name": "cresco_sysinfo_getmetrics",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getmetrics"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getmetrics",
  "metrics": "{\"disk\":[{\"name\":\"disk.available\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"624607502336\",\"group\":\"disk\"},{\"name\":\"disk.total\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"5974917240832\",\"group\":\"disk\"},{\"name\":\"disk.write.bytes\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"3940193542144\",\"group\":\"disk\"},{\"name\":\"disk.read.bytes\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"5397028114432\",\"group\":\"disk\"}],\"system\":[{\"name\":\"system.uptime.seconds\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"2860499\",\"group\":\"system\"},{\"name\":\"system.process.count\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"880\",\"group\":\"system\"},{\"name\":\"sy\u2026(truncated)",
  "status": "10"
}
```


### `cresco_sysinfo_getsysinfo`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getsysinfo` · **Category:** read

Return a full host system-information snapshot (OS, CPU, memory, disk, filesystem, network) as compressed JSON.

**Chain of events / interacts with:** SysInfoBuilder.getSysInfoMap() reads OSHI (JNA-native) → a full OS/CPU/memory/disk/filesystem/network snapshot as compressed JSON.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `perf` | object | yes | compressed JSON: {os,cpu,mem,disk,fs,net,...} |

**Tool call:**

```json
{
  "name": "cresco_sysinfo_getsysinfo",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getsysinfo"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getsysinfo",
  "perf": {
    "disk": [
      {
        "disk-writebytes": "1970098716672",
        "disk-writes": "123530202",
        "disk-size": "1000555581440",
        "disk-model": "APPLE SSD AP1024Z",
        "disk-readbytes": "2698515505152",
        "disk-name": "disk0",
        "disk-transfertime": "39997989",
        "disk-reads": "156439243"
      },
      {
        "disk-writebytes": "280150016",
        "disk-writes": "26966",
        "disk-size": "524288000",
        "disk-model": "APPLE SSD AP1024Z",
        "disk-readbytes": "119095296",
        "disk-name": "disk1",
        "disk-transfertime": "0",
        "disk-reads": "27771"
      },
      {
        "disk-writebytes": "1968556064768",
        "disk-writes": "123501681",
        "disk-size": "994662584320",
        "disk-model": "APPLE SSD AP1024Z",
        "disk-readbytes": "2698393210880",
        "disk-name": "disk3",
        "disk-transfertime": "0",
        "disk-reads": "156411117"
      },
      {
        "disk-writebytes": "1262501888",
        "disk-writes": "1555",
        "disk-size": "5368664064",
        "disk-model": "APPLE SSD AP1024Z",
        "disk-readbytes": "3121152",
        "disk-name": "disk2",
        "disk-transfertime": "0",
        "disk-reads": "342"
      }
    ],
    "os": [
      {
        "sys-manufacturer": "Apple",
        "process-count": "881",
        "sys
… (truncated)
```



---

## stunnel tools

**stunnel plugin** — secure TCP tunnels across the fabric (forward a local port to a remote host:port via a src/dst tunnel pair). Routed with `global_plugin_msgevent(region, agent, pluginid)`.

### `cresco_stunnel_configdstsession`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `configdstsession` · **Category:** artifact

Open a destination-side session (client connection) for an existing dst tunnel.

**Chain of events / interacts with:** Per-connection dst setup — SocketController.createDstSession connects to the target host:port for one client session.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |
| `action_session_config` | object | yes | compressed JSON: {stunnel_id, client_id} |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status` | string | no | 10 success / 9 fail |

**Tool call:**

```json
{
  "name": "cresco_stunnel_configdstsession",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026",
    "action_session_config": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "configdstsession", "action_session_config": ...}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_stunnel_configdsttunnel`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `configdsttunnel` · **Category:** artifact

Configure the destination side of a TCP tunnel (the agent nearest the target host:port).

**Chain of events / interacts with:** Registers the destination side (the agent nearest the target host:port); it dials the real service when a session opens.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |
| `action_tunnel_config` | object | yes | compressed JSON tunnel config |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status` | string | no | 10 success / 9 fail |

**Tool call:**

```json
{
  "name": "cresco_stunnel_configdsttunnel",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026",
    "action_tunnel_config": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "configdsttunnel", "action_tunnel_config": ...}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_stunnel_configsrctunnel`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `configsrctunnel` · **Category:** mutate_safe

Create the source (listener) side of a TCP tunnel.

**Chain of events / interacts with:** SocketController.startSrcTunnel opens a local ServerSocket listener bound to src_port; on each client connection it opens a fabric-spanning channel to the paired dst tunnel and pumps bytes. Interacts with: the dst agent's stunnel (via configdstsession), the local TCP client.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_stunnel_configsrctunnel",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "configsrctunnel"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_tunnel_config": "H4sIANjoR2oC/03MwQ7CIAyA4VcxnK0B3AbbyxC2VSRBMFAOxvjubstmTE/9/qZvVqjGiMH4mQ0nVqonhIXYeVnyZJ4p0xr4CnMhc09lAyHVhS8jjnBc9gdkdD7FlVxIow2ww56tw0h/dUqRcgoB8+9jqM5vD8qrED6gt61udXcFhdJC06gOrJAjCMV1N99aMfYT+3wB8CFqVdQAAAA=",
  "status_desc": "Invalid JSON format in action_tunnel_config.",
  "action": "configsrctunnel",
  "status": "400"
}
```


### `cresco_stunnel_getcapabilities`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return this plugin's self-describing capability document (its message actions as LLM tool specs).

**Chain of events / interacts with:** Scans stunnel's annotations → its tool document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no | CapabilityDocument JSON |

**Tool call:**

```json
{
  "name": "cresco_stunnel_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"stunnel\",\"bundle\":\"io.cresco.stunnel\",\"version\":\"1.3.0.SNAPSHOT-20260703160840\",\"summary\":\"Secure TCP tunnel plugin: forwards a local TCP port across the Cresco fabric to a remote host:port via a src/dst tunnel pair, with health checks and live metrics.\",\"actions\":[{\"namespace\":\"stunnel\",\"action\":\"configsrctunnel\",\"msgType\":\"CONFIG\",\"summary\":\"Create the source (listener) side of a TCP tunnel: opens a local port that forwards to a remote dst tunnel.\",\"why\":\"First step to expose a remote TCP service locally. Pair with configdsttunnel on the remote agent.\",\"target\":\"plugin\",\"routin\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_stunnel_getmetrics`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getmetrics` · **Category:** read

Return live tunnel metrics (active tunnels/clients/targets) as MeasurementEngine gauges JSON.

**Chain of events / interacts with:** SocketController metrics → active tunnels/clients/targets gauges.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metrics` | object | no | getAllMetrics() JSON |

**Tool call:**

```json
{
  "name": "cresco_stunnel_getmetrics",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getmetrics"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getmetrics",
  "metrics": "{\n  \"stunnel\": [\n    {\n      \"name\": \"stunnel.active.tunnels\",\n      \"type\": \"NODE\",\n      \"class\": \"GAUGE\",\n      \"value\": \"0\",\n      \"group\": \"stunnel\"\n    },\n    {\n      \"name\": \"stunnel.active.clients\",\n      \"type\": \"NODE\",\n      \"class\": \"GAUGE\",\n      \"value\": \"0\",\n      \"group\": \"stunnel\"\n    },\n    {\n      \"name\": \"stunnel.active.targets\",\n      \"type\": \"NODE\",\n      \"class\": \"GAUGE\",\n      \"value\": \"0\",\n      \"group\": \"stunnel\"\n    }\n  ]\n}",
  "status": "10"
}
```


### `cresco_stunnel_gettunnelconfig`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `gettunnelconfig` · **Category:** read

Get the full configuration of one tunnel.

**Chain of events / interacts with:** SocketController.getTunnelConfig(id) — the tunnel's src/dst wiring + tuning.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_stunnel_gettunnelconfig",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "gettunnelconfig"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_stunnel_id": "suite-tun",
  "status_desc": "Tunnel config not found for suite-tun",
  "action": "gettunnelconfig",
  "status": "9"
}
```


### `cresco_stunnel_gettunnelstatus`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `gettunnelstatus` · **Category:** read

Get the live status of one tunnel (ACTIVE/RECOVERING/DOWN/UNKNOWN).

**Chain of events / interacts with:** SocketController.getTunnelStatus(id) — ACTIVE (listener open) / RECOVERING (down, reconnect scheduled) / DOWN / UNKNOWN.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_stunnel_gettunnelstatus",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "gettunnelstatus"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_stunnel_id": "suite-tun",
  "status_desc": "Tunnel config not found for suite-tun",
  "action": "gettunnelstatus",
  "status": "9"
}
```


### `cresco_stunnel_listtunnels`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `listtunnels` · **Category:** read

List all active tunnels on this node with their live status.

**Chain of events / interacts with:** SocketController.getActiveTunnels() — all tunnels + live status.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `tunnels` | array | no | JSON array of {stunnel_id, status} |

**Tool call:**

```json
{
  "name": "cresco_stunnel_listtunnels",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "listtunnels"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "tunnels": "[]",
  "status_desc": "Successfully retrieved tunnel list.",
  "action": "listtunnels",
  "status": "10"
}
```


### `cresco_stunnel_nettuning`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `nettuning` · **Category:** artifact

Apply fabric-wide network tuning (buffer/block sizes) to live tunnels.

**Chain of events / interacts with:** SocketController.applyNetTuning applies buffer/block sizes pushed by the controller AutoTuner to live tunnels.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status` | string | no | 10 on success |

**Tool call:**

```json
{
  "name": "cresco_stunnel_nettuning",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "nettuning"}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_stunnel_removedsttunnel`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `removedsttunnel` · **Category:** artifact

Tear down the destination side of a tunnel.

**Chain of events / interacts with:** SocketController.removeDstTunnel closes the dst side.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |
| `action_stunnel_id` | string | yes | tunnel id |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status` | string | no | 10 success / 9 fail |

**Tool call:**

```json
{
  "name": "cresco_stunnel_removedsttunnel",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026",
    "action_stunnel_id": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "removedsttunnel", "action_stunnel_id": ...}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_stunnel_removesrctunnel`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `removesrctunnel` · **Category:** mutate_safe

Tear down the source (listener) side of a tunnel.

**Chain of events / interacts with:** SocketController.removeSrcTunnel closes the listener + all channels for the src tunnel.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_stunnel_removesrctunnel",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "removesrctunnel"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_stunnel_id": "suite-tun",
  "status_desc": "SRC tunnel removal initiated for suite-tun",
  "action": "removesrctunnel",
  "status": "10"
}
```


### `cresco_stunnel_tunnelhealthcheck`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `tunnelhealthcheck` · **Category:** read

Check whether a tunnel config exists on this node.

**Chain of events / interacts with:** Reads SocketController state — is a config present for this tunnel id.

**Parameters:**

_none_

**Returns:**

_none / status only_

**Tool call:**

```json
{
  "name": "cresco_stunnel_tunnelhealthcheck",
  "input": {}
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "tunnelhealthcheck"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action_stunnel_id": "suite-tun",
  "status_desc": "Tunnel config not found locally for suite-tun",
  "action": "tunnelhealthcheck",
  "status": "9"
}
```



---

## wsapi tools

**wsapi plugin** — the WebSocket gateway external clients connect through (control-plane RPC + dataplane streaming). Routed with `global_plugin_msgevent(region, agent, pluginid)`.

### `cresco_wsapi_getcapabilities`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return this plugin's self-describing capability document (its message actions as LLM tool specs).

**Chain of events / interacts with:** Scans wsapi's annotations → its tool document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no | CapabilityDocument JSON |

**Tool call:**

```json
{
  "name": "cresco_wsapi_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"wsapi\",\"bundle\":\"io.cresco.wsapi\",\"version\":\"1.3.0.SNAPSHOT-20260703160837\",\"summary\":\"WebSocket API gateway: external clients connect over wss for control-plane RPC and dataplane streaming; exposes global location, live buffer tuning, and dataplane metrics.\",\"actions\":[{\"namespace\":\"wsapi\",\"action\":\"nettuning\",\"msgType\":\"CONFIG\",\"summary\":\"Apply fabric-wide network tuning (socket buffer / read-chunk / write-high-water) to the live Netty WebSocket server.\",\"why\":\"Pushed by the controller AutoTuner to adapt the wss server\\u0027s I/O sizing under load; new connections pick up the v\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_wsapi_getmetrics`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getmetrics` · **Category:** read

Return wsapi dataplane metrics (active connections, bytes, messages) as MeasurementEngine gauges JSON.

**Chain of events / interacts with:** MeasurementEngine gauges: active dataplane WebSocket connections, total bytes, total messages.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metrics` | object | no | getAllMetrics() JSON |

**Tool call:**

```json
{
  "name": "cresco_wsapi_getmetrics",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getmetrics"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getmetrics",
  "metrics": "{\"wsapi\":[{\"name\":\"wsapi.dataplane.messages\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"0\",\"group\":\"wsapi\"},{\"name\":\"wsapi.dataplane.connections\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"0\",\"group\":\"wsapi\"},{\"name\":\"wsapi.dataplane.bytes\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"0\",\"group\":\"wsapi\"}]}",
  "status": "10"
}
```


### `cresco_wsapi_globalinfo`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `globalinfo` · **Category:** read

Return the global controller's region and agent identity.

**Chain of events / interacts with:** Reads AgentState → the global controller's region+agent identity.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `global_region` | string | no | global controller region |
| `global_agent` | string | no | global controller agent |

**Tool call:**

```json
{
  "name": "cresco_wsapi_globalinfo",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "globalinfo"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "global_region": "global-region",
  "action": "globalinfo",
  "global_agent": "global-controller"
}
```


### `cresco_wsapi_nettuning`

**Tier:** plugin · **MsgEvent:** CONFIG · **Action:** `nettuning` · **Category:** artifact

Apply fabric-wide network tuning (socket buffer / read-chunk / write-high-water) to the live Netty WebSocket server.

**Chain of events / interacts with:** NettyWsServer.applyNetTuning applies socket-buffer/read-chunk/write-high-water sizes to the live wss server; new connections read the updated values.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `status` | string | no | 10 on success |

**Tool call:**

```json
{
  "name": "cresco_wsapi_nettuning",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "CONFIG", {"action": "nettuning"}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.



---

## repo tools

**repo plugin** — the plugin-artifact repository: list/serve/store plugin JARs for the fabric. Routed with `global_plugin_msgevent(region, agent, pluginid)`.

### `cresco_repo_getcapabilities`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getcapabilities` · **Category:** read

Return this plugin's self-describing capability document (its message actions as LLM tool specs).

**Chain of events / interacts with:** Scans repo's annotations → its tool document.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `capabilities` | object | no | CapabilityDocument JSON |

**Tool call:**

```json
{
  "name": "cresco_repo_getcapabilities",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getcapabilities"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "capabilities": "{\"namespace\":\"repo\",\"bundle\":\"io.cresco.repo\",\"version\":\"1.3.0.SNAPSHOT-20260703160835\",\"summary\":\"Plugin repository: lists/serves/stores plugin JARs for the fabric, plus repo inventory metrics.\",\"actions\":[{\"namespace\":\"repo\",\"action\":\"repolist\",\"msgType\":\"EXEC\",\"summary\":\"List every plugin JAR in this repo (name, md5, jar file) plus this repo\\u0027s server identity.\",\"why\":\"Use to discover which plugins are available to deploy and their md5 for integrity-checked pulls.\",\"target\":\"plugin\",\"routingParams\":[\"region\",\"agent\",\"pluginid\"],\"params\":[],\"returns\":[{\"name\":\"repolist\",\"type\":\"object\",\"\u2026(truncated)",
  "action": "getcapabilities",
  "status": "10"
}
```


### `cresco_repo_getjar`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getjar` · **Category:** artifact

Retrieve a plugin JAR's bytes by plugin name + md5.

**Chain of events / interacts with:** Looks up a jar by pluginname+md5 in the inventory and returns its bytes — the artifact-fetch path used during deployment.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |
| `action_pluginname` | string | yes | plugin symbolic name, e.g. io.cresco.sysinfo |
| `action_pluginmd5` | string | yes | md5 of the desired jar (integrity + version select) |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `jardata` | binary | no | raw jar bytes (present only if found) |

**Tool call:**

```json
{
  "name": "cresco_repo_getjar",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026",
    "action_pluginname": "\u2026",
    "action_pluginmd5": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getjar", "action_pluginname": ..., "action_pluginmd5": ...}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_repo_getmetrics`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `getmetrics` · **Category:** read

Return repo inventory metrics (plugin count, total bytes on disk) as MeasurementEngine gauges JSON.

**Chain of events / interacts with:** MeasurementEngine gauges: repo plugin count + total bytes on disk.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `metrics` | object | no | getAllMetrics() JSON |

**Tool call:**

```json
{
  "name": "cresco_repo_getmetrics",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "getmetrics"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "getmetrics",
  "metrics": "{\"repo\":[{\"name\":\"repo.plugin.count\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"0\",\"group\":\"repo\"},{\"name\":\"repo.bytes\",\"type\":\"NODE\",\"class\":\"GAUGE\",\"value\":\"0\",\"group\":\"repo\"}]}",
  "status": "10"
}
```


### `cresco_repo_putjar`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `putjar` · **Category:** artifact

Store a plugin JAR into this repo (writes to disk, verifies md5).

**Chain of events / interacts with:** Writes an uploaded jar to disk and verifies its md5 — the artifact-publish path (savetorepo calls this on every repo).

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |
| `pluginname` | string | yes | plugin symbolic name |
| `md5` | string | yes | expected md5 of the jar |
| `jarfile` | string | yes | target jar file name |
| `version` | string | no | plugin version |
| `jardata` | binary | yes | raw jar bytes |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `uploaded` | boolean | no | true if stored |
| `md5-confirm` | string | no | md5 of the stored jar |

**Tool call:**

```json
{
  "name": "cresco_repo_putjar",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026",
    "pluginname": "\u2026",
    "md5": "\u2026",
    "jarfile": "\u2026",
    "version": "\u2026",
    "jardata": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "putjar", "pluginname": ..., "md5": ..., "jarfile": ..., "version": ..., "jardata": ...}, region, agent, pluginid)
```

**Not exercised live** — requires an artifact/scheduler target not provisioned by the suite. Request shape and effect are documented above.


### `cresco_repo_repolist`

**Tier:** plugin · **MsgEvent:** EXEC · **Action:** `repolist` · **Category:** read

List every plugin JAR in this repo (name, md5, jar file) plus this repo's server identity.

**Chain of events / interacts with:** Scans the repo directory + PluginBuilder.getPluginInventory → {plugins:[{pluginname,md5,jarfile}], server:[{region,agent,pluginid}]}. The catalog of deployable artifacts this repo holds.

**Parameters:**

| param | type | required | description |
|---|---|---|---|
| `region` | string | yes | Routing identity: target region for this call. |
| `agent` | string | yes | Routing identity: target agent for this call. |
| `pluginid` | string | yes | Routing identity: target pluginid for this call. |

**Returns:**

| return param | type | compressed | description |
|---|---|---|---|
| `repolist` | object | yes | compressed JSON: {plugins:[{pluginname,md5,jarfile}], server:[{region,agent,pluginid}]} |

**Tool call:**

```json
{
  "name": "cresco_repo_repolist",
  "input": {
    "region": "\u2026",
    "agent": "\u2026",
    "pluginid": "\u2026"
  }
}
```

**pycrescolib:**

```python
reply = client.messaging.global_plugin_msgevent(True, "EXEC", {"action": "repolist"}, region, agent, pluginid)
```

**Live response** (captured by `tool_suite.py`):

```json
{
  "action": "repolist",
  "repolist": {
    "server": [
      {
        "agent": "global-controller",
        "pluginid": "system-28b503fa-97b2-4db7-9abc-1c20ca1bfb1e",
        "region": "global-region"
      }
    ]
  }
}
```

