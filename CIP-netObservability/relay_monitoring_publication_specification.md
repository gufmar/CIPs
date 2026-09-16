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



# 11. Design Summary

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
  - Mechanisms developed in parallel for writing node versions into 
   blocks are not competitive alternatives but rather complementary 
   mechanisms that take a different approach and may be more suitable 
   depending on the intended use. 
  - Ideally, the results of the two measurement methods will corroborate each other.

---

A relay node configuration defines a set of monitoring publications and may optionally enable proxying of monitoring publications from selected internal producer nodes discovered through `topology.json`. Each configured publication selects a subset of predefined allowed fields using `fields`. The relay constructs its own payloads in a fixed envelope and either publishes them openly or encrypts them for a configured recipient public key. Proxied producer nodes are identified in topology by `monitoring-proxied: true` and `poolBech32Id`. Their monitoring payloads are fetched by the relay over the internal miniprotocol path and republished as additional publication items. Every payload must contain a `source` field, set to `self` for relay-local payloads or to the producer’s `poolBech32Id` for proxied payloads. Internal network coordinates must never be exposed in published source metadata.