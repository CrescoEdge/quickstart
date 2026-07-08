# wsapi — WebSocket API

The `wsapi` plugin (bundle `io.cresco.wsapi`) is the fabric's external client entrypoint. It embeds a Netty HTTPS/WSS server that exposes three authenticated WebSocket endpoints — control, data plane, and log streaming — over `wss://…:8282`. This is the endpoint the [Java](../clients/java.md), [Python](../clients/python.md), and [C++ / Arduino](../clients/cpp.md) client libraries connect to.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/wsapi` |
| Bundle symbolic name | `io.cresco.wsapi` |
| Java files | 10 |
| Loaded | At runtime by the controller's `StaticPluginLoader` (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |
| Server | Embedded **Netty** (`NettyWsServer`) on NIO event loops (1 boss + default worker group). |
| Default port | `8282` (`wss://`). |

## Endpoints

All three endpoints sit behind the `WsRouter` handler, which runs first (right after HTTP aggregation) and authenticates the WebSocket upgrade: a request is rejected with `401 UNAUTHORIZED` unless its `cresco_service_key` HTTP header matches the configured key. It then routes the upgrade to the handler for the requested path.

| Handler | Path | Purpose |
|----------|------|---------|
| `ApiSocketWsHandler` | `/api/apisocket` | Control-plane / RPC. Deserializes `{message_info, message_payload}`, builds the matching [`MsgEvent`](../api/msgevent.md) by `message_type` (e.g. `global_controller_msgevent`, `global_agent_msgevent`, `global_plugin_msgevent`, `kpi_msgevent`, `agent_msgevent`, `plugin_msgevent`), and either `sendRPC` (returning response params as JSON) or `msgOut`, per the `is_rpc` flag. |
| `DataPlaneWsHandler` | `/api/dataplane` | Bridges a client WebSocket to the global JMS [data plane](../architecture/dataplane.md). The first message subscribes (creates a JMS listener from `ident_key`/`ident_id`/`io_type_key`/`output_id`/`input_id`, or a raw `stream_query`); subsequent text → publish and whole-binary → `BytesMessage` on `TopicType.GLOBAL` (with optional `seq_num`/`transfer_id` framing). |
| `LogStreamerWsHandler` | `/api/logstreamer` | Subscribes to agent logger data-plane messages (filtered by session) and RPCs `setloglevel` to set remote agent log levels. |

## Key classes

| Class | Role |
|-------|------|
| `Plugin` | Builds and manages the `NettyWsServer`; ensures a self-signed `ws.keystore` (BouncyCastle RSA-2048, `SHA256WithRSA`) in the plugin data directory; reads the port/buffer config and starts the server. |
| `NettyWsServer` | The Netty bootstrap: NIO `MultiThreadIoEventLoopGroup`s (1 boss + default workers), the TLS + WebSocket pipeline, and adaptive receive buffers. |
| `WsRouter` | First pipeline handler after HTTP aggregation: authenticates the `cresco_service_key` header (`401` on mismatch) and routes the upgrade to the per-path handler. |
| `ApiSocketWsHandler` / `DataPlaneWsHandler` / `LogStreamerWsHandler` | The three endpoint handlers (control-plane RPC, data-plane bridge, log streamer). |
| `PluginExecutor` | EXEC action `globalinfo` (sets `global_region`/`global_agent`). |
| `StreamInfo` | Per-stream data holder (in the `websockets` package). |
| `Activator` | Routes third-party logging through SLF4J. |

## TLS and performance tuning

`NettyWsServer` is tuned for high-throughput streaming:

- TLS via an SSL context whose provider is selectable with `wsapi_ssl_provider` (default `JDK`).
- Adaptive receive-buffer allocation (`AdaptiveRecvByteBufAllocator`, 2 KB … 64 KB … up to `wsapi_read_chunk_bytes`) and `wsapi_socket_buffer_bytes` (4 MB) socket buffers, with a `wsapi_write_high_water_bytes` (2 MB) write watermark for backpressure.
- **`permessage-deflate` is disabled** (`WsRouter` sets `allowExtensions(false)`) — WebSocket compression was the single biggest data-plane CPU bottleneck, so it is deliberately off. See [Data Plane](../architecture/dataplane.md) and [Metrics & Measurements](../architecture/metrics.md).

## Authentication and identity

Clients authenticate with the `cresco_service_key` HTTP header on the WebSocket upgrade. On the return path, external identity — a client's region/agent/plugin — is read from the TLS peer certificate DN. See [Security & Identity](../architecture/security.md).

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `wsapi_port` | `8282` | WSS/HTTPS listen port. |
| `wsapi_keystore_password` | `cresco` | Password for the auto-generated `ws.keystore`. |
| `cresco_service_key` | — | Shared-secret auth header value. |
| `wsapi_ssl_provider` | `JDK` | Netty SSL provider (`JDK` or `OPENSSL`). |
| `dataplane_io_buffer_bytes` | `65536` | Data-plane I/O buffer size (64 KB). |
| `wsapi_read_chunk_bytes` | `262144` | Max adaptive read chunk (256 KB). |
| `wsapi_socket_buffer_bytes` | `4194304` | Socket send/receive buffer (4 MB). |
| `wsapi_write_high_water_bytes` | `2097152` | Write-buffer high-water mark for backpressure (2 MB). |

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
