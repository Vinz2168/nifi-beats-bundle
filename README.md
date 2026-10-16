# nifi-beats-bundle

**ListenBeats for Apache NiFi 2.x.** A community-maintained, standalone port
of Apache NiFi's `ListenBeats` processor (a listener for the
Elasticsearch/Logstash **Beats** protocol, e.g. Filebeat/Metricbeat) to the
**NiFi 2.12.0** API. `ListenBeats` was deprecated in NiFi 1.x and fully
removed from `nifi-extension-bundles` as of the 2.12.0 tag, so it no longer
ships with NiFi core. This bundle rebuilds it as an external NAR, built
against published NiFi 2.12.0 artifacts on Maven Central, so it can be
dropped into any NiFi 2.12.x installation.

> This project is not affiliated with, endorsed by, or released by The Apache
> Software Foundation. Apache, Apache NiFi and NiFi are trademarks of The
> Apache Software Foundation.

## Components

### Processors

- [`ListenBeats`](#listenbeats)

### ListenBeats

Listens on a TCP port for the Beats wire protocol (version 2: `JSON`,
`WINDOW_SIZE` and `COMPRESSED` frames), acknowledges each batch, and emits
one FlowFile per message.

**Properties**

| Property | Required | Default | Description |
|---|---|---|---|
| Port | yes | | The TCP port to listen on. Supports environment-scoped Expression Language. |
| SSL Context Service | no | | Controller service implementing `org.apache.nifi.ssl.SSLContextProvider`. When set, TLS is enabled for socket connections. |
| Client Authentication | no | `REQUIRED` | Client authentication policy when TLS is enabled (only shown when an SSL Context Service is set). |

The processor also exposes NiFi's shared `ListenerProperties`: Local Network
Interface, Receive Buffer Size, Max Size of Message Queue, Max Size of Socket
Buffer, Character Set, Max Batch Size, Batching Message Delimiter and
Worker Threads, with the same meaning as in `ListenTCP`.

**Relationships**

- `success`: messages received successfully.

**Attributes written**

- `beats.sender`: address of the sending Beats client
- `beats.port`: port the message was received on
- `beats.sequencenumber`: Beats sequence number (single-message batches only)
- `mime.type`: `application/json`

**Content**: the raw JSON payload sent by the Beats client.

## Compatibility

| Bundle version | NiFi version | Java | Status |
|---|---|---|---|
| 2.12.0-2 | 2.12.x | 21 | Supported |
| 2.12.0-1 | 2.12.x | 21 | Superseded: flows imported from NiFi 1.x are invalid (see [Migrating flows](#migrating-flows-from-nifi-1x)) |

**Versioning policy**: the bundle version tracks the NiFi release it is built
against (`2.12.0` means built against the NiFi 2.12.0 API). The numeric
suffix (`-1`, `-2`, ...) marks a packaging-only or bug-fix revision of the
bundle for the same NiFi release. A new NiFi minor release that requires a
rebuild gets a new bundle version (e.g. `2.13.0-1`).

## Maintenance

Maintained on a best-effort basis by [@Vinz2168](https://github.com/Vinz2168).
Please report bugs via [GitHub Issues](https://github.com/Vinz2168/nifi-beats-bundle/issues)
and include:

- bundle version and NiFi version
- Java version and OS
- Beats client and version (e.g. Filebeat 9.5.4) and its output configuration
- whether TLS / client authentication is enabled
- relevant `nifi-app.log` excerpts and, if possible, steps to reproduce

## Install

1. Download `nifi-beats-nar-2.12.0-2.nar` from the
   [releases page](https://github.com/Vinz2168/nifi-beats-bundle/releases).
2. Copy it into your NiFi installation's `lib/` directory and restart NiFi,
   or copy it into the `extensions/` directory to have NiFi autoload it
   without a restart.
3. `ListenBeats` will appear in the "Add Processor" dialog.

Or build it yourself (Java 21+, Maven 3.9+):

```
mvn clean verify
```

produces `nifi-beats-nar/target/nifi-beats-nar-2.12.0-2.nar`.

## Migrating flows from NiFi 1.x

The Java package (`org.apache.nifi.processors.beats`), the class name
(`ListenBeats`) and the processor's own property names (`Port`,
`SSL_CONTEXT_SERVICE`, `Client Auth`) are unchanged from the original
processor, so flows exported from NiFi 1.x resolve to this processor without
edits.

The Maven coordinates differ (`io.github.vinz2168:nifi-beats-nar` instead
of `org.apache.nifi:nifi-beats-nar`), but NiFi selects this bundle
automatically when it is the only one providing `ListenBeats`.

Two shared listener properties come from NiFi's `ListenerProperties` class,
whose property names changed in NiFi 2.x (after 2.0.0). Like NiFi's own
`ListenTCP`, this processor renames them on flow import via
`migrateProperties()` (since 2.12.0-2), keeping their values:

| NiFi 1.x property name | NiFi 2.x property name |
|---|---|
| `Max Number of TCP Connections` | `Worker Threads` |
| `Message Delimiter` | `Batching Message Delimiter` |

Bundle versions up to 2.12.0-1 lack this migration: a 1.x flow imported
with them is invalid ("not a supported property") and Worker Threads falls
back to its default.

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

## Testing

- `mvn clean verify` — unit tests (`ListenBeatsTest`) exercise single
  messages, windowed batches, and DEFLATE-compressed batches over a real
  socket, plus the 1.x property migration.

End-to-end tests ran against real NiFi 2.12.0 instances (`apache/nifi:2.12.0`)
fed by an actual **Filebeat 9.5.4** container
(`docker.elastic.co/beats/filebeat:9.5.4`) over the Logstash/Beats output:

- **Plain TCP**: ACKs, FlowFile attributes and JSON content confirmed.
- **TLS** with `StandardSSLContextService`:
  - Client Authentication `REQUIRED` with a trusted client certificate: delivered.
  - `REQUIRED` with no client certificate, or one from an untrusted CA: rejected.
  - Client Authentication `NONE`: delivered.
  - A plaintext client connecting to the TLS port: rejected.
- **Local Network Interface**: when the property names a second interface, the
  listener binds only to that interface's address. A connection to another
  interface's address is refused, and an unknown interface name fails
  validation. NiFi reads the list of valid interfaces at startup, so an
  interface added while NiFi is running is accepted only after a restart.
- **Migration from NiFi 1.28.1**: a flow built in a real NiFi 1.28.1 instance
  (with the original ASF `nifi-beats-nar-1.28.1`) used ListenBeats with TLS
  via `StandardRestrictedSSLContextService`, Client Auth `NONE`, Max Batch
  Size 7 and 3 worker threads. Its `flow.json.gz` was loaded at first
  startup of a fresh NiFi 2.12.0:
  - the processor was resolved to this bundle and started automatically;
  - all values were kept, including those of the renamed properties;
  - batching matched 1.28.1 (20 events → 7+7+6).
  Uploading the same process group as a flow definition JSON also migrates
  correctly. Sensitive values such as keystore passwords are not part of a
  flow definition and must be re-entered.

## License

Licensed under the [Apache License 2.0](LICENSE), inherited from the original
Apache NiFi source (`org.apache.nifi.processors.beats`). See [NOTICE](NOTICE)
for attribution.
