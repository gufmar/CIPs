# Relay Monitoring Publication Specification

This document describes the configuration and publication format for relay-based monitoring data. Relay nodes act as the publicly reachable publication endpoint and may optionally proxy monitoring data from internal block-producing nodes.

The publication system supports:

- **open publications** (plaintext)
- **encrypted publications** (recipient-specific)
- **optional proxying of monitoring data from internal producer nodes**

The relay constructs the publication envelope, while payload fields originate from the node’s internal monitoring dataset.

---



# 1. Configuration Template

Relay nodes define monitoring publications inside their node configuration.

## Example configuration

```json
{
  "publications": {
    "localRootProxy": true,
    "items": [
      {
        "type": "open",
        "fields": [
          "generated_at",
          "node_name",
          "node_version_major"
        ]
      },
      {
        "type": "encrypted",
        "recipient_name": "Monitor A",
        "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
        "fields": [
          "generated_at",
          "node_name",
          "node_version_major",
          "node_version_minor",
          "node_version_hotfix",
          "cores",
          "memory",
          "state_hash",
          "node_type",
          "slot_height",
          "block_height",
          "network"
        ]
      }
    ]
  }
}
```



## Configuration elements



### `publications.localRootProxy`

Boolean flag enabling proxy monitoring for producer nodes.

```
true  → relay queries proxied producer nodes  
false → only relay self-publications are generated
```



### `publications.items`

Array of publication templates.

Each publication template defines:


| field                  | required               | description                       |
| ---------------------- | ---------------------- | --------------------------------- |
| `type`                 | yes                    | `"open"` or `"encrypted"`         |
| `fields`               | yes                    | list of allowed field identifiers |
| `recipient_public_key` | required for encrypted | monitoring entity public key      |
| `recipient_name`       | optional               | human-readable label              |


The `fields` array only selects which fields the node should include.  
Operators cannot define custom JSON structures or placeholder variables.

---



# 2. Topology Extension for Proxied Producers

Relay nodes may proxy monitoring publications from internal producer nodes configured in `topology.json`.

## Example

```json
{
  "bootstrapPeers": [],
  "localRoots": [
    {
      "accessPoints": [
        {
          "address": "10.10.123.01",
          "port": 6000,
          "description": "CLIO1",
          "monitoring-proxied": true,
          "poolBech32Id": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi"
        }
      ],
      "advertise": false,
      "trustable": true,
      "hotValency": 2
    }
  ],
  "publicRoots": []
}
```



## Meaning


| field                | description                                  |
| -------------------- | -------------------------------------------- |
| `monitoring-proxied` | enables monitoring proxy for this producer   |
| `poolBech32Id`       | source identifier used in published payloads |


Requirements:

- `poolBech32Id` must be a valid **56-character stake pool bech32 identifier**
- internal IP and port must **never appear in published payloads**

---



# 3. Publication Output Format

The relay publishes monitoring data as an array of publication objects.

## Example output

```json
{
  "publications": [
    {
      "type": "open",
      "payload": {
        "source": "self",
        "generated_at": "2026-03-17T14:20:00Z",
        "node_name": "cardano-node",
        "node_version_major": 10
      }
    },
    {
      "type": "encrypted",
      "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
      "ciphertext": "BASE64_CIPHERTEXT_FOR_SELF"
    },
    {
      "type": "open",
      "payload": {
        "source": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi",
        "generated_at": "2026-03-17T14:20:00Z",
        "node_name": "cardano-node",
        "node_version_major": 10,
        "state_hash": "abcdef123456",
        "network": "preview",
        "slot_height": 1234567
      }
    }
  ]
}
```

---



# 4. Payload Source Descriptor

Every payload must include a `source` field.

### Allowed values

```
self
```

or

```
<poolBech32Id>
```

Meaning:


| source value   | meaning                      |
| -------------- | ---------------------------- |
| `self`         | relay node publication       |
| `poolBech32Id` | proxied producer publication |


This prevents exposing internal network topology while allowing monitoring consumers to identify the originating node.

---



# 5. Default Field Set

The following fields are considered the **initial standard field set**.


| field                 | description                            |
| --------------------- | -------------------------------------- |
| `generated_at`        | ISO-8601 timestamp of payload creation |
| `node_name`           | node implementation name               |
| `node_version_major`  | major version number                   |
| `node_version_minor`  | minor version number                   |
| `node_version_hotfix` | hotfix / patch version                 |
| `cores`               | number of CPU cores                    |
| `memory`              | total memory available to node         |
| `state_hash`          | node state hash                        |
| `node_type`           | relay, block-producer, etc             |


The node version was intentionally split into separate fields:

```
node_name
node_version_major
node_version_minor
node_version_hotfix
```

This allows operators to selectively publish:

- only major version publicly
- full version only in encrypted publications

---



# 6. Extensible Field Model

The default fields listed above represent only the **baseline shared monitoring schema**.

Each node implementation may define additional fields such as:

- additional ledger state hashes
- node telemetry metrics
- resource utilization metrics
- network performance indicators
- implementation-specific diagnostics

Operators may choose to publish these fields by including them in the `fields` list.

### Important principles

1. **Nodes control which fields they support**
2. **Operators control which supported fields they publish**
3. **Monitoring consumers must tolerate differing field sets**

Different node implementations may therefore expose different monitoring datasets.

---



# 7. Field Standardization

To encourage ecosystem interoperability:

- new monitoring fields should ideally be **publicly documented**
- node teams are encouraged to coordinate field definitions
- field semantics should remain stable across implementations

The **Cardano Improvement Proposal (CIP)** documenting this monitoring specification should be updated whenever:

- a new common monitoring field is proposed
- a field definition changes
- a new field becomes widely adopted across node implementations

This allows other node teams to implement the same field consistently.

---



# 8. Encryption Behavior

Encrypted publications are generated as follows:

1. The node builds the plaintext payload.
2. The payload is serialized to JSON.
3. The payload is encrypted using the recipient's public key.
4. The ciphertext is encoded in base64.

Each encrypted publication must be encrypted independently.

Even if the plaintext payload does not change, ciphertext should differ between encryptions due to the properties of the encryption method.

---



# 9. Implementation Guidelines

Node implementations should follow these rules:

- reject unknown field names if strict mode is enabled
- reject duplicate field entries
- emit payload fields in canonical order
- always include `source`
- generate `generated_at` in **ISO-8601 UTC format**

Example:

```
2026-03-17T14:20:00Z
```

Proxy failures should not interrupt relay self-publications.

If a proxied producer is unreachable:

- omit its publication
- log the failure locally

---



# 10. Operator Guidance

Operators configure monitoring publications by:

1. selecting desired fields
2. defining which publications are open or encrypted
3. optionally enabling producer proxying

Operators should consider:

- publishing minimal public data
- sharing detailed telemetry only through encrypted publications
- coordinating with monitoring providers about supported fields

---



## 11. Transport and Mini-Protocol Integration

The monitoring publication mechanism requires a transport path between a monitoring consumer and a Cardano node. Two implementation approaches should be considered:

1. extending the existing node-to-node handshake mechanism; or
2. introducing a dedicated observability mini-protocol.

The final choice should consider implementation complexity, protocol semantics, extensibility, security, and the expected future scope of observability data.

It is clear that other mini-protocols, such as Chainsync, have longer-lasting sessions and will therefore take up a significant portion of the available connections. A handshake request is already possible to query, for example, N2N versions and peer sharing. These sessions are limited to a few seconds and do not place a sustained load on the node.  

### 11.1 Extending the Existing Handshake

The node-to-node handshake already exposes information that can be considered part of the proposed observability dataset, including information such as:

- `node2nodeSupportedVersions`
- `peerSharingEnabled`

Extending the handshake therefore has the advantage of building upon an existing mechanism through which peers already exchange node capabilities and protocol-related metadata.

Potential advantages include:

- reuse of an already implemented and deployed protocol path;
- reduced implementation complexity for an initial deployment;
- no additional mini-protocol negotiation solely for basic observability;
- natural placement for information directly related to node-to-node protocol capabilities.

However, the handshake primarily exists to establish protocol compatibility and negotiate a connection. Expanding it with increasingly rich monitoring information could mix two different concerns: connection establishment and operational observability.

This becomes particularly relevant if the field set grows over time to include resource information, state hashes, implementation-specific telemetry, encrypted publications, or proxied producer publications.

The implementation must also ensure that monitoring queries do not interfere with, alter, or unnecessarily repeat the normal node-to-node handshake lifecycle.

### 11.2 Dedicated Observability Mini-Protocol

An alternative is to define a new node-to-node mini-protocol dedicated to observability.

Conceptually, a consumer would establish a normal compatible node-to-node connection and subsequently query the observability mini-protocol for the currently generated publication dataset.

A dedicated mini-protocol provides a clearer separation of concerns:

- the handshake remains responsible for protocol negotiation and connection establishment;
- the observability mini-protocol is responsible for retrieving monitoring publications;
- observability can evolve independently from handshake semantics;
- additional fields and publication types can be introduced without continuously extending the handshake;
- encrypted and proxied publications fit naturally into a dedicated response structure;
- implementations can explicitly advertise support for the observability capability.

This approach may require more initial implementation work because a new mini-protocol, protocol identifier, message format, capability negotiation, and corresponding client/server behavior need to be defined.

### 11.3 Relationship to Existing Handshake Information

The introduction of a dedicated observability mini-protocol does not necessarily imply that existing handshake information should be removed or duplicated unnecessarily.

Information already available during the handshake, such as supported node-to-node protocol versions or peer-sharing capabilities, can remain part of the handshake for its existing protocol purpose.

The observability specification may nevertheless define equivalent standardized fields where exposing that information as part of a monitoring publication is useful. In that case, the node derives the publication field from the same internal state rather than treating the observability response as the authoritative source for handshake negotiation.

This distinction allows the same underlying node property to serve two purposes without coupling the protocols:

- **handshake:** protocol negotiation and peer compatibility;
- **observability:** monitoring, aggregation, historical collection, and analysis.



### 11.4 Considerations for Proxying

Both approaches must support the relay-to-producer proxy model described in this specification.

When `localRootProxy` is enabled, a relay must be able to retrieve the producer node's own configured monitoring publications through the internal node-to-node connection.

A dedicated observability mini-protocol provides a particularly direct model for this operation:

```text
Monitoring Provider
        |
        | public N2N connection
        v
      Relay
        |
        | internal N2N connection
        v
 Block Producer
```

The relay retrieves the producer's monitoring publications and republishes them as additional publication items while preserving the source and privacy rules defined by this specification.

No internal producer address or port is exposed to the external monitoring provider.

### 11.5 Protocol Selection

This specification does not initially mandate whether monitoring publications are transported through an extension of the existing handshake or through a dedicated observability mini-protocol.

Implementers should evaluate both approaches, with particular consideration given to the expected evolution of the monitoring dataset.

A handshake extension may provide the shortest implementation path for a small and largely static set of protocol-related fields. A dedicated observability mini-protocol provides a stronger separation of concerns and greater flexibility if the mechanism is expected to evolve into a broader, extensible monitoring interface.

Whichever transport mechanism is selected, the publication schema, field definitions, encryption model, source identification, and proxy behavior defined by this specification should remain independent of the transport implementation.

---



# 12. Design Summary

The monitoring publication system follows three key principles:

1. **Relay-centric publication**
  - relays act as the public monitoring endpoint
2. **Flexible payload datasets**
  - nodes define which monitoring fields they support
  - operators decide which fields to publish
3. **Source-safe proxying**
  - relays may proxy monitoring data from internal producer nodes
  - internal network details are never exposed
  - source identity uses the pool's bech32 identifier
4. **Supplementing block-markers**
  - Mechanisms developed in parallel for writing node versions into blocks are not competitive alternatives but rather plementary 
   mechanisms that take a different approach and may be more suitable depending on the intended use. 
  - Ideally, the results of the two measurement methods will corroborate each other.

---

A relay node configuration defines a set of monitoring publications and may optionally enable proxying of monitoring publications from selected internal producer nodes discovered through `topology.json`. Each configured publication selects a subset of predefined allowed fields using `fields`. The relay constructs its own payloads in a fixed envelope and either publishes them openly or encrypts them for a configured recipient public key. Proxied producer nodes are identified in topology by `monitoring-proxied: true` and `poolBech32Id`. Their monitoring payloads are fetched by the relay over the internal miniprotocol path and republished as additional publication items. Every payload must contain a `source` field, set to `self` for relay-local payloads or to the producer’s `poolBech32Id` for proxied payloads. Internal network coordinates must never be exposed in published source metadata.