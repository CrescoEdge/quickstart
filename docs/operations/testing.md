# Testing

Cresco validates behavior primarily through **integration harnesses** under `run/tests/` that stand up a
real fabric (global + regions + agents), exercise it, and assert outcomes — plus a set of Python
performance harnesses. This page maps the suites so you know what proves what.

## Running a suite

Most suites are self-contained shell scripts that build/launch nodes, run assertions, and tear down:

```bash
cd run/tests
./tenant_namespacing_test.sh      # e.g. tenant isolation under namespacing
```

They source `run/cresco.env` for the shared secrets/keys, launch nodes into `run/nodes/`, and print
`PASS`/`FAIL` lines with a final tally. Rebuild the agent jar first (`./build.sh agent`) if you changed code.

## Security & tenancy

| Suite | Proves |
|-------|--------|
| `security_foundation_test.sh` | identity ⇄ DN, message signing (accept/reject), chain-trust to a region CA |
| `mtls_binding_test.sh` | mutual-TLS binds the principal from the client-cert DN (anti-spoof) |
| `regional_ca_mtls_test.sh` | regional-CA enrollment + fabric-wide mutual-TLS federation |
| `tenant_isolation_mesh_test.sh` | cross-tenant deny at every broker in a federated mesh (namespacing off) |
| `tenant_namespacing_test.sh` | end-to-end tenant namespacing: fabric forms + zero self-denial; TENANT confined; SUPERUSER god view |

## Health, state & recovery

| Suite | Proves |
|-------|--------|
| `exhaustive_health_test.sh` | health checks, grace/sticky windows, parent-link loss → recovery |
| `b1_startup_race_test.sh` | TLS trust-ready connect gate (no PKIX race on same-host bring-up) |
| `mesh_massive_test.sh` | large-mesh bring-up and fault isolation |

## Capability & metrics

| Suite | Proves |
|-------|--------|
| `capability_inventory_test.sh` | fabric-wide `getcapabilityinventory` (all actions self-described) |
| `metrics_unification_test.sh` | `getmetricinventory` returns every plugin's metrics at node/region/global scope |

## Performance

| Harness | Measures |
|---------|----------|
| `dataplane_perf.py` / `dataplane_internode.py` / `dataplane_aggregate.py` | data-plane throughput/latency |
| `stunnel_perf.py` / `stunnel_direct.py` | tunnel throughput |
| `msgevent_perf.py` | control-plane message rate |
| `adaptive_net_suite.sh` | network auto-tuning / link-metric behavior |
| `run_speedtests.sh` | aggregate speed suite (see `SPEEDTEST_RESULTS.md`) |

## Writing a new integration test

Follow the pattern of the existing scripts: source `cresco.env`, define a `launch()` helper that runs the
agent jar with role/port/secret flags into a per-node dir, wait for readiness (grep the node log for a
marker such as `Starting Bridge`), assert with a small `ck()` PASS/FAIL helper, then `pkill` and exit with
the fail count. Keep every new flag **default-off** so the suite stays green with the feature both off and
on.
