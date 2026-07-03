# Data Plane

The data plane is Cresco's **high-throughput streaming** channel. Where the [control plane](messaging.md)
moves addressed MsgEvents over queues, the data plane moves application data over ActiveMQ **topics**
(publish/subscribe), implemented by `DataPlaneServiceImpl` and exposed to plugins as the
`DataPlaneService` library interface.

## Topics

There are three topic scopes, matching the hierarchy:

| Topic type | Default name | Reaches |
|------------|--------------|---------|
| `AGENT` | `agent.event` | subscribers on the same agent |
| `REGION` | `region.event` | subscribers within the region |
| `GLOBAL` | `global.event` | subscribers across the whole federated mesh |

A publisher sends to a scope; every subscriber to that scope with a matching **selector** receives the
message. Cross-region delivery on `global.event` rides the same broker [federation](messaging.md#federation-network-of-brokers)
bridges as the control plane (demand-forwarded).

Under [tenant namespacing](tenancy.md) these topics are qualified per tenant —
`T.<tenant>.{agent,region,global}.event` — so a tenant's streams are isolated at every broker while
same-tenant cross-region flow still works.

## Using it (from a plugin)

```java
DataPlaneService dp = controllerEngine.getDataPlaneService();

// subscribe
String id = dp.addMessageListener(TopicType.GLOBAL, (Message m) -> {
    String body = ((TextMessage) m).getText();
    // ...
}, "myselector = 'value'");

// publish
dp.sendMessage(TopicType.GLOBAL, dp.createTextMessage("payload"));
```

Selectors are JMS message selectors, letting subscribers filter by message properties without the broker
delivering everything.

## Streaming to external clients

The [wsapi](../plugins/wsapi.md) plugin bridges the data plane to external [clients](../clients/overview.md)
over WebSockets: a client opens a data-plane stream and receives topic messages as text or binary frames.
This powers dashboards and remote consumers.

## Sharding (throughput)

For very high throughput a topic type can be **sharded** into `N` shard-topics (`global.event.0..N-1`) via
`dataplane_shards`, so ActiveMQ's per-destination demand-forwarding spreads cross-node traffic across
parallel bridge connectors. With `dataplane_parallel_connections`, each shard rides its own dedicated
broker connection (parallel sockets) rather than multiplexing over one pooled session. Defaults keep the
original single-topic behavior. See the [Configuration reference](../reference/configuration.md).

## File transfer

Large payloads (files) are chunked and streamed as a sequence of `BytesMessage` parts through the broker,
reassembled and MD5-validated by the receiver. This is the [BULK](messaging.md#quality-of-service) QoS
tier — persistent but lowest priority, so it never delays the control plane.

## Complex event processing

`DataPlaneServiceImpl` embeds a lightweight CEP engine (`CEPEngine`) that can run windowed/aggregating
queries over a stream (e.g. time-batched averages), producing derived streams — useful for in-fabric
telemetry roll-ups without an external stream processor.
