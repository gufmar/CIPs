# Relay Monitoring Publication Specification

## Status

Draft proposal for discussion with Cardano node, networking, monitoring, and stake pool operator communities.

This document describes a common configuration and publication model for node observability data. Relay nodes act as the publicly reachable publication endpoint and may optionally proxy monitoring publications from internal block-producing nodes.

The proposal is intended to provide a small interoperable baseline while allowing individual node implementations to expose additional implementation-specific observability fields.

The publication system supports:

- **open publications** (plaintext);
- **encrypted publications** (recipient-specific);
- **optional proxying of monitoring publications from internal producer nodes**;
- **slot-aligned, cached snapshots** that are generated independently of incoming monitoring requests.

The logical publication schema is intentionally independent from the final node-to-node wire encoding. JSON is used throughout this document for configuration and explanatory examples.

---

# 1. Goals and Design Principles

The proposal follows these principles:

1. **Relay-centric public reachability**
   - public monitoring consumers query relay nodes;
   - block-producing nodes do not need public monitoring exposure.

2. **Operator-controlled disclosure**
   - node implementations define the fields they are capable of exposing;
   - operators choose which supported fields are published openly and which are only published encrypted.

3. **Stable publication envelopes**
   - the outer publication structure is standardized;
   - the payload field set can vary between node implementations and between individual operator configurations.

4. **Cached, request-independent generation**
   - publications are generated on a periodic slot-aligned cadence;
   - incoming monitoring requests only retrieve already-generated cached publications;
   - external requests must never trigger expensive metric collection, encryption, or producer-node queries.

5. **Source-safe proxying**
   - relays may retrieve publications from explicitly configured internal producer nodes;
   - internal network addresses and ports are never exposed to monitoring consumers.

6. **Extensible but interoperable fields**
   - this CIP defines a baseline set of common unprefixed fields;
   - implementations may introduce namespaced extension fields;
   - useful experimental fields can later be standardized through an update to this CIP.

---

# 2. Relay Configuration Template

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
          "node_name",
          "node_version_major"
        ]
      },
      {
        "type": "encrypted",
        "recipient_name": "Monitor A",
        "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
        "fields": [
          "node_name",
          "node_version_major",
          "node_version_minor",
          "node_version_patch",
          "cores",
          "memory",
          "state_hash",
          "node_type",
          "tip_slot",
          "block_height",
          "network",
          "node2node_supported_versions",
          "peer_sharing_enabled"
        ]
      }
    ]
  }
}
```

## Configuration elements

### `publications.localRootProxy`

Boolean flag controlling whether the relay attempts to discover and retrieve monitoring publications from configured internal producer nodes.

```text
true  -> proxy discovery and retrieval are enabled
false -> only relay self-publications are generated
```

Enabling this flag does not by itself make every internal peer eligible for proxying. Individual producer endpoints must also be explicitly marked as monitoring-proxied by the implementation-specific producer discovery configuration.

### `publications.items`

Array of relay-local publication templates.

Each publication template defines:

| field | required | description |
| --- | --- | --- |
| `type` | yes | `"open"` or `"encrypted"` |
| `fields` | yes | list of supported field identifiers selected for this publication |
| `recipient_public_key` | encrypted only | monitoring recipient's public encryption key |
| `recipient_name` | no | operator-facing convenience label |

The `fields` array is a selection list only.

Operators cannot use it to define arbitrary JSON structures, arbitrary output keys, variable substitutions, or placeholder expressions. The node implementation resolves each selected field from its own internal state and constructs the payload.

`recipient_name` has no protocol meaning. It exists only to make operator configuration easier to understand.

---

# 3. Producer Discovery and Proposed Topology Mapping

A relay may proxy monitoring publications from internal producer nodes already reachable through its private producer-relay topology.

For `cardano-node`, one possible mapping is to extend selected `localRoots[].accessPoints[]` entries in `topology.json`.

This mapping is **illustrative and proposed**, not a requirement on all node implementations.

## Example

```json
{
  "bootstrapPeers": [],
  "localRoots": [
    {
      "accessPoints": [
        {
          "address": "10.10.123.1",
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

### Proposed mapping fields

| field | description |
| --- | --- |
| `monitoring-proxied` | marks this internal endpoint as eligible for observability proxying |
| `poolBech32Id` | identifies the stake pool represented by the proxied producer |

`poolBech32Id` should be a valid Cardano pool identifier using the `pool` Bech32 representation defined by CIP-5.

The internal `address`, `port`, and local `description` are transport/configuration information only and must never be copied into externally published observability data.

## Alternative node implementations

Alternative Cardano node implementations are not required to use `topology.json` or the exact field placement shown above.

They may use any native configuration mechanism that can express the same concepts:

- which internal producer endpoint is eligible for observability proxying;
- how the relay reaches that producer;
- which pool identity the producer represents.

The CIP standardizes the observable behavior, not a single implementation-specific topology file format.

---

# 4. Publication Output Format

The relay exposes an array of publication objects.

Two metadata properties are part of the publication envelope and are not operator-selectable payload fields:

- `source`
- `snapshot_slot`

## Open publication example

```json
{
  "type": "open",
  "source": "self",
  "snapshot_slot": 123456600,
  "payload": {
    "node_name": "cardano-node",
    "node_version_major": 11
  }
}
```

## Encrypted publication example

```json
{
  "type": "encrypted",
  "source": "self",
  "snapshot_slot": 123456600,
  "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
  "ciphertext": "BASE64_CIPHERTEXT"
}
```

## Proxied producer publication example

```json
{
  "type": "encrypted",
  "source": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi",
  "snapshot_slot": 123456600,
  "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
  "ciphertext": "BASE64_CIPHERTEXT_FROM_PRODUCER"
}
```

## Full example response

```json
{
  "publications": [
    {
      "type": "open",
      "source": "self",
      "snapshot_slot": 123456600,
      "payload": {
        "node_name": "cardano-node",
        "node_version_major": 11
      }
    },
    {
      "type": "encrypted",
      "source": "self",
      "snapshot_slot": 123456600,
      "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
      "ciphertext": "BASE64_CIPHERTEXT_FOR_RELAY"
    },
    {
      "type": "encrypted",
      "source": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi",
      "snapshot_slot": 123456600,
      "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
      "ciphertext": "BASE64_CIPHERTEXT_FROM_PRODUCER"
    }
  ]
}
```

The encrypted ciphertext contains only the producer-generated payload. The relay does not need to decrypt or modify it.

---

# 5. Mandatory Publication Metadata

## `source`

Every publication contains a `source` string in the outer publication envelope.

Allowed baseline forms are:

```text
self
```

or:

```text
<poolBech32Id>
```

Meaning:

| source value | meaning |
| --- | --- |
| `self` | publication describes the publicly queried relay itself |
| `poolBech32Id` | publication describes a producer proxied through the relay |

Placing `source` in the publication envelope rather than inside the payload is important for encrypted proxied publications: the relay can associate an already-encrypted producer publication with its pool identity without decrypting or re-encrypting the producer payload.

### Privacy consideration

A pool identifier in the outer envelope is visible even when the payload is encrypted.

This is considered acceptable for the proposed trust model because monitoring consumers are expected to query relays based on the relay endpoints publicly registered on-chain for stake pools. A pool identity associated with a relay is therefore normally already public information.

Implementations and reviewers should nevertheless explicitly evaluate this metadata exposure before finalizing the CIP.

## `snapshot_slot`

Every publication contains a `snapshot_slot`.

`snapshot_slot` identifies the slot-aligned observation window for which the cached publication was generated. It replaces the previously proposed mandatory ISO-8601 `generated_at` timestamp.

It is deliberately distinct from a node's chain tip slot:

- `snapshot_slot` describes **when the monitoring snapshot is scheduled**;
- `tip_slot`, when selected as a payload field, describes **the slot of the node's currently selected chain tip**.

This distinction is important for nodes that are syncing or temporarily behind the network tip.

---

# 6. Slot-Aligned Snapshot Generation and Caching

Publication generation must be independent from incoming observability requests.

The recommended baseline cadence is one snapshot every **600 slots**.

On Cardano networks with one-second slots this corresponds to ten minutes, but the protocol definition is expressed in slots rather than wall-clock time.

## Snapshot boundaries

The default snapshot boundaries are absolute slot numbers divisible by 600:

```text
... 123456000
... 123456600
... 123457200
...
```

Node implementations should derive the current slot using their normal consensus/network time facilities.

For each snapshot interval, the implementation generates at most one publication set and caches it.

The generated publication is labelled with the corresponding `snapshot_slot`.

Implementations should not attempt to reconstruct missed historical snapshots after being offline.

## Why slot-based scheduling

Slot-aligned snapshots provide several useful properties:

- independent node implementations can label observations against the same protocol time axis;
- monitoring providers can compare snapshots from multiple nodes without relying on the nodes' formatted wall-clock timestamps;
- publication cadence is predictable;
- stale or replayed publications are easy to identify;
- external observers cannot infer node-version changes merely from whether ciphertext changed, because encrypted publications are regenerated at every scheduled snapshot.

Slot-based scheduling does not imply that all nodes have identical chain tips. `snapshot_slot` is a scheduling/time coordinate, while `tip_slot` is node state.

## Request behavior

A remote monitoring request:

- MUST return only cached publication data;
- MUST NOT cause the node to regenerate metrics synchronously;
- MUST NOT trigger encryption work beyond what was already performed for the current snapshot;
- MUST NOT trigger a new connection or query to an internal block producer.

This requirement protects both public relays and private producers from request-amplification or polling-induced load.

---

# 7. Proxy Retrieval and Caching

When `localRootProxy` is enabled, the relay periodically retrieves already-generated monitoring publications from eligible internal producer nodes over the configured private node-to-node connection path.

Producer retrieval should be aligned with the same snapshot cadence where practical.

The producer determines its own publication templates and field selection. The relay does **not** request individual payload fields from the producer.

Consequently:

- a producer publication may contain a different field set than the relay's own publication;
- different producers may expose different datasets;
- the relay acts as a retrieval/cache/forwarding point, not as the authority defining the producer's payload.

## Failure behavior

Failure to retrieve a producer publication must not affect relay-local publication generation.

If a proxied producer is unavailable:

- the relay should log the retrieval failure locally;
- a publication for the failed producer should be omitted from the current snapshot response unless the implementation explicitly supports serving stale cached publications;
- if stale data is ever returned, its original `snapshot_slot` must be preserved so the consumer can detect its age.

## Multiple producer endpoints for the same pool

A topology may contain more than one eligible producer endpoint associated with the same `poolBech32Id`, for example for active/standby or high-availability arrangements.

The baseline behavior should remain simple:

- endpoints are grouped by `poolBech32Id`;
- the relay attempts them in a stable implementation-defined priority, with configuration order being a reasonable default;
- the first reachable endpoint returning a valid publication is used for that pool for the current snapshot;
- duplicate publications for the same pool identity should not normally be emitted.

If the topology contains different `poolBech32Id` values, each distinct pool identity may be proxied independently.

Future versions may define explicit priority or weighting if operational experience demonstrates a need for it.

---

# 8. Draft Default Field Set

The fields below are the proposed **initial shared field vocabulary**.

They should be treated as draft baseline definitions to refine together with Cardano node development teams before this proposal becomes normative.

The primary goal at this stage is to establish useful names, types, and intended semantics so multiple implementations can converge on compatible meanings.

| field | proposed type | draft semantic definition |
| --- | --- | --- |
| `node_name` | string | stable name of the node implementation, e.g. `cardano-node`, `amaru` |
| `node_version_major` | unsigned integer | major component of the node software version |
| `node_version_minor` | unsigned integer | minor component of the node software version |
| `node_version_patch` | unsigned integer | patch component of the node software version |
| `node_type` | string | operational role represented by the publication, e.g. relay or block producer |
| `cores` | unsigned integer | number of logical CPU execution units available to the node process; container/cgroup limits should be respected where applicable |
| `memory` | unsigned integer | memory in bytes available to the node process; an effective container/cgroup limit should take precedence over host RAM where applicable |
| `state_hash` | string | provisional common state identifier; exact state domain, hash algorithm, encoding, and cross-implementation semantics require agreement with node teams |
| `tip_slot` | unsigned integer | slot number of the node's currently selected chain tip |
| `block_height` | unsigned integer | block number/height of the node's currently selected chain tip |
| `network` | string | network identifier; exact canonical representation should be agreed with node teams |
| `node2node_supported_versions` | array | node-to-node protocol versions currently supported by the implementation |
| `peer_sharing_enabled` | boolean | whether peer sharing is enabled/available according to the node's negotiated/configured networking capability |

## Version granularity

Node software version is intentionally split into:

```text
node_name
node_version_major
node_version_minor
node_version_patch
```

This allows an operator, for example, to publish:

- implementation name and major version openly;
- minor and patch version only to selected encrypted monitoring recipients.

The third version component is called `patch` rather than `hotfix` to align with common software version terminology.

## Provisional definitions

Some baseline fields require further agreement before they should be considered cross-implementation normative.

In particular:

- `state_hash` needs an exact definition of what state is hashed and which algorithm/encoding is used;
- `network` needs a canonical cross-implementation representation;
- CPU and memory semantics should be confirmed for bare-metal, containerized, and restricted-runtime environments;
- node role terminology should be aligned across alternative implementations.

The CIP discussion process should be used to refine these definitions with node development teams.

---

# 9. Extensible Field Model

The default fields are only the common baseline.

Each node implementation may define and expose additional observability fields, including for example:

- additional state hashes;
- ledger or consensus state indicators;
- implementation-specific telemetry;
- resource utilization data;
- networking statistics;
- mempool information;
- protocol-specific readiness information.

Operators decide which supported fields they want to include in their configured publications.

Monitoring consumers must therefore tolerate differing payload field sets.

## Standard and extension namespaces

To avoid collisions:

- **unprefixed field names are reserved for fields standardized by this CIP**;
- implementation-specific or experimental fields should use a stable namespace.

Examples:

```text
cardano_node.mempool_tx_count
cardano_node.chain_db_size
amaru.some_state_hash
alternative_node.scheduler_metric
```

The exact namespace registry mechanism can be refined during the CIP process, but namespaces should be:

- stable;
- publicly documented;
- unique enough to avoid collisions;
- independent from individual operator configuration.

Consumers should ignore unknown fields they do not understand rather than rejecting the entire publication.

---

# 10. Path from Experimental to Standard Fields

Implementation-specific experimentation is explicitly encouraged.

A new metric does not need to become a standardized CIP field before a node implementation can expose it under its own namespace.

A successful field can later become part of the common vocabulary when:

1. its operational value has been demonstrated;
2. its semantics are sufficiently precise;
3. multiple node implementations could reasonably expose equivalent information;
4. there is agreement on type, units, encoding, and privacy implications.

At that point an amendment to this CIP should define the unprefixed standardized field.

The CIP should therefore be considered a living registry for interoperable observability fields.

Changes worthy of CIP updates include:

- addition of a new common field;
- clarification of a field's semantics;
- changes to type, unit, or encoding;
- deprecation or replacement of an existing standard field;
- promotion of a successful implementation-specific experiment into the shared vocabulary.

---

# 11. Encryption Behavior

Encrypted publications are intended to provide recipient-specific confidentiality.

The proposed baseline uses **libsodium sealed boxes**.

Conceptually:

1. the node constructs the plaintext payload from its configured fields;
2. the payload is serialized in the canonical format required by the transport/profile;
3. the payload is encrypted for the monitoring provider's public key;
4. the encrypted bytes are returned as the publication ciphertext.

In the JSON representation used by this document, ciphertext and recipient public keys are represented as base64 strings.

A binary node-to-node wire format should carry these values as byte strings and does not need base64.

## Independent recipient encryption

Each recipient publication is encrypted independently.

If an operator configures two monitoring recipients, the node generates two encrypted publications, one for each recipient public key.

The monitoring entities remain cryptographically independent and do not share private keys or ciphertext decryption capability.

Encrypted publications can be addressed to multiple monitoring entities or node development teams. for Example Dingo Node could deliver his config files with a prepared template encrypted and addressed for the Dingo Development team. A spread of different encrypted recipients ensures monitoring data can be verified by multiple entities.  

## Ciphertext regeneration

Libsodium sealed boxes use a fresh ephemeral key pair for each encryption.

Therefore, repeated encryption of the same plaintext for the same recipient produces different ciphertext.

This property is important to the monitoring privacy model: an observer without the recipient private key cannot determine from ciphertext equality whether the underlying monitoring values remained unchanged.

No application-level random salt is required for this purpose.

---

# 12. Confidentiality, Authenticity, and Trust Model

Encryption and authenticity are separate concerns.

## What sealed-box encryption provides

The proposed sealed-box mechanism provides:

- confidentiality for the selected recipient;
- integrity against undetected modification of the ciphertext;
- fresh ciphertext for repeated encryption.

## What it does not provide

Sealed boxes do **not** authenticate the sender.

Anyone who knows a monitoring provider's public key can construct a sealed-box ciphertext that the provider can decrypt.

Therefore, an encrypted publication must not be described as a cryptographic attestation that a specific Cardano node produced a particular dataset.

## On-chain relay registration as operational provenance

The intended baseline attribution model is based on the relay endpoint registered on-chain for a stake pool.

A monitoring provider should:

1. obtain the current relay endpoints from stake pool registration data;
2. connect to a relay endpoint known to be registered for the pool being monitored;
3. retrieve observability publications from that endpoint;
4. for a proxied publication whose `source` is a pool ID, verify that the queried relay endpoint is currently registered on-chain for that pool identity.

This provides a useful operational provenance relationship because the monitoring provider deliberately queries an endpoint registered by the pool operator.

However, **on-chain relay registration is not cryptographic authentication of the live TCP/N2N peer or of the publication payload**.

The baseline design therefore relies on:

- on-chain endpoint registration;
- normal network routing/DNS assumptions;
- control of the registered relay by the pool operator.

If future use cases require cryptographic proof that a particular node or pool signed an observability snapshot, a separate authenticated publication mechanism should be defined.

Such a mechanism should not casually reuse Cardano cold, KES, or VRF signing keys.

---

# 13. Source Privacy and Multi-Pool Relays

Exposing a proxied `poolBech32Id` in the publication envelope reveals the pool identity even when the payload is encrypted.

For the intended discovery model, this normally reveals little or no additional information because the monitoring provider is expected to find the relay through public on-chain pool registration data.

The `source` value is primarily needed to disambiguate publications when:

- a relay serves more than one pool;
- a multi-pool operator proxies multiple producer nodes through one relay;
- multiple proxied publication items are returned in the same response.

A monitoring provider should reject or mark as untrusted a proxied source whose pool identity cannot be associated with the queried relay through the provider's current on-chain registration view.

If future deployments identify a material privacy issue with exposing the pool ID in the envelope, an alternative source-routing mechanism can be evaluated. Hashing the pool ID alone would not provide meaningful privacy because pool IDs are public and enumerable.

---

# 14. Implementation Guidelines

Node implementations should apply the following rules.

## Configuration validation

- reject unsupported field names in operator configuration;
- reject duplicate field names within one publication template;
- allow namespaced implementation-specific fields only if the node implementation explicitly supports them;
- validate encrypted recipient public keys before accepting the configuration.

## Payload construction

- construct payloads from node-owned internal values only;
- do not interpret operator-provided template expressions or arbitrary placeholders;
- use deterministic/canonical field ordering where the selected encoding requires it;
- ensure field types and units match their documented definitions.

## Mandatory metadata

- always include `source`;
- always include `snapshot_slot`;
- keep these properties outside the operator-selectable payload field list.

## Caching and load protection

- generate publication sets on the slot-based cadence rather than per request;
- cache the generated response;
- serve monitoring requests from cache;
- never let an external request synchronously trigger producer polling;
- enforce implementation-defined limits on publication count, message size, execution time, and protocol resource consumption.

Exact limits should be agreed with the networking/node teams and defined in the final protocol profile.

## Proxy failure isolation

Failure of one proxied producer must not prevent:

- relay self-publications;
- publications from other reachable proxied producers;
- normal node-to-node operation.

---

# 15. Operator Guidance

Operators configure monitoring publications by:

1. selecting the supported fields they want to expose;
2. choosing whether each publication is open or encrypted;
3. adding recipient public keys for encrypted publications;
4. optionally enabling producer proxying;
5. explicitly identifying which internal producer endpoints are eligible for proxy retrieval.

Operators should consider the privacy implications of every selected field.

A reasonable pattern may be:

- expose implementation name and major version openly;
- expose detailed software version, system characteristics, and additional state information only in encrypted publications;
- expose experimental implementation-specific fields only where their meaning and privacy impact are understood.

Operators should not assume that all node implementations expose the same fields.

---

# 16. Transport and Mini-Protocol Integration

The publication model requires a transport path between monitoring consumers and Cardano nodes.

Two approaches are technically possible:

1. extending the existing node-to-node handshake query mechanism;
2. defining a dedicated observability mini-protocol.

This draft recommends the **dedicated observability mini-protocol** as the target architecture.

## 16.1 Why the handshake is relevant

The existing node-to-node handshake already exposes information that is observability-relevant, including capabilities such as:

- supported node-to-node protocol versions;
- peer-sharing configuration/capability.

Handshake query behavior also demonstrates that short-lived capability queries can be performed without establishing long-running chain synchronization workloads.

This makes a handshake extension attractive as a small implementation experiment or bootstrap approach.

## 16.2 Why a dedicated observability mini-protocol is preferred

The handshake primarily exists to establish protocol compatibility and negotiate connection behavior.

The proposed observability mechanism can include:

- software version information;
- resource characteristics;
- chain/state information;
- encrypted publication sets;
- implementation-specific telemetry;
- proxied producer publications;
- future standardized observability extensions.

Placing this growing dataset into the handshake would couple operational monitoring to connection establishment and version negotiation.

A dedicated observability mini-protocol provides a cleaner separation:

- handshake remains responsible for protocol/version negotiation;
- observability is independently versionable;
- publication retrieval has its own resource and message-size limits;
- encrypted and proxied publication sets fit naturally into request/response messages;
- future observability evolution does not continuously expand handshake version data;
- support can be explicitly negotiated or associated with compatible N2N protocol versions.

The recommended observability protocol should be deliberately simple and read-only.

Conceptually:

```text
Client                         Node
  |                              |
  |------ GetPublications ------>|
  |                              |
  |<----- Publications ----------|
  |                              |
```

The request does not select fields and does not trigger snapshot generation.

It only retrieves the node's currently cached publication set.

## 16.3 Relationship to existing handshake information

Introducing an observability mini-protocol does not require removing information already used by the handshake.

The same underlying node property may serve different purposes:

- **handshake:** protocol compatibility and connection negotiation;
- **observability:** monitoring, historical collection, network analysis, and comparison.

For example, a node may expose `node2node_supported_versions` as an observability field while continuing to negotiate those versions through the normal handshake path.

The observability value is informational; it is not authoritative for connection negotiation.

## 16.4 Logical JSON model versus wire encoding

The JSON structures in this document define:

- operator configuration;
- the logical publication model;
- human-readable examples.

They should not be interpreted as a requirement to transport JSON over Ouroboros N2N.

A dedicated Ouroboros mini-protocol should define a versioned binary wire representation consistent with existing Ouroboros networking practices, expected to use CBOR with a corresponding CDDL specification.

In that wire representation:

- integers should remain integers;
- recipient keys should be byte strings;
- ciphertext should be a byte string;
- base64 is unnecessary.

The logical schema should remain stable regardless of JSON or CBOR representation.

## 16.5 Proxy transport

The same observability mini-protocol can be used:

```text
Monitoring Provider
        |
        | public N2N
        v
      Relay
        |
        | private/internal N2N
        v
 Block Producer
```

The block producer serves its own cached publication set.

The relay periodically retrieves it, associates it with the configured pool source, caches it, and makes it available as an additional publication item.

A public monitoring request to the relay never directly causes the relay to query the producer.

---

# 17. Security and Resource Considerations

The observability mechanism is intentionally read-only, but public availability still creates an attack surface.

Implementations should consider:

- connection-rate limiting;
- request-rate limiting;
- maximum response size;
- maximum publication count;
- mini-protocol state timeouts;
- memory limits;
- CPU cost of serialization and encryption;
- stale-cache handling;
- malformed recipient-key configuration;
- malformed or oversized proxied responses;
- isolation of proxy failures from normal diffusion operation.

Because encrypted content is regenerated on a fixed snapshot cadence, an unauthenticated observer can see that a new ciphertext exists but cannot use ciphertext equality to determine whether a specific underlying value changed.

The protocol does not attempt to hide:

- that a relay supports observability;
- the configured snapshot cadence;
- response size;
- the number of publication items;
- source metadata in the publication envelope.

If these metadata become sensitive in future use cases, padding or additional privacy mechanisms can be defined separately.

---

# 18. Relationship to Block-Embedded Version Markers

Mechanisms that record node implementation/version information in produced blocks are complementary to this observability proposal rather than competing alternatives.

They provide different properties:

- block-embedded markers directly observe software information associated with block production;
- relay observability can cover nodes that are not currently producing blocks and can expose a broader, operator-selected dataset;
- encrypted observability can reveal detailed information to authorized monitors without exposing it publicly;
- independent measurement methods can corroborate each other.

The appropriate mechanism depends on the intended analysis.

---

# 19. Design Summary

The monitoring publication system follows five key principles:

1. **Relay-centric publication**
   - relays act as the publicly reachable monitoring endpoint.

2. **Operator-controlled disclosure**
   - node implementations define supported observability fields;
   - operators decide which supported fields are published openly or encrypted.

3. **Slot-aligned cached snapshots**
   - monitoring data is periodically generated and cached;
   - public requests never trigger expensive collection or producer polling.

4. **Source-safe producer proxying**
   - relays may proxy monitoring publications from private producer nodes;
   - internal network coordinates are never exposed;
   - proxied sources use public pool identity for assignment.

5. **Extensible interoperability**
   - the CIP defines shared unprefixed fields;
   - node-specific experimentation uses namespaces;
   - successful cross-implementation fields can graduate into the shared CIP registry.

---

# 20. Concise Normative Description

A participating node generates a cached set of observability publications on a slot-aligned cadence. Each publication contains mandatory `source` and `snapshot_slot` metadata and either an open payload or a recipient-specific encrypted ciphertext.

Operators select payload fields only from fields supported by their node implementation. The node itself constructs all field values and output structure; arbitrary operator-defined JSON templates and placeholder variables are not supported.

Relay nodes may optionally retrieve cached publications from explicitly configured private producer nodes. Producer field selection is controlled by the producer's own configuration. Public monitoring requests to the relay must never synchronously trigger producer retrieval.

For Cardano implementations using `topology.json`, `localRoots[].accessPoints[]` metadata containing a proxy marker and pool identifier is one proposed way to map producer endpoints to observable pool identities. Alternative node implementations may use different internal configuration mechanisms while providing equivalent observable behavior.

A dedicated read-only node-to-node observability mini-protocol is the preferred transport design. The JSON examples in this specification describe the logical model; the final N2N wire format should be separately versioned and encoded using the conventions adopted by Ouroboros networking, expected to be CBOR/CDDL.

Encrypted publications provide recipient confidentiality but do not by themselves authenticate the publishing node. Monitoring providers are expected to derive relay endpoints from current on-chain stake pool registration data and use that relationship as baseline operational provenance. Strong cryptographic node attestation, if required in future, should be specified separately.

---

# 21. References and Related Work

The following existing Cardano/Ouroboros concepts are relevant to the proposal:

- Cardano stake pool registration and on-chain relay endpoints:
  <https://developers.cardano.org/docs/operate-a-stake-pool/block-producer/register-stake-pool/>

- CIP-5 common Bech32 prefixes, including the `pool` identifier:
  <https://cips.cardano.org/cip/CIP-5>

- Ouroboros Network repository and mini-protocol architecture:
  <https://github.com/IntersectMBO/ouroboros-network>

- Ouroboros Network changelog, including current N2N versioning and additional mini-protocol development:
  <https://github.com/IntersectMBO/ouroboros-network/blob/main/ouroboros-network/CHANGELOG.md>

- Cardano networking protocol overview:
  <https://docs.cardano.org/about-cardano/explore-more/cardano-network/networking-protocol>

- Cardano time and slot handling:
  <https://docs.cardano.org/about-cardano/explore-more/time>

- Libsodium sealed boxes:
  <https://doc.libsodium.org/public-key_cryptography/sealed_boxes>
