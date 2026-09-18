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

It defines a common configuration and publication model in which relays are the public contact point. Publications are slot-aligned cached snapshots retrieved through a dedicated read-only node-to-node mini-protocol. Relays may blindly forward encrypted producer publications without publishing pool identity on the wire. The model supports:

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
- [Publication identity and multi-pool relays](#publication-identity-and-multi-pool-relays)
- [Implementation guidelines](#implementation-guidelines)
- [Operator guidance](#operator-guidance)
- [Transport and mini-protocol integration](#transport-and-mini-protocol-integration)
- [Security and resource considerations](#security-and-resource-considerations)
- [Concise normative description](#concise-normative-description)



### Goals and Design Principles

1. **Ouroboros Network mini-protocol extension**
  - deliver observability over a dedicated read-only N2N mini-protocol in the Ouroboros Network stack;
  - keep handshake for connection negotiation; do not overload it with growing monitoring payloads;
  - version the observability protocol independently (CBOR/CDDL wire profile expected).
2. **Relay-centric public reachability**
  - monitors query relays;
  - block producers need no public monitoring exposure.
3. **Operator-controlled disclosure**
  - implementations define which fields they can expose;
  - operators choose which of those are open vs encrypted.
4. **Stable publication envelopes**
  - outer structure is standardized (`type`, `snapshot_slot`, payload or ciphertext);
  - **Do not** put pool identity on the public envelope;
  - payload fields may differ by implementation and operator config.
5. **Cached, request-independent generation**
  - publications are built on a slot-aligned cadence;
  - requests only read the cache;
  - **Do not** let external requests trigger metric collection, encryption, or producer queries.
6. **Blind, source-safe proxying**
  - relays may fetch publications from configured internal endpoints and forward them unchanged;
  - **Do not** expose internal addresses or ports to monitors;
  - **Do not** require the relay to decrypt or relabel proxied ciphertext;
  - pool identity, if any, lives in plaintext inside the issuing node's payload (typically encrypted).
7. **Extensible but interoperable fields**
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
          "chain_pool_bech32",
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

The relay selects endpoints by local configuration (which internal IP/port to query). It **blindly** retrieves and forwards that node's already-cached publications. It does not need a public pool label on the wire.

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

| field | description |
| --- | --- |
| `monitoring-proxied` | marks this internal endpoint as eligible for observability proxying |
| `poolBech32Id` | optional **local-only** hint for operator tooling, logging, or HA grouping |

`poolBech32Id`, if present, is relay-local configuration. **Do not** copy it into the public publication envelope. Pool identity for monitors belongs in the issuing node's plaintext payload (see `chain_pool_bech32`), usually under encryption.

Internal `address`, `port`, and local `description` are config/transport only. **Do not** copy them into published observability data.

#### Alternative node implementations

Other implementations need not use `topology.json` or this exact field placement.

They need any native config that can express:

- which internal producer endpoint is eligible for proxying;
- how the relay reaches it.

Optional local labels (including a pool id for ops) are fine. They are not part of the public protocol.

The CIP standardizes observable behavior, not one topology file format.

---



### Publication Output Format

The relay returns an array of publication objects. From a public N2N perspective this is simply a bag of publications: open and/or encrypted. Encrypted items do **not** advertise whether they describe the relay or a proxied producer, nor which pool they belong to.

Envelope metadata (not operator-selectable payload fields):

- `snapshot_slot`

Optional pool identity belongs in the **plaintext payload** as `chain_pool_bech32` (see field set), typically only in encrypted publications.

#### Open publication example

Open publications are from the node being queried (normally the public relay). They should not be used to proxy producer identity.

```json
{
  "type": "open",
  "snapshot_slot": 123456600,
  "payload": {
    "node_name": "cardano-node",
    "node_version_major": 11
  }
}
```

#### Encrypted publication example

Ciphertext is opaque on the wire. After decryption, plaintext may include optional `chain_pool_bech32` if the issuing node configured a pool id.

```json
{
  "type": "encrypted",
  "snapshot_slot": 123456600,
  "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
  "ciphertext": "BASE64_CIPHERTEXT"
}
```

Example plaintext inside that ciphertext (not visible publicly):

```json
{
  "node_name": "cardano-node",
  "node_type": "block_producer",
  "chain_pool_bech32": "pool1abcdefghijklmnopqrstuvwxyz1234567890abcdefghi",
  "chain_tip_slot": 123456589,
  "chain_block_height": 9876543
}
```

#### Full example response

Relay self open + encrypted, plus one blindly forwarded producer ciphertext. Public observers cannot tell which encrypted item is which.

```json
{
  "publications": [
    {
      "type": "open",
      "snapshot_slot": 123456600,
      "payload": {
        "node_name": "cardano-node",
        "node_version_major": 11
      }
    },
    {
      "type": "encrypted",
      "snapshot_slot": 123456600,
      "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
      "ciphertext": "BASE64_CIPHERTEXT_A"
    },
    {
      "type": "encrypted",
      "snapshot_slot": 123456600,
      "recipient_public_key": "BASE64_PUBLIC_KEY_MONITOR_A",
      "ciphertext": "BASE64_CIPHERTEXT_B"
    }
  ]
}
```

Proxied ciphertext is produced by the internal node. The relay forwards it unchanged and does not decrypt or rewrite it.

**Recommendation:** proxied producer publications should be **encrypted-only**. Open proxied producer payloads would re-expose producer data on the public relay without a strong privacy story.

---

### Mandatory Publication Metadata

#### `snapshot_slot`

Every publication has a `snapshot_slot` on the outer envelope.

It names the slot-aligned observation window for the cached publication.

It is not the chain tip:

- `snapshot_slot` = when the monitoring snapshot was scheduled;
- `chain_tip_slot` (optional payload field) = slot of the node's currently selected tip.

That split matters for nodes that are syncing or temporarily behind.

#### No outer `source` / pool id

There is **no** mandatory outer `source` field.

Pool affiliation is optional payload field `chain_pool_bech32`, set by the **issuing node** when it builds plaintext (before encryption). The public envelope must not carry `self` vs pool Bech32 labels.

Consequences:

- unauthorized N2N observers see only opaque encrypted blobs (plus any open relay fields);
- authorized recipients decrypt, then read `chain_pool_bech32` if present;
- monitors disambiguate multi-item responses after decryption, not from envelope metadata.

#### Privacy consideration

Publication count and ciphertext sizes remain visible. That is weaker leakage than publishing pool ids, but not zero. Padding or other cover traffic can be defined later if needed.

Open publications always describe the answering node. Operators who want producer privacy should keep producer pubs encrypted and omit pool id from any open payload.

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

When `localRootProxy` is enabled, the relay periodically retrieves already-generated publications from eligible internal endpoints over the private N2N path.

Timing follows [Slot-aligned snapshot generation and caching](#slot-aligned-snapshot-generation-and-caching): the producer must have its cache ready before the relay queries it; the relay retrieves near the snapshot boundary together with its own generation.

The internal node owns its templates, field selection, and encryption. The relay:

- queries the configured internal address/port;
- accepts the returned publication set blindly;
- forwards selected items into its public cache **unchanged**;
- does **not** ask for individual fields;
- does **not** decrypt ciphertext;
- does **not** attach or rewrite pool identity on the envelope.

So:

- producer field sets may differ from the relay's;
- different producers may differ from each other;
- the relay is retrieval/cache/forward only;
- public responses are a bag of pubs with no outer producer labels.

If the producer includes `chain_pool_bech32` in its plaintext before encryption, only authorized recipients learn that affiliation after decrypt.

Public monitor requests never cause the relay to query the producer (same mini-protocol path; see [Transport and mini-protocol integration](#transport-and-mini-protocol-integration)).

#### Failure behavior

Producer retrieval failure must not block relay-local publications.

If a proxied endpoint is unavailable:

- log locally;
- omit that endpoint's items from the current response unless the implementation explicitly serves stale cache;
- if stale data is returned, keep the original `snapshot_slot` so consumers see the age.

#### Multiple producer endpoints (HA)

A topology may list more than one eligible internal endpoint (active/standby, HA).

Baseline:

- try them in a stable implementation-defined order (config order is a fine default);
- use publications from the first reachable endpoint that returns a valid set for that snapshot;
- optional local `poolBech32Id` may help operators group endpoints for logging, but it is not published.

Explicit priority/weighting can wait until ops experience shows a need.

---

### Draft Default Field Set

Proposed **initial shared field vocabulary**. Draft names, types, and semantics for node teams to refine before this becomes normative.

Standard fields use **topic prefixes** for grouping (`node_`, `system_`, `chain_`). These prefixes are part of the CIP baseline, not implementation namespaces (see [Extensible field model](#extensible-field-model)).

| field | proposed type | draft semantic definition |
| --- | --- | --- |
| `node_name` | string | stable name of the node implementation, e.g. `cardano-node`, `amaru` |
| `node_version_major` | unsigned integer | major component of the node software version |
| `node_version_minor` | unsigned integer | minor component of the node software version |
| `node_version_patch` | unsigned integer | patch component of the node software version |
| `node_type` | string | operational role, e.g. relay or block producer |
| `node_n2n_supported_versions` | array | N2N protocol versions currently supported |
| `node_peer_sharing_enabled` | boolean | whether peer sharing is enabled/available per negotiated/configured capability |
| `system_cores` | unsigned integer | logical CPU units available to the node process; respect container/cgroup limits where applicable |
| `system_memory` | unsigned integer | memory in bytes available to the node process; effective container/cgroup limit before host RAM where applicable |
| `chain_pool_bech32` | string | optional CIP-5 `pool` Bech32 id of the stake pool this node represents; set by the issuing node when configured; prefer encrypted publications |
| `chain_state_hash` | string | provisional common state identifier; domain, algorithm, encoding, and cross-impl semantics need node-team agreement |
| `chain_tip_slot` | unsigned integer | slot of the node's currently selected chain tip |
| `chain_block_height` | unsigned integer | block number/height of the currently selected tip |
| `chain_network` | string | network identifier; canonical form needs agreement |

When a producer (or relay) knows its pool id, it may auto-include `chain_pool_bech32` in plaintext before encryption. Relays must not inject this field into proxied ciphertext.

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
- node role terminology across implementations;
- whether `chain_pool_bech32` should ever appear in open payloads (baseline recommendation: encrypted only).

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
chain_pool_bech32
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
4. decrypt encrypted items addressed to it;
5. if plaintext contains `chain_pool_bech32`, check that the queried relay is currently registered on-chain for that pool identity.

That is useful operational provenance: the monitor chose an endpoint the pool operator registered, then read a self-reported pool id only after decryption.

It is **not** cryptographic authentication of the live TCP/N2N peer or of the payload. Inner `chain_pool_bech32` is still self-reported.

Baseline assumptions:

- on-chain endpoint registration;
- normal routing/DNS;
- operator control of the registered relay.

If a use case needs cryptographic proof that a node or pool signed a snapshot, define a separate authenticated publication mechanism. **Do not** casually reuse cold, KES, or VRF keys.

---

### Publication Identity and Multi-Pool Relays

Public envelopes do not name which publication is the relay vs which producer, and do not list pool ids.

Authorized monitors learn identity from plaintext after decrypt (`chain_pool_bech32`, `node_type`, and related fields).

That is especially important when:

- one relay proxies more than one producer;
- a multi-pool operator forwards several internal nodes through one public relay;
- one response carries multiple encrypted items for the same recipient.

Monitor policy:

- decrypt all items addressed to the configured recipient key;
- group by `chain_pool_bech32` when present;
- treat missing pool id as relay-local or unlabeled;
- reject or mark untrusted a decrypted pool id that cannot be associated with the queried relay in the current on-chain registration view.

Open publications are attributed to the answering relay by connection context, not by an outer `source` field.

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

- always include `snapshot_slot` on the envelope;
- **Do not** put pool identity or `source` on the outer envelope;
- keep envelope metadata outside the operator-selectable payload field list.



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
5. marking which internal endpoints are eligible for proxy retrieval;
6. on producer (and optionally relay) nodes, configuring pool id so `chain_pool_bech32` can be auto-included in plaintext.

Weigh privacy for every selected field.

A reasonable pattern:

- open (relay): implementation name + major version only;
- encrypted: full version, system characteristics, richer state, and `chain_pool_bech32` when known;
- proxied producer pubs: encrypted-only;
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

Producer serves its cache (plaintext already includes optional `chain_pool_bech32` before encryption). Relay retrieves on its cadence, forwards items unchanged into its public cache. Runtime rules: see [Proxy retrieval and caching](#proxy-retrieval-and-caching).

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
- recipient public keys on encrypted items.

It does hide pool identity and relay-vs-producer assignment on encrypted items until decryption.

If remaining metadata becomes sensitive later, define padding or other privacy measures separately.

---



### Concise normative description

A participating node builds a cached observability publication set on a slot-aligned cadence. Each publication has mandatory envelope `snapshot_slot` plus either an open payload or recipient-specific ciphertext. There is no outer `source` or public pool id.

Operators select payload fields only from fields the implementation supports. The node builds values and structure; no operator JSON templates or placeholders. Optional `chain_pool_bech32` is set by the issuing node in plaintext when configured, preferably under encryption.

Relays may retrieve cached publications from configured private endpoints and forward them unchanged. Producer field selection and encryption are the producer's own. Public relay requests must never synchronously trigger producer retrieval.

For `topology.json` deployments, `localRoots[].accessPoints[]` with a proxy marker is one proposed mapping. Optional local `poolBech32Id` is ops-only and must not be copied to the public envelope. Other implementations may use equivalent native config.

Preferred transport: dedicated read-only N2N observability mini-protocol. Field names here are logical IDs (config/docs/registry); wire format is separately versioned CBOR/CDDL, typically with compact integer keys mapped to those IDs.

Encryption gives recipient confidentiality, not publisher authentication. Monitors should use on-chain registered relay endpoints as baseline operational provenance, then bind decrypted `chain_pool_bech32` (if present) to that registration view. Strong node attestation, if needed later, is a separate mechanism.

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

Recipient-specific confidentiality is the first privacy need for richer fields and for pool identity. Libsodium sealed boxes give that without forcing a new key hierarchy on operators. Pool id is optional plaintext (`chain_pool_bech32`), not an outer envelope label, so unauthorized N2N observers do not see which encrypted item belongs to which producer.

Sealed boxes do not authenticate the publisher, so this CIP treats on-chain registered relay endpoints as operational provenance, then lets monitors reconcile decrypted pool ids with that registration view. Stronger attestation, if required later, should be a separate mechanism and must not casually reuse cold, KES, or VRF keys.

### Why blind proxy without outer `source`

The relay must forward already-encrypted producer publications without decrypting them. Putting pool id on the envelope would force either public disclosure or relay-side re-encryption. Blind forward plus optional inner `chain_pool_bech32` keeps proxy simple and keeps producer affiliation visible only to authorized recipients.

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
4. At least one monitoring consumer can retrieve publications from on-chain registered relay endpoints, interpret `snapshot_slot`, decrypt recipient pubs, and bind optional `chain_pool_bech32` to registration data.
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