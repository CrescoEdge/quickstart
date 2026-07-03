# Installation & Build

Cresco builds from source into a single runnable **agent uber-jar**. This page covers requirements,
the build, and the resulting artifacts.

## Requirements

| Requirement | Version / note |
|-------------|----------------|
| **JDK** | 21 (every module targets `<jdk>21</jdk>`) |
| **Maven** | 3.8+ |
| **Python** | 3.8+ (only for the [Python client](../clients/python.md)) |
| **OS** | Linux / macOS (developed on both) |

## Build from source

Cresco is a set of OSGi bundles — one module (and one git repository) per directory under `code/`. The
`build.sh` driver builds each in dependency order and stages the runnable jar into `run/`.

```bash
./build.sh            # build everything: components + clients + agent uber-jar
./build.sh components # just the OSGi bundles (logger, library, core, repo, sysinfo, wsapi, stunnel, controller)
./build.sh agent      # stage bundles + assemble the agent uber-jar into run/
./build.sh clients    # build the Java clientlib + set up the Python venv
```

!!! warning "The `bundle:bundle` gotcha"
    Each module declares `<packaging>jar</packaging>`, and the `maven-bundle-plugin` only rewrites the
    artifact into a **real OSGi bundle** when its `bundle:bundle` goal runs explicitly. The build recipe
    is therefore `mvn clean package bundle:bundle` — a bare `mvn install` produces a non-bundle jar that
    Felix cannot start. Always build through `build.sh` (or run `mvn ... bundle:bundle` yourself).

Build order (from `build.sh`): base layer `logger`, `library` (no Cresco deps) → plugin layer `core`,
`repo`, `sysinfo`, `wsapi`, `stunnel`, `controller` → then the `agent` uber-jar last.

## Artifacts

| Artifact | Location | What it is |
|----------|----------|-----------|
| `agent-1.3-SNAPSHOT.jar` | `run/` | The runnable Cresco node (Felix host + all bundles). This is what you launch. |
| Per-module bundles | `~/.m2/...` | Installed to the local Maven repo; also staged version-less into `agent/src/main/resources/`. |
| `clientlib` jar | `code/clientlib/target/` | The external [Java SDK](../clients/java.md) (shaded). |
| `pycrescolib` | `run/venv/` | The [Python SDK](../clients/python.md), installed editable into a venv. |

The bootstrap (`HostApplication`) and `StaticPluginLoader` reference the staged bundles by **version-less**
names (e.g. `controller.jar`), so a version bump never touches runtime Java — only the build version.

## Run hygiene

Each node keeps on-disk state (Felix cache, Derby DB, ActiveMQ journal) under a `cresco-data/`
directory in its working directory. **Two nodes on the same host must never share a working directory** —
the launch scripts give each node its own `run/nodes/<region>-<agent>/`.

!!! note "Wipe state between builds/runs"
    Stale Derby/broker state from a previous run can cause spurious plugin/state errors. Between runs (and
    especially after a rebuild) remove `run/cresco-data` and `run/nodes`:
    ```bash
    rm -rf run/cresco-data run/nodes
    ```

Next: **[Quickstart](quickstart.md)** to launch a mesh and connect a client.
