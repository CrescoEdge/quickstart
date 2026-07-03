# core — Controller & JVM Lifecycle Service

The `core` module (bundle `io.cresco.core`) is a small framework service that registers the `CoreState` OSGi service, giving the fabric a way to stop, restart, or update the [controller](controller.md) and the JVM itself.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/core` |
| Bundle symbolic name | `io.cresco.core` |
| Java files | 2 |
| Loaded | At boot, after `library` and before `controller`. |
| Service published | `CoreState` (interface defined in [library](library.md)) |

## Key classes

| Class | Role |
|-------|------|
| `Activator` | OSGi `BundleActivator` that publishes `CoreStateImpl` as the `CoreState` service. On stop, it disables the controller's `AgentServiceImpl` Declarative-Services component via `ServiceComponentRuntime` (waiting up to ~10s). |
| `CoreStateImpl` | The `CoreState` implementation. Every action runs on a background thread. |

## CoreState operations

`CoreStateImpl` implements the `CoreState` interface from [library](library.md):

| Method | Behaviour |
|--------|-----------|
| `updateController(jarPath)` | Hot-update the controller: stop → uninstall → install → start the new controller bundle. |
| `stopController()` | Stop the controller bundle. |
| `restartController()` | Stop then start the controller bundle. |
| `restartFramework()` | Update the OSGi system bundle (bundle 0), restarting the framework. |
| `killJVM()` | Stop the controller, then `System.exit(1)`. |

These operations are how the mesh performs remote lifecycle control: the [controller](controller.md)'s `AgentExecutor` exposes `controllerupdate`, `stopcontroller`, `restartcontroller`, `restartframework`, and `killjvm` CONFIG actions that delegate to this service, driven remotely through [wsapi](wsapi.md) and the client libraries.

## See also

- [controller](controller.md) — the target of these lifecycle operations.
- [library](library.md) — where the `CoreState` interface is defined.
