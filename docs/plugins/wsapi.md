# wsapi — WebSocket API

The `wsapi` plugin (bundle `io.cresco.wsapi`) is the fabric's external client entrypoint. It embeds a Jetty 12 HTTPS/WSS server that exposes three authenticated WebSocket endpoints — control, data plane, and log streaming — over `wss://…:8282`. This is the endpoint the [Java](../clients/java.md) and [Python](../clients/python.md) client libraries connect to.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/wsapi` |
| Bundle symbolic name | `io.cresco.wsapi` |
| Java files | 9 |
| Loaded | At runtime by the controller's `StaticPluginLoader` (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Server | Embedded Jetty 12 (ee10) on JDK 21 virtual threads. |
| Default port | `8282` (`wss://`). |

## Endpoints

All three endpoints sit behind a shared-secret `AuthFilter`: a request is rejected with `401` unless its `cresco_service_key` HTTP header matches the configured key.

| Endpoint | Path | Purpose |
|----------|------|---------|
| `APISocket` | `/api/apisocket` | Control-plane / RPC. Deserializes `{message_info, message_payload}`, builds the matching [`MsgEvent`](../api/msgevent.md) by `message_type` (e.g. `global_controller_msgevent`, `global_agent_msgevent`, `global_plugin_msgevent`, `kpi_msgevent`, `agent_msgevent`, `plugin_msgevent`), and either `sendRPC` (returning response params as JSON) or `msgOut`, per the `is_rpc` flag. |
| `APIDataPlane` | `/api/dataplane` | Bridges a client WebSocket to the global JMS [data plane](../architecture/dataplane.md). The first message subscribes (creates a JMS listener from `ident_key`/`ident_id`/`io_type_key`/`output_id`/`input_id`, or a raw `stream_query`); subsequent text → publish and whole-binary → `BytesMessage` on `TopicType.GLOBAL` (with optional `seq_num`/`transfer_id` framing). |
| `APILogStreamer` | `/api/logstreamer` | Subscribes to agent logger data-plane messages (filtered by session) and RPCs `setloglevel` to set remote agent log levels. |

## Key classes

| Class | Role |
|-------|------|
| `Plugin` | Builds and manages the Jetty server; holds `public static PluginBuilder pluginBuilder` so Jetty-instantiated endpoints/filter can reach the plugin. Ensures a self-signed `ws.keystore` (BouncyCastle RSA-2048); registers `APISocket`/`APIDataPlane`/`APILogStreamer`. |
| `AuthFilter` | Servlet `Filter` on `/*` enforcing the `cresco_service_key` header (`401` on mismatch). |
| `PluginExecutor` | EXEC action `globalinfo` (sets `global_region`/`global_agent`). |
| `SessionInfo` / `StreamInfo` | Per-session/stream data holders. |
| `Activator` | Routes third-party logging through SLF4J. |

## TLS and performance tuning

The `Plugin` configures Jetty for high-throughput streaming:

- TLSv1.2 / TLSv1.3, direct-buffer SSL.
- 4 MB accepted `SO_RCVBUF`/`SO_SNDBUF`.
- `SniHostCheck=false`.
- **`permessage-deflate` is unregistered** — WebSocket compression was the single biggest data-plane CPU bottleneck, so it is deliberately disabled. See [Data Plane](../architecture/dataplane.md) and [Metrics & Measurements](../architecture/metrics.md).

## Authentication and identity

Clients authenticate with the `cresco_service_key` HTTP header on the WebSocket upgrade. On the return path, external identity — a client's region/agent/plugin — is read from the TLS peer certificate DN. See [Security & Identity](../architecture/security.md).

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `wsapi_port` | `8282` | WSS/HTTPS listen port. |
| `wsapi_keystore_password` | `cresco` | Password for the auto-generated `ws.keystore`. |
| `cresco_service_key` | — | Shared-secret auth header value. |
| `dataplane_io_buffer_bytes` | `262144` | Data-plane I/O buffer size. |
| `wsapi_dp_selftest*` | off | Diagnostic self-test knobs. |

See [Configuration](../getting-started/configuration.md).

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `wsapi.dataplane.connections`,
  `wsapi.dataplane.bytes`, and `wsapi.dataplane.messages` (process-wide dataplane ingress counters),
  aggregated by `getmetricinventory`. See
  [Metrics & Measurements › wsapi](../architecture/metrics.md#plugin-wsapi).
- **Health** — registers the `wsapi` Felix HealthCheck (tag `local`), **only after the Netty server
  binds**, so an OK reports the live wss port. See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [Client Libraries Overview](../clients/overview.md) · [Java (clientlib)](../clients/java.md) · [Python (pycrescolib)](../clients/python.md)
- [Data Plane](../architecture/dataplane.md) — what `/api/dataplane` bridges to.
- [Security & Identity](../architecture/security.md) — the auth header and TLS-cert identity model.
