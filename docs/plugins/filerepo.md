# filerepo — Distributed File/Artifact Repository

The `filerepo` plugin (bundle `io.cresco.filerepo`) stores, catalogs, and moves files and plugin JARs
across the fabric. Where [`repo`](repo.md) is specialized for plugin JARs, `filerepo` is a general
file repository: it watches a directory, tracks every file in an embedded catalog, and moves files of
**any size** across the mesh — inline for small files, byte-range dataplane streaming for large ones,
watched-directory sync between peers, and push-to-remote-repo.

## At a glance

| Fact | Value |
|------|-------|
| Module path | `code/filerepo` |
| Bundle symbolic name | `io.cresco.filerepo` |
| Catalog | Embedded Derby (file path / md5 / size / mtime) |
| Loaded | At runtime by the controller (default-on for a global). |
| Type | Functional plugin (`@Component` implementing `PluginService`). |

## What it does

A **repo** is a directory of files tracked in a Derby catalog. In *producer* mode `filerepo` scans a
`scan_dir` on an interval and keeps the catalog current; in *consumer* mode it receives files pushed
or synced from peers. Files move by one of three mechanisms:

- **Inline** (`getfile` / `getjar`) — the whole file rides back in one control-plane reply. Guarded by
  `max_inline_bytes` (default 512 KB) so an inline read can never exceed the wss frame limit; larger
  files must stream.
- **Streaming** (`streamfile`) — a byte-range is sent over the dataplane in chunks, each carrying a
  `transfer_id` + `seq_num` header the receiver reassembles by. Any size; the chunk size is clamped
  to a safe ceiling so a caller can't oversize a frame.
- **Directory sync** (`repolistin` / `repoconfirm`) — a producer's catalog diff is sent to a consumer,
  which pulls only the changed files.

Safeguards throughout: all catalog queries are parameterized (SQL-injection-safe); `putjar`/`getfile`
paths are canonicalized and confined to the repo dir (path-traversal-safe); md5 is verified on write
(a mismatch is deleted and rejected); the transfer pool is bounded and cleaned up on stop.

## Executor actions

`ExecutorImpl` handles EXEC actions ([`MsgEvent`](../api/msgevent.md) type `EXEC`). `status`/
`status_code` `10` = success; `object` returns are gzip+base64 compressed, `bytes` returns are base64.
Full table: [Plugin Actions › filerepo](../api/plugin-actions.md#filerepo-plugin).

| Action | Behaviour |
|--------|-----------|
| `getrepofilelist` | List catalog rows (path/md5/size/mtime), paginated via `limit`/`offset`. |
| `repolist` | Plugin/jar inventory of the repo dir + server identity. |
| `getfile` / `getjar` | Return a cataloged file / plugin jar inline (size-guarded). |
| `putjar` / `putfiles` | Write a jar (md5-verified) / land pushed files into a repo. |
| `putfilesremote` | Push a set of local files to another agent's repo. |
| `streamfile` / `streamfilecancel` | Stream a byte-range over the dataplane (any size) / cancel it. |
| `removefile` / `clearrepo` | Remove one file / wipe a repo (disk + catalog). |
| `repolistin` / `repoconfirm` | Directory-sync consumer side / handshake ack. |
| `getscandir` | Report the configured scan directory. |
| `getmetrics` | Central metrics — see [Observability](#observability). |

## Configuration read

| Key | Default | Meaning |
|-----|---------|---------|
| `scan_dir` | *(unset)* | Directory to watch/catalog (producer mode). |
| `filerepo_name` | *(unset)* | Logical repo name for mesh discovery/sync. |
| `enable_scan` | `true` | Enable the periodic scan. |
| `scan_delay` / `scan_period` | `5000` / `15000` ms | First-scan delay / scan interval. |
| `repo_dir` | `filerepo` | Repo directory for on-demand/consumer storage. |
| `max_inline_bytes` | `524288` (512 KB) | Max file size returned inline by `getfile`/`getjar`. |
| `stream_buffer_size` | `262144` (256 KB) | Default streamfile chunk size (hard-capped at 768 KB). |
| `transfer_threads` | `8` | Max concurrent streamfile transfers (bounded pool). |

See [Configuration](../getting-started/configuration.md).

## Observability

- **Metrics** — `getmetrics` exposes `MeasurementEngine` gauges `filerepo.files.count` (catalog size)
  and `filerepo.active.transfers` (in-flight `streamfile` count); aggregated by the controller's
  `getmetricinventory`. See [Metrics & Measurements](../architecture/metrics.md#plugin-filerepo).
- **Health** — registers the `filerepo` Felix HealthCheck (tag `local`), discovered by the
  controller's health executor: reports the catalog size and verifies the repo dir is writable
  (`WARN` if not). See [Health & State](../architecture/health.md#plugin-checks).

## See also

- [repo](repo.md) — the plugin-JAR-specialized sibling repository.
- [Plugin Actions](../api/plugin-actions.md#filerepo-plugin) — the full action table.
- [Data Plane](../architecture/dataplane.md) — how `streamfile` chunks ride the mesh.
