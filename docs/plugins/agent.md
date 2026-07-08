# agent — OSGi Host Bootstrap

The `agent` module builds the executable Cresco agent: an [Apache Felix](https://felix.apache.org/) OSGi host that launches the framework, provisions the embedded bundle set and the [controller](controller.md), and manages an ordered shutdown. It is the process you actually run.

!!! tip "The headline module"
    `agent` is the **final bundle** of the project — the artifact that composes every other module (`library`, `core`, `controller`, and the functional plugins) into one runnable uber-jar. Of the per-module repositories under [github.com/CrescoEdge](https://github.com/CrescoEdge), **[CrescoEdge/agent](https://github.com/CrescoEdge/agent) is the headline repo**: if you're looking for "the Cresco repo," it's this one.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/agent` |
| Main-Class | `io.cresco.main.AgentEngine` |
| Artifact | `agent-<version>.jar` (an **uber-jar**, not an OSGi bundle) |
| Java files | 6 |
| Key dependencies | `org.apache.felix.main`, `commons-configuration` |

Unlike every other Cresco module, the agent uber-jar is **not** itself an OSGi bundle — it is the Felix launcher. The component bundles (`logger`, `library`, `core`, `controller`, and the functional plugins) are embedded **inside** the uber-jar under `src/main/resources/` and installed from the classpath at boot.

## Key classes

| Class | Role |
|-------|------|
| `AgentEngine` | Process entry point. Quiets pax/commons logging, resolves the config file (`argv[1]` or `agent.ini`) into the `agentConfig` system property, and creates the singleton `HostApplication`. |
| `HostApplication` | The OSGi host. Launches Felix, sets framework properties, installs embedded + drop-in bundles, and registers the ordered shutdown hook. All boot logic runs in the constructor. |
| `HostActivator` | System-bundle `BundleActivator` that captures the framework `BundleContext` (used to enumerate/install bundles). |
| `Config` | Thread-safe config over a `Map` with `-D<param>` and `CRESCO_<param>` overrides (override wins and is cached back). Typed getters: `getBooleanParam`/`getDoubleParam`/`getIntegerParam`/`getLongParam`/`getStringParam`. |
| `FileConfig` | Reads INI files via commons-configuration (`agent.ini`, `version.ini`). |
| `AgentEngineShutdown` | Trivial helper whose `main` calls `System.exit(0)` (orphaned). |

## Boot sequence

`HostApplication`'s constructor performs the full boot:

1. **Build config.** Load `conf/agent.ini` (overridable with `-DagentConfig`) and `conf/version.ini` (`-DversionConfig`). Resolve `tmp_data` → a per-run `cresco_data_location`; derive `platform`/`environment`/`location` from environment → ini → hostname.
2. **Configure Felix.** `felix.log.level=1`; system-package substitution on; `FRAMEWORK_SYSTEMPACKAGES_EXTRA=sun.*,com.sun.*`; `bootdelegation=sun.*,com.sun.*,org.graalvm.*`; storage clean-on-first-init at `${cresco_data_location}/felix-cache`; HTTP port `-Dport` → `CRESCO_port` → `8080`; register `HostActivator`.
3. **Register the JVM shutdown hook**, which stops bundles in reverse order (console, jetty, base, controller, core, library, logger) and then deletes the data tree if `tmp_data`.
4. **Start Felix** — load the `FrameworkFactory`, `init()` then `start()`.
5. **Install drop-in bundles** — install/start each `.jar` in `externaljars/` (`-Dcresco_externaljars_dir`).
6. **Install the embedded bundles in order:**

   ```
   configadmin → logger.jar → metatype
       → osgi.cmpn / promise / function / component / http.servlet-api
       → (Jetty webconsole set when enable_console=true)
       → gogo.runtime / gogo.command → scr
       → healthcheck.api (INSTALLED, not started)
       → library.jar → core.jar
   ```
7. **Install the controller.** If `version.ini` names an external controller jar, install and start it, falling back to the embedded `controller.jar` if it does not reach `ACTIVE`.

The embedded `repo.jar` / `stunnel.jar` / `sysinfo.jar` / `wsapi.jar` are staged in the uber-jar but installed **at runtime as plugins** by the controller's `StaticPluginLoader`, not at boot. See [Overview](overview.md#how-the-agent-uber-jar-bootstraps-the-bundles).

## Versionless bundle staging

Component JARs are staged into `src/main/resources/` under stable, versionless names (e.g. `library.jar`, `controller.jar`) so that `HostApplication` can install them by a fixed classpath resource name regardless of the built version. `prebuild.sh` performs the staging (pulling published snapshots or locally-built bundles); the assembly plugin then packs them into the uber-jar. The optional `version.ini` mechanism lets an operator substitute an **external** controller jar at boot without rebuilding the uber-jar.

## Configuration read

| Key | Meaning |
|-----|---------|
| `agentConfig` / `versionConfig` | Override paths for `agent.ini` / `version.ini`. |
| `cresco_data_location` | Runtime state root (derived from `tmp_data` when set). |
| `port` / `CRESCO_port` | Felix HTTP port (default `8080`). |
| `enable_console` | Install the Jetty web console bundle set. |
| `cresco_externaljars_dir` | Directory of drop-in `.jar` bundles to install at boot. |
| `platform` / `environment` / `location` | Host identity metadata. |

See [Configuration](../getting-started/configuration.md) for the full parameter model.

## See also

- [Overview](overview.md) — the module/plugin model and boot ordering.
- [controller](controller.md) — what the controller brings up once it is `ACTIVE`.
- [logger](logger.md) — the first service started, so later bundles can log.
