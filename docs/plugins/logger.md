# logger — Logging Bootstrap

The `logger` module (bundle `io.cresco.logger`) provisions the logging subsystem for the Cresco agent. It is installed and started immediately after the OSGi configuration-admin service so that every later bundle can log through it.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/logger` |
| Bundle symbolic name | `io.cresco.logger` |
| Java files | 1 |
| Loaded | First service the agent starts (right after `configadmin`). |
| Backing stack | Pax Logging 2.x (log4j2 core, SLF4J 2.x) via ConfigAdmin. |

## What it does

The single `Activator`:

1. Installs and starts `org.osgi.service.cm` (ConfigAdmin), `pax-logging-api`, and `pax-logging-log4j2`.
2. Writes the ConfigAdmin PID `org.ops4j.pax.logging` with a log4j2 configuration:
   - Root level from `-Droot_log_level` (default `INFO`).
   - A **Console** appender and a **rolling File** appender at `${cresco_data_location}/cresco-logs/main.log` (50 MB / daily rotation, gzip-compressed, keep 10).
   - Quieting for noisy third-party loggers.

Because logging is installed before `library`, `core`, and the [controller](controller.md), every subsequent bundle obtains a working SLF4J logger the moment it starts.

## Relationship to plugin logging

Plugins do not use this bundle directly. They log through the `CLogger` interface (from [library](library.md)) obtained from their `PluginBuilder`; the [controller](controller.md)'s `CLoggerImpl` bridges those calls onto SLF4J (and mirrors them onto the [data plane](../architecture/dataplane.md) for the [wsapi](wsapi.md) log-streamer). This `logger` bundle provides the underlying SLF4J/log4j2 backend that all of those calls ultimately reach.

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `root_log_level` | `INFO` | Root log level. |
| `cresco_data_location` | — | Base directory for `cresco-logs/main.log`. |

See [Configuration](../getting-started/configuration.md).

## See also

- [agent](agent.md) — boot ordering (logger is started right after configadmin).
- [library](library.md) — the `CLogger` interface plugins log through.
