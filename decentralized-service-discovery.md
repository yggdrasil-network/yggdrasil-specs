# Decentralized Service Discovery

Status: Proposed (document ID pending review)

## Abstract

This proposal defines discovery of existing Yggdrasil services without a central
registry. Publishers sign complete service snapshots with their Yggdrasil Ed25519
identity. Participating nodes validate, cache and optionally relay those snapshots
over existing peer connections. Applications obtain endpoints from a local API.

The proof of concept is [floscodes/yggdrasil-go, feat/discovery](https://github.com/floscodes/yggdrasil-go/tree/feat/discovery).
This proposal is based on commit
[`1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0`](https://github.com/floscodes/yggdrasil-go/commit/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0).
All source citations below are pinned to that revision.

The terms MUST, MUST NOT, SHOULD and MAY express requirements of this proposed
protocol. The cited implementation is experimental; this document does not claim
that the protocol or its numeric assignments have been accepted upstream.
The refresh ceiling, capacity policy and overload requirements below are proposed
changes beyond the pinned prototype and are identified as such.

## Motivation and scope

A user should be able to discover a named service, its IPv6 address, port and
transport without consulting a separately operated directory. Names belong to a
publisher's public key; there is no global name authority or uniqueness guarantee.

This mechanism is independent of multicast peer discovery (YS003): it uses peers
that are already connected rather than finding new peers. It is also independent
of request-driven node information (YS004). It does not create routes, DNS records,
port forwards, application proxies or peering connections. Applications continue
to connect to ordinary IPv6 endpoints.

Discovery is best effort. A result is an authenticated advertisement, not evidence
of current reachability, application availability or trustworthiness.

## Participation and local configuration

An implementation MUST provide control over participation and relaying:

| Setting | Proof-of-concept default | Meaning |
|:--|:--|:--|
| `Enabled` | `true` | Accept announcements and publish configured services |
| `Forward` | `true` | Relay accepted foreign announcements |
| `RefreshInterval` | `"15m"` | Interval between newly signed local snapshots |
| `Services` | empty list | Complete set of locally advertised services |

Disabling discovery MUST stop publication, acceptance and forwarding.
`Forward` has no effect while discovery is disabled. A node with no configured
services MUST NOT originate announcements during normal operation; it can still
receive and relay announcements. Disabling forwarding only controls that node:
recipients may relay its own advertisements further.

Publishers MUST accept configured refresh intervals from 5 through 30 minutes,
inclusive, and MUST reject values outside that range. An omitted or empty interval
SHOULD select 15 minutes. The 30-minute ceiling leaves at least half of the
60-minute lifetime for delivery of a scheduled renewal. At exactly 30 minutes,
losing that renewal can still cause a gap because the next one is due at expiry;
the 15-minute default provides more retry opportunities. Delivery is best effort. The pinned proof of concept instead permits up to 60 minutes
and needs adjustment to meet this proposal. Invalid intervals,
invalid local services and an oversized local announcement cause startup failure,
including when discovery is disabled. Configuration changes require a restart.
These defaults and configuration syntax are implementation policy proposed for
review, rather than prerequisites for parsing the wire format.
See [configuration](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/config/config.go)
and [initialization](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go#L99-L138).

For example, the following HJSON config advertises a wiki. The address MUST be
replaced by this publisher's actual node address or an address in its derived
routed subnet:

```hjson
Discovery: {
  Enabled: true
  Forward: true
  RefreshInterval: "15m"
  Services: [
    {
      Name: "wiki"
      Address: "300:1234:5678:abcd::10"
      Port: 80
      Protocol: "tcp"
    }
  ]
}
```

## Service identity and address validation

A service is identified by the pair `(publisher public key, Name)`. Two publishers
MAY use the same name. A snapshot MUST contain at most 32 services with distinct,
case-sensitive names. No Unicode normalization is performed by the prototype.

Each service contains these JSON members:

| Member | Type | Validation |
|:--|:--|:--|
| `Name` | string | Nonempty, at most 128 UTF-8 bytes, valid UTF-8, no Unicode control characters, no leading or trailing Unicode whitespace |
| `Address` | string | IPv6 literal without a zone identifier, bound to the publisher as below |
| `Port` | integer | 1 through 65535 |
| `Protocol` | string | Exactly `tcp`, `udp` or `tcp+udp` |

`tcp+udp` means both transports on the same port. Application protocols such as
HTTP are not transport identifiers.

An address MUST equal the publisher's Yggdrasil key-derived node address or belong
to its key-derived routed /64 subnet, using the existing Yggdrasil address
derivation. This binds endpoints to the signing identity's address space.
A router can advertise devices in its routed subnet. An independently identified
node must advertise under its own key unless using an address from that router's
subnet.

Receivers MUST check ownership against the original publisher's key, even when
another peer relays the message. If any service is invalid, the entire snapshot
MUST be rejected without caching or forwarding.

The prototype's address check is quoted verbatim from
[`validateDiscoveryServices`](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go#L68-L97):

```go
ip, err := netip.ParseAddr(s.Address)
if err != nil || !ip.Is6() || ip.Zone() != "" || (ip != own && !subnet.Contains(ip)) {
    return fmt.Errorf("service %q address %q must be the publisher address %s or belong to its routed subnet %s", s.Name, s.Address, own, subnet)
}
```

Indentation in quoted Go excerpts is normalized for readability.

## Transport and announcement format

An announcement travels inside an encrypted Yggdrasil session addressed to the
public key of a directly connected peer. The prototype prefixes the announcement
with two bytes: `typeSessionProto = 2` and `typeProtoDiscovery = 3`.
These are proposed assignments, subject to upstream review. Receivers without a
discovery handler ignore the unrecognized subtype in the prototype's dispatcher.
No additional IP socket, multicast group or discovery port is required.
See [packet type definitions](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/types.go)
and [protocol dispatch](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/proto.go#L56-L74).

The following sender excerpt is from
[`send`](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go):

```go
bs := append([]byte{typeSessionProto, typeProtoDiscovery}, wire...)
_, _ = d.core.PacketConn.WriteTo(bs, iwt.Addr(to[:]))
```

The announcement body MUST be at most 8192 bytes, excluding the two prefix bytes.
This byte limit and the 32-service limit MUST both hold for the complete serialized
envelope, including its signature, JSON escaping and whitespace. A valid name
can expand during serialization; 32 individually valid services are not guaranteed
to fit. A publisher MUST reject an oversized snapshot with an actionable error.
It MUST NOT silently truncate the list or split it across independent announcements:
each accepted announcement replaces the entire snapshot. Before sending, an
implementation MUST also ensure that the body plus both framing bytes fits its
encrypted session MTU, and MUST handle send errors without unbounded retries.
The TUN interface MTU does not determine this in-band message limit.

It is a UTF-8 JSON object with members `Payload` and `Signature`. `Payload` is
an embedded JSON object, not a quoted JSON string or base64 string. `Signature`
is the standard padded base64 encoding of a 64-byte Ed25519 signature, as produced
by Go's `encoding/json` for `[]byte`.

The payload has these members:

| Member | Type | Meaning |
|:--|:--|:--|
| `Version` | integer | MUST be 1 |
| `Key` | string | Standard padded base64 of the publisher's 32-byte Ed25519 public key |
| `Issued` | signed 64-bit integer | Publication time in Unix seconds |
| `Services` | array | Complete service snapshot using the fields above |

The prototype also accepts `Services: null` as an empty snapshot; its Go encoder
emits null for a nil service slice. An empty snapshot withdraws all services for
that publisher when it has a newer timestamp.

For interoperable emission, senders SHOULD use the member spelling and types
above. The prototype's Go decoder tolerates unknown fields, omitted zero-valued
fields and case-insensitive struct-field matches. Duplicate-member handling and
strict schema validation need resolution before standardization (see below).

### Signature input

The signature MUST cover the concatenation of:

1. The ASCII bytes of `yggdrasil-service-discovery-v1`, followed by one zero byte.
2. The exact JSON bytes of the embedded `Payload` object.

There is no JSON canonicalization step. A verifier MUST use the received payload
bytes, not decode and reserialize the object before verification. Relays MUST
preserve the signed payload; the prototype forwards the entire received
announcement unchanged. Formatting changes inside the payload invalidate the
signature. The framing bytes and outer JSON formatting are not signed.

The following excerpts from
[`announce`](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go)
show the payload construction and domain-separated signature:

```go
p := discoveryPayload{Version: 1, Key: d.core.public, Issued: now.Unix(), Services: d.core.config.discovery.Services}
payload, err := json.Marshal(p)
```

After checking the marshaling error, the prototype wraps the payload:

```go
wire, err := json.Marshal(discoveryAnnouncement{payload, ed25519.Sign(d.core.secret, append([]byte(discoveryDomain), payload...))})
```

## Receive, cache and relay algorithm

For an incoming announcement, a participating receiver MUST:

1. Reject a body larger than 8192 bytes or a sender whose session public key is
   not currently associated with an authenticated direct peer connection.
2. Apply the pre-verification overload controls below. Decode the envelope and
   payload; reject malformed JSON.
3. Require version 1 and a 32-byte publisher key. With `now` in Unix seconds,
   require `now - 3600 < Issued <= now + 30`.
4. Reject its own publisher key and validate all advertised services.
5. Verify the Ed25519 signature over the domain prefix and raw payload bytes.
6. Remove expired cache entries. If a record for this publisher exists, require
   a strictly greater `Issued` value. If no record exists and the local capacity
   is exhausted, reject the new publisher. The default capacity
   is 1024 publishers including any reserved local publisher slot, as defined below.
7. Atomically replace that publisher's complete cached snapshot and retain its
   original wire representation.
8. If forwarding is enabled, attempt to send the accepted announcement at most
   once to each currently connected peer, excluding the incoming peer and the
   original publisher, subject to the bounded relay budgets below. Exhausted
   budgets MAY cause eligible sends to be dropped.

The source of a session is the immediate relay; `Payload.Key` identifies the
publisher. They do not need to match. Direct peer membership is tracked around
the authenticated connection's lifetime; multiple links to the same public key
are counted and yield one discovery peer. See
[link integration](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/link.go#L700-L702).

The timestamp and capacity gate in
[`handle`](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go) is:

```go
old, exists := d.records[origin]
if (exists && old.payload.Issued >= p.Issued) || (!exists && len(d.records) >= discoveryMaxOrigins) {
    d.mu.Unlock()
    return
}
```

The cache keeps one snapshot per publisher, not one record per service.
A changed snapshot replaces the list in full. A renewal with an identical list
updates the signed timestamp and expiry without adding duplicate services.
An empty snapshot leaves a timestamp-bearing record until expiry, so older
nonempty snapshots cannot immediately restore withdrawn services.

Duplicate and older snapshots MUST NOT be forwarded. A newer timestamp is a new
announcement and MAY be relayed again even if its service list is unchanged.
This suppresses loops while retaining periodic renewal. There is no hop limit.

A node MUST NOT periodically rebroadcast cached foreign snapshots or replay them
to newly connected peers. Only acceptance of a newer foreign snapshot triggers
its relay. Each relay send is an attempt, without acknowledgment or retry.
Consequently a lost relay packet can leave downstream nodes unaware of a service
until a subsequent publisher renewal.

## Publication, renewal and expiration

An enabled publisher with at least one service creates a snapshot immediately
when its discovery worker starts, then renews it at the configured interval.
A renewal uses a newly signed timestamp even if the service list is unchanged.

The prototype checks peer connections at most five seconds apart. A newly observed
peer receives the publisher's own current snapshot on its first three checks;
regular renewal also sends to all current peers. Multiple direct links to the
same public key do not multiply sends. This retry policy helps session setup but
does not guarantee delivery. Foreign records are excluded from these sends.
See [publication loop](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go).

A snapshot expires at `Issued + 3600` seconds, rather than an hour after receipt.
Receiving or relaying it MUST NOT extend this lifetime. Expired records MUST NOT
appear in local results. The prototype cleans the cache in its worker and before
returning results, and also before accepting a validated incoming snapshot.

Clocks should be synchronized. A timestamp up to 30 seconds ahead is accepted,
so the remaining lifetime at receipt can slightly exceed an hour. The proposed
30-minute refresh ceiling leaves headroom for delivery delay. A lost renewal
can still cause expiration, particularly at the 30-minute setting. The prototype's 60-minute ceiling provides no
such headroom and can cause gaps even with a small transmission delay.

The protocol supports signed empty withdrawals, but the prototype's publication
loop never originates them: removing the last service and restarting stops
publication, leaving prior advertisements to expire naturally. Shutdown also
does not send a withdrawal. A restart within the same Unix second may produce
a changed snapshot that receivers reject as equal-timestamped.

## Local consumer interface

The prototype exposes [`Core.GetServices()`](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery.go)
and [the admin `getServices` handler](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/admin/getservices.go).
For example:

```sh
yggdrasilctl getServices
yggdrasilctl -json getServices
```

The admin response body contains a `Services` array. Each entry contains
`Name`, `Address`, `Port`, `Protocol`, `PublicKey` and `Expires`.
`PublicKey` is hexadecimal here (unlike the base64 key in wire payloads);
`Expires` is the JSON timestamp produced by Go's `time.Time` encoder.
Results are sorted by publisher key, then service name, and include the local
publisher's cached services after its worker has created them. Disabled discovery
returns an empty array.

Consumers SHOULD retain the publisher key when displaying or selecting a service.
Names are not automatically resolved and MUST NOT be treated as globally trusted
DNS names. An alternative implementation MAY provide a different local API;
the announcement format and validation rules define network interoperability.

## Security and resource considerations

Signatures authenticate the publisher and protect relayed content. Address
validation prevents a publisher from advertising another identity's node address
or routed subnet. Neither mechanism establishes application ownership, uptime,
honesty or suitability for a user.

All participating peers can inspect and redistribute announcements. Encrypted
session transport does not make advertised services private to a chosen audience.
Do not publish sensitive service names or endpoints based solely on `Forward:
false`. The feature uses existing session transport and adds no independent
access-control mechanism.

### Size and capacity policy

Retain the limits of 32 services, 128 UTF-8 bytes per name and 8192 bytes per
serialized announcement. Compact encoding leaves useful room for ordinary
service lists without increasing the maximum relayed message size.

For illustration, using a 10-digit timestamp, a 38-character IPv6 address, port
65535, transport `tcp+udp`, and standard base64 key and signature fields gives
these calculated envelope sizes with compact JSON and Go-style HTML escaping:

| Service list | Envelope bytes |
|:--|--:|
| One service named `wiki-00` | 318 |
| 32 services named `wiki-00` through `wiki-31` | 3542 |
| 32 distinct 128-byte names, each a two-digit index followed by 126 ASCII `a` characters | 7414 |
| Ten distinct 128-byte names, each a two-digit index followed by 126 `<` characters | 8764 (rejected) |

These are size calculations, not performance measurements or protocol test
vectors. The escaping behavior is documented by
[Go's JSON encoder](https://pkg.go.dev/encoding/json#Marshal).
In the pinned transport, Yggdrasil configures a 131070-byte peer message limit;
Ironwood subtracts routing and encryption overhead when reporting its session
MTU. An 8194-byte discovery write is comfortably below that MTU. This does not
mean it fits into a single underlying IP packet. See
[Yggdrasil transport setup](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/core.go),
[Ironwood network MTU](https://github.com/Arceliar/ironwood/blob/0712eccbb962/network/packetconn.go)
and [encrypted MTU](https://github.com/Arceliar/ironwood/blob/0712eccbb962/encrypted/packetconn.go).

Publisher capacity is a local resource policy, not a network-wide publisher limit
or a promise of complete discovery. Implementations SHOULD default to 1024 cached
publishers and MAY expose a configurable finite capacity. They MUST enforce the
configured capacity atomically, including empty withdrawal records and the local
publisher. If local publication is configured, one slot MUST be reserved before
foreign announcements can consume the remaining capacity. A new foreign origin
MUST be rejected when those slots are full; existing origins MAY still update
subject to the overload controls. Expired records MUST release their slots.

This reservation is a proposed correction: the prototype shares the map with its
local publisher but does not apply its foreign-origin capacity gate to local
insertion, so the map can exceed 1024 entries by one.

At the default capacity, retained wire bodies alone can occupy 8 MiB and the
service count can reach 32768. Decoded strings, structs, map entries, allocator
costs and temporary processing require additional memory. Implementations MUST
also bound pending work; the wire-body limit is not a process-memory limit.

A full cache can remain occupied by renewing publishers indefinitely. Different
nodes can retain different subsets of publishers, and a rejected origin is not
forwarded. Implementations MUST expose capacity rejections through local counters
or status so consumers can distinguish an empty result from known incomplete
coverage. A larger cache alone does not solve fairness or Sybil admission.

### Overload controls

Before stable deployment, implementations MUST bound discovery work independently
of ordinary application traffic. At minimum they MUST provide:

- Per-authenticated-peer and global ingress budgets, accounting for both message
  count and bytes before JSON decoding and signature verification. Invalid,
  duplicate and older messages MUST consume those budgets too.
- Bounded verification concurrency and bounded queues. Excess work MUST be dropped
  rather than accumulated indefinitely or allowed to monopolize application
  packet processing.
- Per-publisher budgets for accepting newer snapshots after authentication, plus
  global and per-outgoing-peer relay budgets for message count and bytes. Publisher
  budgets MUST NOT be allocated into an unbounded map for arbitrary claimed keys.
- Finite burst allowances and local counters for ingress, verification-queue and
  relay drops. Overload handling MUST NOT generate one response or log entry for
  every dropped packet.

Budgets MUST have documented finite defaults and SHOULD be configurable for node
capacity and expected discovery population. Exact rates require load measurements
before stable release; the size calculation here is not evidence for a safe CPU
budget. A publisher's configured five-minute minimum refresh interval MUST NOT be
assumed to constrain incoming traffic: the prototype does not enforce that
interval at reception, and valid timestamp sequences can arrive in a burst.

An implementation MAY skip repeated verification of an exact previously verified
announcement using a bounded cache of full-message digests. Unverified publisher
keys or timestamps MUST NOT advance replay state or suppress a later valid
snapshot. A different byte sequence still requires normal authentication.

For scale, 1024 publishers sending 8192-byte bodies every 15 minutes produce about
9.1 KiB/s per outgoing copy, including the two discovery framing bytes. Four
eligible outgoing peers require about 36.4 KiB/s; at five-minute refresh this
becomes about 109.3 KiB/s. These steady-state calculations exclude session and
link overhead, incoming duplicates, initial retries and synchronized bursts.
Relay budgets must account for fan-out rather than just ingress bytes.

The pinned prototype has none of these rate or queue controls in its discovery
handler and verifies duplicates before rejecting their timestamps. Packet and
cache size bounds alone therefore do not make it ready for stable deployment.

Replay suppression lasts only while a publisher's latest timestamp remains in
the cache. After cache loss or expiry, an otherwise valid unexpired snapshot can
be accepted again. This can also affect withdrawals after receiver restart.
A future-dated accepted snapshot can temporarily suppress a publisher's
lower-timestamped updates. Timestamp sequencing and clock rollback require care.

## Required validation and proof-of-concept evidence

Implementations MUST validate the following before stable release:

- Otherwise valid envelopes of exactly 8192 bytes pass the size gate; envelopes
  of 8193 bytes fail without caching or relaying. Both framing bytes must fit the
  session MTU, and send failures must remain bounded.
- Both service-count and byte limits apply, including 32 long names, JSON-escaped
  names, 128/129 UTF-8 byte name boundaries, and 32/33 service boundaries. An
  oversized local snapshot produces a clear error and no partial publication.
- Refresh configuration accepts 5, 15 and 30 minutes and rejects values above
  30 minutes or below five. Expiration remains 60 minutes from signed issuance.
- At full capacity, local publication, concurrent foreign insertion, empty
  withdrawals, renewals and expiration never exceed the configured publisher
  capacity. Capacity rejection is observable.
- Duplicate floods, forged signatures, rapid valid renewals and many claimed
  identities cannot bypass the budgets or grow queues or limiter state without
  bound. Relay fan-out stays within outgoing budgets, and application traffic
  continues to progress under discovery load.

These are required future implementation checks, not tests executed by this
documentation change.

### Existing evidence and remaining review questions

The [discovery test suite](https://github.com/floscodes/yggdrasil-go/blob/1c34b913e1c5ae9e8d99e54e1da9395cf732d2a0/src/core/discovery_test.go) contains tests for
service validation, endpoint ownership (including routed devices and foreign
relays), forwarding controls, forged signatures, timestamp replay, empty
withdrawals, expiry, automatic publication, configurable renewal, forwarding
once, silent operation with no services, and defaults. These are implementation
evidence, not a complete interoperability or adversarial conformance suite.
This proposal does not claim a fresh execution of those tests.

Before accepting a stable specification, reviewers should resolve:

- Allocation of a document ID and confirmation of the protocol subtype value.
- Whether default acceptance and forwarding should remain enabled.
- A strict cross-language JSON profile: duplicate members, unknown members,
  omitted fields, nulls, case matching, invalid Unicode and integer boundaries.
  Include signed test vectors with exact payload bytes and expected signatures.
- Whether publisher ordering needs a sequence number or stronger restart and
  clock-rollback semantics instead of second-resolution timestamps.
- Whether explicit withdrawal publication and durable replay state are needed.
- Measured defaults for the required rate budgets, queue bounds and burst
  allowances, plus admission fairness against Sybil-driven cache exhaustion.
- Whether best-effort renewal is sufficient, or a later version should add
  synchronization for newly connected peers and recovery from lost relays.
- Transport boundary and overload validation as required above, and compatibility
  testing with older Yggdrasil releases.

These questions are intentionally explicit: the proposal describes the working
mechanism at the pinned revision, identifies proposed hardening beyond that
revision, and leaves the remaining standardization decisions open for review.
