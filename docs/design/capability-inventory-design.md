# Cresco — Self-Describing Capability Inventory for LLM Tool-Calling

**Status:** ✅ SHIPPED + PROVEN (2026-07-03). Live mesh test `run/tests/capability_inventory_test.sh`
**10/10** — one `getcapabilityinventory` (scope=global) returns **89 tools** across all 7 namespaces
(agent/regional/global controller tiers + sysinfo/stunnel/wsapi/repo plugins), every one well-formed
Anthropic tool JSON, and an action (`cresco_global_listagents`) was **invoked using only its
descriptor/binding** via the MCP tool-runner. Drift check `capability_drift.py` PASS (all 89 switch cases
match their annotations). No regression: metrics 11/11, tenant isolation 10/10.

**Key constraint (user):** external clients reach Cresco only over the WebSocket message bus, so **only
MsgEvent actions are callable tools.** The OSGi service/package surface is included as *informational
metadata* (opt-in, `include_osgi`) and is never turned into a tool — the tool-runner walks only
`capabilities_by_source[*].actions`. The client's `get_capability_inventory` defaults `include_osgi=false`
(MsgEvent-only view).

**No hardcoded identity:** bundle symbolic name + version come from the OSGi manifest via
`FrameworkUtil.getBundle(clazz)` — never hardcoded in source.

Built on user direction: every bundle (controller tiers + every plugin) publishes, via the inventory, a
machine-readable description of every MsgEvent action it handles AND the classes/services it exposes via
OSGi — in a form an LLM can consume **directly as tool definitions**.

**Decisions (locked with the user):**
- **Authoring = annotations on handlers.** `@CrescoAction`/`@CrescoParam`/`@CrescoReturn` on per-action
  methods, scanned by reflection. A drift test asserts every `switch` case has a matching annotation.
- **Scope = message actions + OSGi services/packages.** Also enumerate each bundle's `Export-Package`
  and registered service interfaces.
- **Client depth = full, incl. MCP tool-runner — but the runner is a TEST harness** in `run/tests/`
  (proves the catalog can drive Cresco), NOT a new plugin/library.

Mirrors the proven metrics-unification pattern one layer up: per-bundle EXEC (`getcapabilities`) →
controller aggregation (`getcapabilityinventory`, node/region/global fan-out) → client pull → (test) runner.

## 1. Current reality
7 executor classes (Agent/Regional/Global controllers + stunnel/sysinfo/wsapi/repo) implement the
`io.cresco.library.plugin.Executor` interface (`executeEXEC/CONFIG/DISCOVER/ERROR/INFO/WATCHDOG/KPI`),
each dispatching a bare `switch(incoming.getParam("action"))` over ~60 total actions. **No metadata
exists** — params are read ad-hoc (`getParam("action_region")`), returns set ad-hoc; semantics live only
in code + sparse comments. The action surface is completely opaque to any caller.

## 2. Descriptor model — `io.cresco.library.capability`
Each action → an `ActionDescriptor` with two faces:

**LLM-facing (Anthropic tool spec):**
```json
{ "name":"cresco_global_listagents",
  "description":"List agents in the fabric, optionally filtered by region. Use before targeting a command to discover which agents are online. Agents are the unit of plugin placement.",
  "input_schema":{"type":"object","properties":{"action_region":{"type":"string","description":"Region filter; omit for all."}},"required":[]} }
```
**Cresco binding (how a runner invokes it):**
```json
{ "msg_type":"EXEC", "target":"global", "action":"listagents",
  "routing_params":[], "returns":{"agentslist":"compressed JSON {agents:[...]}"} }
```
`target` ∈ {global, regional, agent, plugin}; `routing_params` names identity params a caller must
supply to route (plugin actions need region+agent+pluginid). This is what lets a generic runner build the
right `global_controller_msgevent`/`plugin_msgevent`, set action+params, send RPC, and parse the reply.

### Types
- `@CrescoCapabilities` (class-level): `namespace` (e.g. "global","stunnel"), `target` default, shared
  `routingParams` default, human bundle summary.
- `@CrescoAction` (method-level): `name`, `type` (MsgEvent.Type, default EXEC), `summary`, `why`,
  `params()=@CrescoParam[]`, `returns()=@CrescoReturn[]`, optional `target`/`routingParams` override.
- `@CrescoParam` / `@CrescoReturn` (nested): `name`, `type` (json schema type), `required`, `description`,
  `compressed` (rides as a gzip+base64 param).
- DTOs: `ParamDescriptor`, `ActionDescriptor`, `ServiceDescriptor`, `CapabilityDocument`
  (bundle id/version/summary + actions[] + services[] + packages[]).
- `CapabilityScanner`: `scanActions(Object executor)` (reflect annotations) + `scanOsgi(BundleContext)`
  (Export-Package headers + `getRegisteredServices()` interface names).
- `ToolSpecSerializer`: `toAnthropicTools(List<ActionDescriptor>)` → tool JSON + binding; name =
  `cresco_<namespace>_<action>`.
- `AbstractCapabilityInventoryPrinter`: implements Felix `InventoryPrinter` (JSON), prints the
  `CapabilityDocument`. Felix inventory API imported optional so the library still resolves without it.

## 3. Delivery surfaces (all three)
1. **`getcapabilities` EXEC** on every bundle — returns this bundle's `CapabilityDocument` JSON. Uniform,
   fabric-reachable, identical local + remote (same contract as `getmetrics`).
2. **`InventoryPrinter` (JSON)** per bundle — same data through the Felix inventory (shows in
   `/system/console/status` + the support-ZIP). Plus a bundles/services printer for the OSGi half.
3. **`getcapabilityinventory`** on the controller — aggregates every local bundle's `getcapabilities` +
   the controller tiers' own actions, `scope=node|region|global` with the concurrent bounded fan-out
   reused verbatim from `getmetricinventory`. Emits ONE tool catalog for the whole mesh.

## 4. OSGi class/service inventory
`CapabilityScanner.scanOsgi` walks `bundleContext.getBundles()`: `Bundle-SymbolicName`/version/
`Bundle-Description`, `Export-Package`, and `getRegisteredServices()` interface names → folded into
`CapabilityDocument.services/packages`. Answers "what classes/services does this node expose via OSGi".

## 5. Clients + MCP tool-runner (test)
- `clientlib` (Java) + `pycrescolib` (Python): `get_capability_inventory(scope=…)` → the tool catalog.
- **`run/tests/mcp_tool_runner.py` (TEST, not a bundle):** loads the catalog, exposes each descriptor as
  an MCP/LLM tool, and on invocation builds+routes the MsgEvent from the `binding` block and returns the
  parsed reply — proving an LLM can drive Cresco from the catalog alone.

## 6. Drift protection
A test (`CapabilityDrift`) reflects each executor's `switch` cases (via source scan or a declared case
list) and asserts a 1:1 match with `@CrescoAction` names — the inventory can never silently lie.

## 7. Phasing
P0 library package · P1 controller tiers · P2 plugins · P3 OSGi scan · P4 clients + drift test + docs ·
P5 MCP tool-runner test · Proof mesh test (pull catalog → validate tool JSON → invoke an action using
only its descriptor). All additive, gated by `capabilities_enabled` (default on for read; no behavior
change to existing actions).

## 8. Verification
`run/tests/capability_inventory_test.sh`: bring up global+region+agent, pull `getcapabilityinventory`
(scope=global), assert (a) it's well-formed Anthropic-tool JSON for every bundle+tier, (b) counts match
the known action surface, (c) the MCP runner invokes a described action (e.g. `listagents`) purely from
its descriptor and gets the documented return. No regression to metrics/security/startup suites.
</content>
