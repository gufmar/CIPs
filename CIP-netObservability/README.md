---
CIP: "?"
Title: Network Observability - Publication Model for Public Relays
Category: Network
Status: Proposed
Authors:
    - Markus Gufler <markus.gufler@cardanofoundation.org>
Implementors: []
Discussions: []
Created: 2026-09-17
License: CC-BY-4.0
---

## Abstract

This proposal invites the community to turn the mini-protocol endpoints of publicly reachable Cardano network nodes into shared observability points. With a clear operator opt-in or opt-out, those endpoints can publish a small, comparable set of live signals so operators, monitors, and node teams get better situation awareness across a diverse network.

It defines a common configuration and publication model in which relays are the public contact point. Publications are slot-aligned cached snapshots retrieved through a dedicated read-only node-to-node mini-protocol. The model supports:

- **open** publications (plaintext);
- **encrypted** publications (recipient-specific);
- **optional producer proxying** without exposing internal addresses;
- a shared field vocabulary under topic prefixes, plus implementation namespaces for experiments.

Field names in JSON examples are logical identifiers for configuration and documentation. The wire encoding is expected to be versioned CBOR/CDDL with a compact key registry.

## Motivation: Why is this CIP necessary?

Shared, operator-controlled observability is becoming more useful than ad hoc telemetry or single-implementation dashboards. We should lock a small interoperable baseline now so tooling, SPO practice, and node teams can converge before network diversity and protocol performance work raise the cost of flying blind.

### Case 1: Node diversity and comparable deployments

Multiple node implementations and deployment styles will coexist. Peering, propagation, mempool behavior, and resource profiles already differ a lot. Without a common publication model, comparing tip progress, peer behavior, and config effects stays fragmented across clients and operators.

### Case 2: Fork risk and convergence visibility

As stake, topologies, and software mixes change, late or silent divergence gets more expensive. Slot-aligned tip and height signals give operators and monitors an earlier shared view of network convergence, without exposing block producers publicly.

### Case 3: Protocol performance rollouts

Leios, Peras, and related networking changes need client-independent, request-safe observability to validate rollouts and diagnose regressions once they are live.

## Specification



### Table of Contents

- [Goals and design principles](#goals-and-design-principles)
- [Relay configuration template](#relay-configuration-template)
- [Producer discovery and proposed topology mapping](#producer-discovery-and-proposed-topology-mapping)
- [Publication output format](#publication-output-format)
- [Mandatory publication metadata](#mandatory-publication-metadata)
- [Slot-aligned snapshot generation and caching](#slot-aligned-snapshot-generation-and-caching)
- [Proxy retrieval and caching](#proxy-retrieval-and-caching)
- [Draft default field set](#draft-default-field-set)
- [Extensible field model](#extensible-field-model)
- [Path from experimental to standard fields](#path-from-experimental-to-standard-fields)
- [Encryption behavior](#encryption-behavior)
- [Confidentiality, authenticity, and trust model](#confidentiality-authenticity-and-trust-model)
- [Source privacy and multi-pool relays](#source-privacy-and-multi-pool-relays)
- [Implementation guidelines](#implementation-guidelines)
- [Operator guidance](#operator-guidance)
- [Transport and mini-protocol integration](#transport-and-mini-protocol-integration)
- [Security and resource considerations](#security-and-resource-considerations)
- [Concise normative description](#concise-normative-description)



### Goals and Design Principles

1. **Relay-centric public reachability**
  - monitors query relays;
  - block producers need no public monitoring exposure.
2. **Operator-controlled disclosure**
  - implementations define which fields they can expose;
  - operators choose which of those are open vs encrypted.
3. **Stable publication envelopes**
  - outer structure is standardized;
  - payload fields may differ by implementation and operator config.
4. **Cached, request-independent generation**
  - publications are built on a slot-aligned cadence;
  - requests only read the cache;
  - **Do not** let external requests trigger metric collection, encryption, or producer queries.
5. **Source-safe proxying**
  - relays may fetch publications from configured internal producers;
  - **Do not** expose internal addresses or ports to monitors.
6. **Extensible but interoperable fields**
  - this CIP defines common fields under approved topic prefixes (`node_`, `system_`, `chain_`, …);
  - implementations may add impl-namespaced extensions;
  - useful experiments can later move into this CIP.

---



### Relay Configuration Template

Relays define monitoring publications in node configuration.

#### Example configuration

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
          "node_type",
          "node_n2n_supported_versions",
          "node_peer_sharing_enabled",
          "system_cores",
          "system_memory",
          "chain_state_hash",
          "chain_tip_slot",
          "chain_block_height",
          "chain_network"
        ]
      }
    ]
  }
}
```



#### Configuration elements



##### `publications.localRootProxy`

Whether the relay discovers and retrieves publications from configured internal producers.

```text
true  -> proxy discovery and retrieval enabled
false -> relay self-publications only
```

This flag alone does not make every internal peer eligible. Each producer endpoint must also be marked as monitoring-proxied in the implementation's producer discovery config.

##### `publications.items`

Array of relay-local publication templates.


| field                  | required       | description                                      |
| ---------------------- | -------------- | ------------------------------------------------ |
| `type`                 | yes            | `"open"` or `"encrypted"`                        |
| `fields`               | yes            | supported field identifiers for this publication |
| `recipient_public_key` | encrypted only | recipient public encryption key                  |
| `recipient_name`       | no             | operator-facing label only                       |


`fields` is a selection list only.

**Do not** treat it as free-form JSON, custom output keys, or placeholder templates. The node resolves each selected field from internal state and builds the payload.

`recipient_name` has no protocol meaning.

---



### Producer Discovery and Proposed Topology Mapping

A relay may proxy publications from internal producers already reachable on its private producer-relay topology.

For `cardano-node`, one possible mapping is extending selected `localRoots[].accessPoints[]` entries in `topology.json`.

This mapping is **illustrative**, not required for all implementations.

#### Example

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



##### Proposed mapping fields


| field                | description                                                         |
| -------------------- | ------------------------------------------------------------------- |
| `monitoring-proxied` | marks this internal endpoint as eligible for observability proxying |
| `poolBech32Id`       | stake pool represented by the proxied producer                      |


`poolBech32Id` should be a valid CIP-5 `pool` Bech32 identifier.

Internal `address`, `port`, and local `description` are config/transport only. **Do not** copy them into published observability data.

#### Alternative node implementations

Other implementations need not use `topology.json` or this exact field placement.

They need any native config that can express:

- which internal producer is eligible for proxying;
- how the relay reaches it;
- which pool identity it represents.

The CIP standardizes observable behavior, not one topology file format.

---



### Publication Output Format

The relay returns an array of publication objects.

Envelope metadata (not operator-selectable payload fields):

- `source`
- `snapshot_slot`



#### Open publication example

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



#### Encrypted publication example

```json
{
  "type": "encrypted",
  "source": "self",
  "snapshot_slot": 123456600,
  "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
  "ciphertext": "BASE64_CIPHERTEXT"
}
```



#### Proxied producer publication example

```json
{
  "type": "encrypted",
  "source": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi",
  "snapshot_slot": 123456600,
  "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
  "ciphertext": "BASE64_CIPHERTEXT_FROM_PRODUCER"
}
```



#### Full example response

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

Encrypted ciphertext is the producer-generated payload only. The relay does not need to decrypt or rewrite it.

---



### Mandatory Publication Metadata



#### `source`

Every publication has a `source` string in the outer envelope.

Allowed baseline forms:

```text
self
```

or:

```text
<poolBech32Id>
```


| source value   | meaning                                        |
| -------------- | ---------------------------------------------- |
| `self`         | describes the publicly queried relay           |
| `poolBech32Id` | describes a producer proxied through the relay |


`source` stays in the envelope (not inside the payload) so the relay can attach pool identity to an already-encrypted producer publication without decrypting it.

##### Privacy consideration

A pool ID in the envelope is visible even when the payload is encrypted.

That is acceptable under the intended trust model: monitors query relays from on-chain registered endpoints, so the pool↔relay link is usually already public. Reviewers should still explicitly weigh this metadata exposure before the CIP is finalized. Multi-pool cases are covered in [Source privacy and multi-pool relays](#source-privacy-and-multi-pool-relays).

#### `snapshot_slot`

Every publication has a `snapshot_slot`.

It names the slot-aligned observation window for the cached publication.

It is not the chain tip:

- `snapshot_slot` = when the monitoring snapshot was scheduled;
- `chain_tip_slot` (optional payload field) = slot of the node's currently selected tip.

That split matters for nodes that are syncing or temporarily behind.

---



### Slot-Aligned Snapshot Generation and Caching

Publication generation must be independent of incoming observability requests.

Baseline cadence: one snapshot every **600 slots** (ten minutes on networks with 1s slots). The protocol speaks in slots, not wall-clock time.

#### Snapshot boundaries

Default boundaries are absolute slot numbers divisible by 600:

```text
... 123456000
... 123456600
... 123457200
...
```

Implementations should derive current slot from normal consensus/network time facilities.

Per interval: generate at most one publication set, cache it, label it with `snapshot_slot`.

**Do not** reconstruct missed historical snapshots after downtime.

#### Timing near snapshot boundaries

All cached publications for a given `snapshot_slot` (self-generated and proxy-retrieved) should be ready on a tight but safe window around that boundary.

Rules of thumb:

- compute as close to the snapshot boundary as practical, so published values stay fresh;
- leave enough headroom for field collection, serialization, and encryption before any peer is expected to read the cache;
- a node that may be queried by another node (e.g. a producer queried by its relay) **must** finish preparing its cache for that `snapshot_slot` before those queries arrive;
- a relay should generate its own cache and retrieve producer publications in the same near-boundary window, in time for both to land under the same `snapshot_slot` label.

Exact offsets are implementation-defined. Prefer a conservative safe range over racing the boundary. Missed or late producer retrieval follows [Failure behavior](#failure-behavior).

#### Why slot-based scheduling

- independent implementations share the same protocol time axis;
- monitors can compare nodes without trusting formatted wall-clock timestamps;
- cadence is predictable;
- stale or replayed publications are easy to spot;
- ciphertext is regenerated every snapshot, so observers cannot infer value changes from ciphertext equality alone.

Slot scheduling does not imply identical tips. `snapshot_slot` is a schedule coordinate; `chain_tip_slot` is node state.

#### Request behavior

A remote monitoring request:

- MUST return only cached publication data;
- MUST NOT regenerate metrics synchronously;
- MUST NOT trigger encryption beyond work already done for the current snapshot;
- MUST NOT open a new connection or query to an internal block producer.

This protects relays and private producers from request amplification.

#### Generation cost

Cached snapshots still run on the cadence above, so field collection must stay cheap.

**Do not** define or select publication fields that need extraordinary node work each interval (for example a full ledger snapshot or other heavy state dump). At a 600-slot baseline that cost would hit the node about every ten minutes even with zero monitoring traffic.

Prefer values already available from normal node state, or cheap incremental summaries. Heavy diagnostics belong outside this publication path.

---



### Proxy Retrieval and Caching

When `localRootProxy` is enabled, the relay periodically retrieves already-generated publications from eligible internal producers over the private N2N path.

Timing follows [Slot-aligned snapshot generation and caching](#slot-aligned-snapshot-generation-and-caching): the producer must have its cache ready before the relay queries it; the relay retrieves near the snapshot boundary together with its own generation.

The producer owns its templates and field selection. The relay does **not** ask the producer for individual fields.

So:

- producer field sets may differ from the relay's;
- different producers may differ from each other;
- the relay is retrieval/cache/forward only, not the authority for producer payloads.

Public monitor requests never cause the relay to query the producer (same mini-protocol path; see [Transport and mini-protocol integration](#transport-and-mini-protocol-integration)).

#### Failure behavior

Producer retrieval failure must not block relay-local publications.

If a proxied producer is unavailable:

- log locally;
- omit that producer from the current response unless the implementation explicitly serves stale cache;
- if stale data is returned, keep the original `snapshot_slot` so consumers see the age.



#### Multiple producer endpoints for the same pool

A topology may list more than one eligible producer for the same `poolBech32Id` (active/standby, HA).

Baseline:

- group endpoints by `poolBech32Id`;
- try them in a stable implementation-defined order (config order is a fine default);
- use the first reachable endpoint that returns a valid publication for that snapshot;
- **Do not** normally emit duplicate publications for the same pool identity.

Distinct `poolBech32Id` values may be proxied independently.

Explicit priority/weighting can wait until ops experience shows a need.

---



### Draft Default Field Set

Proposed **initial shared field vocabulary**. Draft names, types, and semantics for node teams to refine before this becomes normative.

Standard fields use **topic prefixes** for grouping (`node_`, `system_`, `chain_`). These prefixes are part of the CIP baseline, not implementation namespaces (see [Extensible field model](#extensible-field-model)).


| field                         | proposed type    | draft semantic definition                                                                                           |
| ----------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| `node_name`                   | string           | stable name of the node implementation, e.g. `cardano-node`, `amaru`                                                |
| `node_version_major`          | unsigned integer | major component of the node software version                                                                        |
| `node_version_minor`          | unsigned integer | minor component of the node software version                                                                        |
| `node_version_patch`          | unsigned integer | patch component of the node software version                                                                        |
| `node_type`                   | string           | operational role, e.g. relay or block producer                                                                      |
| `node_n2n_supported_versions` | array            | N2N protocol versions currently supported                                                                           |
| `node_peer_sharing_enabled`   | boolean          | whether peer sharing is enabled/available per negotiated/configured capability                                      |
| `system_cores`                | unsigned integer | logical CPU units available to the node process; respect container/cgroup limits where applicable                   |
| `system_memory`               | unsigned integer | memory in bytes available to the node process; effective container/cgroup limit before host RAM where applicable    |
| `chain_state_hash`            | string           | provisional common state identifier; domain, algorithm, encoding, and cross-impl semantics need node-team agreement |
| `chain_tip_slot`              | unsigned integer | slot of the node's currently selected chain tip                                                                     |
| `chain_block_height`          | unsigned integer | block number/height of the currently selected tip                                                                   |
| `chain_network`               | string           | network identifier; canonical form needs agreement                                                                  |




#### Version granularity

Version is split on purpose:

```text
node_name
node_version_major
node_version_minor
node_version_patch
```

So an operator can publish name + major openly, and minor/patch only to encrypted recipients.

`patch` (not `hotfix`) matches common version naming.

#### Provisional definitions

Still need cross-implementation agreement for:

- `chain_state_hash` (what is hashed, algorithm, encoding);
- `chain_network` (canonical representation);
- `system_cores` / `system_memory` semantics on bare metal, containers, restricted runtimes;
- node role terminology across implementations.

Refine these with node development teams via the CIP process.

---



### Extensible Field Model

Baseline fields are the common set only.

Implementations may expose more, e.g.:

- additional state hashes;
- ledger/consensus indicators;
- implementation-specific telemetry;
- resource utilization;
- networking stats;
- mempool information;
- protocol-specific readiness.

Operators choose which supported fields go into their publications. Consumers must tolerate differing field sets.

#### Standard topic prefixes vs implementation namespaces

Two different naming layers:

- **CIP standard fields** use approved **topic prefixes**: `node_`, `system_`, `chain_`, and later additions agreed in this CIP.
- **Implementation-specific or experimental fields** use an **impl namespace**, not a topic prefix alone.

Examples:

```text
node_name
system_cores
chain_tip_slot
cardano_node.mempool_tx_count
cardano_node.chain_db_size
amaru.some_state_hash
alternative_node.scheduler_metric
```

Namespace registry details can be refined later. Impl namespaces should be stable, documented, collision-resistant, and independent of operator config.

**Do not** mint new top-level topic prefixes in local configs. Propose them via CIP update if a new standard category is needed.

Consumers should ignore unknown fields rather than reject the whole publication.

---



### Path from Experimental to Standard Fields

Namespaced experiments are encouraged. A metric need not be a CIP field before an implementation can expose it.

Promote a field into the common vocabulary when:

1. operational value is shown;
2. semantics are precise enough;
3. multiple implementations could expose equivalent data;
4. type, units, encoding, and privacy impact are agreed.

Then amend this CIP with a standard field under an approved topic prefix.

Treat this CIP as a living registry. Updates include:

- new common fields;
- new approved topic prefixes when needed;
- semantic clarifications;
- type/unit/encoding changes;
- deprecation or replacement;
- promotion of a successful namespaced experiment.

---



### Encryption Behavior

Encrypted publications give recipient-specific confidentiality.

Baseline: **libsodium sealed boxes**.

Steps:

1. build plaintext payload from configured fields;
2. serialize in the canonical transport format;
3. encrypt to the recipient public key;
4. return ciphertext on the publication.

In JSON examples, ciphertext and recipient keys are base64. A binary N2N wire format should carry byte strings (no base64).

#### Independent recipient encryption

Each recipient publication is encrypted separately.

Two configured recipients ⇒ two encrypted publications, one per public key. Recipients do not share keys or decryption capability.

Multiple recipients are useful so different monitoring parties (or a node team's own ops) can verify the same operator without sharing private keys. Operators may ship config templates with a recipient key already filled for that team's monitor.

#### Ciphertext regeneration

Sealed boxes use a fresh ephemeral key pair per encryption. Same plaintext + same recipient ⇒ different ciphertext.

So an observer without the private key cannot tell from ciphertext equality whether underlying values changed. No extra application-level salt is required.

---



### Confidentiality, Authenticity, and Trust Model

Encryption and authenticity are separate.

#### What sealed-box encryption provides

- confidentiality for the selected recipient;
- integrity against undetected ciphertext modification;
- fresh ciphertext on each encryption.



#### What it does not provide

Sealed boxes do **not** authenticate the sender.

Anyone who knows a monitor's public key can build a decryptable sealed box.

Encrypted publication is **not** a cryptographic attestation that a specific Cardano node produced the dataset.

#### On-chain relay registration as operational provenance

Baseline attribution: the relay endpoint registered on-chain for a stake pool.

A monitor should:

1. take current relay endpoints from stake pool registration data;
2. connect to a relay registered for the pool under watch;
3. retrieve publications from that endpoint;
4. for a proxied `source` pool ID, check that the queried relay is still registered for that pool.

That is useful operational provenance: the monitor chose an endpoint the pool operator registered.

It is **not** cryptographic authentication of the live TCP/N2N peer or of the payload.

Baseline assumptions:

- on-chain endpoint registration;
- normal routing/DNS;
- operator control of the registered relay.

If a use case needs cryptographic proof that a node or pool signed a snapshot, define a separate authenticated publication mechanism. **Do not** casually reuse cold, KES, or VRF keys.

---



### Source Privacy and Multi-Pool Relays

Envelope `source` privacy for a single pool is covered in [Mandatory publication metadata](#mandatory-publication-metadata). This section covers multi-pool relays.

`source` is needed to disambiguate when:

- one relay serves more than one pool;
- a multi-pool operator proxies several producers through one relay;
- one response carries multiple proxied items.

Monitors should reject or mark untrusted a proxied source whose pool ID cannot be tied to the queried relay in the current on-chain registration view.

If envelope pool IDs later prove to be a real privacy problem, evaluate an alternative source-routing scheme. Hashing the pool ID alone does not help: pool IDs are public and enumerable.

---



### Implementation Guidelines



#### Configuration validation

- reject unsupported field names;
- reject duplicate fields within one template;
- allow namespaced fields only if the implementation supports them;
- validate encrypted recipient public keys before accepting config.



#### Payload construction

- build payloads from node-owned internal values only;
- **Do not** interpret operator template expressions or placeholders;
- use deterministic/canonical field order when the encoding requires it;
- match documented types and units.



#### Mandatory metadata

- always include `source` and `snapshot_slot`;
- keep them outside the operator-selectable field list.



#### Caching and load protection

- generate on the slot cadence, not per request;
- serve from cache;
- never let an external request synchronously trigger producer polling;
- enforce limits on publication count, message size, execution time, and protocol resources.

Agree exact limits with networking/node teams in the final protocol profile.

#### Proxy failure isolation

One failed proxied producer must not block:

- relay self-publications;
- other reachable proxied producers;
- normal N2N diffusion.

---



### Operator Guidance

Configure publications by:

1. selecting supported fields to expose;
2. choosing open vs encrypted per publication;
3. adding recipient public keys for encrypted items;
4. optionally enabling producer proxying;
5. marking which internal producers are eligible for proxy retrieval.

Weigh privacy for every selected field.

A reasonable pattern:

- open: implementation name + major version;
- encrypted: full version, system characteristics, richer state;
- namespaced experiments only when meaning and privacy impact are understood.

Do not assume every implementation exposes the same fields.

---



### Transport and Mini-Protocol Integration

Publications need a transport path from monitors to nodes.

This CIP specifies a **dedicated observability mini-protocol** as the target architecture (see Rationale for handshake alternatives).

Keep the protocol simple and read-only:

```text
Client                         Node
  |                              |
  |------ GetPublications ------>|
  |                              |
  |<----- Publications ----------|
  |                              |
```

The request does not select fields and does not trigger snapshot generation. It only returns the current cache.

#### Relationship to existing handshake information

Do not remove handshake fields just because observability exists.

Same underlying property, different jobs:

- **handshake:** compatibility and connection negotiation;
- **observability:** monitoring, history, comparison.

Example: publish `node_n2n_supported_versions` for monitors while still negotiating those versions in the handshake. Observability values are informational, not authoritative for negotiation.

#### Logical field IDs versus wire encoding

Field names in this document (`node_name`, `system_cores`, `chain_tip_slot`, …) are the **logical field identifiers**. They appear in:

- operator configuration (`fields` selection lists);
- documentation and this CIP;
- the logical publication model used in JSON examples.

JSON examples are for humans. They are not a requirement to ship JSON string keys over Ouroboros N2N.

The mini-protocol should define a versioned binary wire form (expected: CBOR + CDDL). Typical pattern:

- CBOR maps use compact **integer keys** (or equivalent tags);
- a small registry maps those keys to the logical field identifiers above;
- encrypted payloads encrypt the canonical CBOR plaintext of the selected fields, then decrypt back through the same registry.

In that wire representation:

- integers stay integers;
- recipient keys and ciphertext are byte strings;
- base64 is unnecessary;
- logical field IDs need not appear as UTF-8 strings on every message.

Logical schema stays stable across JSON examples and CBOR wire encoding.

#### Proxy transport

Same mini-protocol on both hops:

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

Producer serves its cache. Relay retrieves on its cadence, attaches configured pool `source`, caches, and exposes the item. Runtime rules: see [Proxy retrieval and caching](#proxy-retrieval-and-caching).

---



### Security and Resource Considerations

Read-only, but still a public attack surface.

Implementations should cover:

- connection-rate and request-rate limits;
- max response size and publication count;
- mini-protocol timeouts;
- memory and CPU (serialize/encrypt);
- stale-cache handling;
- malformed recipient keys;
- malformed or oversized proxied responses;
- isolation of proxy failures from diffusion.

Ciphertext regenerates every snapshot, so observers see new ciphertext without learning whether a specific value changed.

The protocol does not hide:

- that observability is supported;
- snapshot cadence;
- response size / publication count;
- envelope `source` metadata.

If that metadata becomes sensitive later, define padding or other privacy measures separately.

---



### Concise normative description

A participating node builds a cached observability publication set on a slot-aligned cadence. Each publication has mandatory `source` and `snapshot_slot`, plus either an open payload or recipient-specific ciphertext.

Operators select payload fields only from fields the implementation supports. The node builds values and structure; no operator JSON templates or placeholders.

Relays may retrieve cached publications from configured private producers. Producer field selection is the producer's own. Public relay requests must never synchronously trigger producer retrieval.

For `topology.json` deployments, `localRoots[].accessPoints[]` with a proxy marker and `poolBech32Id` is one proposed mapping. Other implementations may use equivalent native config.

Preferred transport: dedicated read-only N2N observability mini-protocol. Field names here are logical IDs (config/docs/registry); wire format is separately versioned CBOR/CDDL, typically with compact integer keys mapped to those IDs.

Encryption gives recipient confidentiality, not publisher authentication. Monitors should use on-chain registered relay endpoints as baseline operational provenance. Strong node attestation, if needed later, is a separate mechanism.

---



## Rationale: How does this CIP achieve its goals?



### Why relay-centric cached publications

Public monitors should not need reachability into private producer networks. Relays are already the public edge of stake-pool topology and on-chain registration. Generating publications on a fixed slot cadence, then serving only cache on request, keeps observability from amplifying load or coupling metric collection to untrusted polling.

### Handshake as a bootstrap experiment

The existing N2N handshake already exposes observability-adjacent data (supported N2N versions, peer-sharing capability) and shows that short capability queries need not start long chain-sync work. A handshake extension remains a fair bootstrap experiment.

### Dedicated mini-protocol as the target

Handshake exists for protocol compatibility and connection negotiation. Observability can grow to include version info, resources, chain/state, encrypted sets, namespaced telemetry, proxied producer pubs, and later extensions. Putting that into the handshake couples monitoring to connection setup.

A dedicated mini-protocol keeps handshake for negotiation, lets observability version independently, gives it its own resource and message-size limits, and fits encrypted and proxied sets without bloating handshake version data.

### Why sealed boxes and on-chain provenance

Recipient-specific confidentiality is the first privacy need for richer fields. Libsodium sealed boxes give that without forcing a new key hierarchy on operators. They do not authenticate the publisher, so this CIP treats on-chain registered relay endpoints as operational provenance, not cryptographic attestation. Stronger attestation, if required later, should be a separate mechanism and must not casually reuse cold, KES, or VRF keys.

### Why topic-prefixed logical field IDs

Topic prefixes (`node_`, `system_`, `chain_`) keep the shared vocabulary readable in operator config while staying distinct from implementation namespaces (`cardano_node.*`, `amaru.*`). JSON examples document the logical registry; CBOR wire can use compact integer keys mapped to those IDs so string names need not ride every message.

### Relationship to block-embedded version markers

Block-embedded / header version markers and this relay observability model are complementary, not substitutes.

- block markers observe software tied to block production;
- relay observability covers non-producing nodes and a broader operator-selected dataset;
- encrypted pubs can share detail with authorized monitors only;
- independent methods can corroborate each other.

Pick by analysis goal. Trust limits on self-reported data: see trust model and known limitations below.

---



### Known limitations

Intentional or unavoidable limits of a first baseline. Treat them as consumer design constraints, not envelope bugs.

#### Self-reported fields are not proofs

Values such as implementation name, version, and config flags are **self-reported**.

Same class as block-embedded / header version markers: no cryptographic proof that fields match the binary or codebase on the host. A misconfigured, compromised, or malicious node can lie.

[Confidentiality, authenticity, and trust model](#confidentiality-authenticity-and-trust-model) already separates confidentiality from authenticity.

Still useful at network scale: if observability is default-on like handshake info, with an explicit operator opt-out, monitors can sample most public relays. Aggregate tip, software mix, and config trends get robust; isolated fake pubs have relatively small impact when monitors cross-check many registered endpoints and compare signals over time.

Read each publication as a claim from a reachable endpoint. Draw network conclusions from coverage, repetition, and corroboration.

#### Connection admission and transient unavailability

Relays enforce inbound connection limits and will not always accept a new peer.

An observability session may fail even when the relay is healthy and serving diffusion. The request is short and read-only against cache, so it should not hold a slot long or displace ordinary peers when [Security and resource considerations](#security-and-resource-considerations) limits and timeouts apply.

Treat refusal, handshake failure, and mini-protocol timeout as normal. Retry with backoff and rotate across a pool's registered relays. Fresh snapshots usually arrive without producer-network access.

---



## Path to Active



### Acceptance Criteria

This CIP may become Active when all of the following are met:

1. A dedicated read-only observability mini-protocol is specified with versioned CBOR/CDDL (or an equivalent Ouroboros-network-native encoding) consistent with this logical model.
2. At least one Cardano node implementation can expose open and encrypted publications from a publicly reachable relay according to this CIP, with operator opt-in or opt-out.
3. Operator configuration for field selection, recipients, and optional producer proxying is documented for that implementation.
4. At least one monitoring consumer can retrieve publications from on-chain registered relay endpoints and interpret `source` / `snapshot_slot` correctly.
5. Interoperability evidence exists: either a second independent implementation, or published golden vectors for the field registry and encrypted publication round-trip.



### Implementation Plan

1. Freeze the logical publication model and initial field vocabulary with node and networking reviewers.
2. Specify the mini-protocol messages and CDDL in coordination with Ouroboros Network maintainers (optional early handshake experiment allowed, not required for Active).
3. Implement relay publication generation, caching, and encryption in at least one node stack; add producer proxy retrieval where topology config allows.
4. Ship operator docs and a reference monitoring client that uses on-chain relay registration for discovery.
5. Collect review feedback on provisional fields (`chain_state_hash`, `chain_network`, system resource semantics) and amend this CIP before declaring the baseline normative.

Implementors are listed in the preamble when teams commit; currently none are formally signed up (`Implementors: []`).

## References

- Cardano stake pool registration and on-chain relay endpoints:
[https://developers.cardano.org/docs/operate-a-stake-pool/block-producer/register-stake-pool/](https://developers.cardano.org/docs/operate-a-stake-pool/block-producer/register-stake-pool/)
- CIP-5 common Bech32 prefixes, including the `pool` identifier:
[https://cips.cardano.org/cip/CIP-5](https://cips.cardano.org/cip/CIP-5)
- Ouroboros Network repository and mini-protocol architecture:
[https://github.com/IntersectMBO/ouroboros-network](https://github.com/IntersectMBO/ouroboros-network)
- Ouroboros Network changelog (N2N versioning and mini-protocol work):
[https://github.com/IntersectMBO/ouroboros-network/blob/main/ouroboros-network/CHANGELOG.md](https://github.com/IntersectMBO/ouroboros-network/blob/main/ouroboros-network/CHANGELOG.md)
- Cardano networking protocol overview:
[https://docs.cardano.org/about-cardano/explore-more/cardano-network/networking-protocol](https://docs.cardano.org/about-cardano/explore-more/cardano-network/networking-protocol)
- Cardano time and slot handling:
[https://docs.cardano.org/about-cardano/explore-more/time](https://docs.cardano.org/about-cardano/explore-more/time)
- Libsodium sealed boxes:
[https://doc.libsodium.org/public-key_cryptography/sealed_boxes](https://doc.libsodium.org/public-key_cryptography/sealed_boxes)



## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).