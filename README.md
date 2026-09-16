# nifi-beats-bundle

Standalone port of Apache NiFi's `ListenBeats` processor (a listener for the
Elasticsearch/Logstash **Beats** protocol, e.g. Filebeat/Metricbeat) to the
**NiFi 2.12.0** API.

`ListenBeats` was deprecated in NiFi 1.x and fully removed from
`nifi-extension-bundles` as of the 2.12.0 tag, so it no longer ships with
NiFi core. This bundle rebuilds it as an external NAR, built against
published NiFi 2.12.0 artifacts on Maven Central, so it can be dropped into
any NiFi 2.12.0 installation.

## What it does

Listens on a TCP port for the Beats wire protocol (version 2: `JSON`,
`WINDOW_SIZE` and `COMPRESSED` frames), acknowledges each batch, and emits
one FlowFile per message with:

- Relationship: `success`
- Attributes: `beats.sender`, `beats.port`, `beats.sequencenumber` (single-message
  batches only), `mime.type` (`application/json`)
- Content: the raw JSON payload sent by the Beats client

## Install

1. Download `nifi-beats-nar-2.12.0.nar` from the
   [releases page](https://github.com/Vinz2168/nifi-beats-bundle/releases).
2. Copy it into your NiFi installation's `lib/` directory.
3. Restart NiFi. `ListenBeats` will appear in the "Add Processor" dialog.

Or build it yourself:

```
mvn clean verify
```

produces `nifi-beats-nar/target/nifi-beats-nar-2.12.0.nar`.

## Porting notes: differences from the original 1.28.1 processor

The protocol codec itself (`protocol/`, `handler/`, `server/` packages) is
carried over unchanged. Two real API breaks between NiFi 1.28 and 2.12
required code changes:

- **`PORT` property**: `ListenerProperties.PORT` was removed from the shared
  listener properties class in 2.x. This processor now defines its own local
  `PORT` property (same property name, `"Port"`, so existing flow
  definitions keep working) and implements
  `org.apache.nifi.components.listen.ListenComponent` /
  `getListenPorts(ConfigurationContext)`, the same pattern NiFi 2.x's own
  `ListenTCP`/`ListenUDP` use to register with the framework's listen-port
  registry.
- **SSL Context Service**: `RestrictedSSLContextService`/`SSLContextService`
  is deprecated for removal in 2.12. The `SSL Context Service` property now
  identifies `org.apache.nifi.ssl.SSLContextProvider` instead. The property
  name is unchanged, but a flow migrating from 1.28.1 with TLS enabled will
  need a controller service that implements `SSLContextProvider` (e.g. NiFi
  2.x's `StandardSSLContextService`) assigned to it.

Not verified: the TLS/mutual-auth path end-to-end, binding to a non-default
network interface, and NiFi's own flow.json import/migration tooling for a
flow exported from a real 1.28.1 instance.

## Testing

- `mvn clean verify` — unit tests (`ListenBeatsTest`) exercise single
  messages, windowed batches, and DEFLATE-compressed batches over a real
  socket.
- End-to-end: deployed the built NAR into a real NiFi 2.12.0 instance and
  fed it traffic from an actual **Filebeat 9.5.4** container
  (`docker.elastic.co/beats/filebeat:9.5.4`) over the Logstash/Beats output
  — confirmed ACKs, FlowFile attributes, and JSON content.

## License

Apache License 2.0, inherited from the original Apache NiFi source
(`org.apache.nifi.processors.beats`).
