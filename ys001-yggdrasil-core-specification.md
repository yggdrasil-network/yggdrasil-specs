# YS001: Yggdrasil Core Specification

| ID    | Name                         | Version | Status | Authors                  | Date        |
|:------|:-----------------------------|:--------|:-------|:-------------------------|:------------|
| YS001 | Yggdrasil Core Specification | 0.1     | Draft  | Neil Alexander, Arceliar | 28-Oct-2019 |

## About this document

This document outlines the core behaviours of an Yggdrasil node. Note that this
document does not necessarily include implementation-specific detail, just the
general behaviours to which an Yggdrasil node should comply.

Please note the use of the terms **must**, **must not**, **should**, **should
not** and **may** throughout - all **must** and **must not** behaviours must be
met in order for an implementation to be compliant with this specification, and
all **should** and **should not** behaviours should ideally be met in order to
guarantee the best level of interoperability with existing implementations.

## Introduction

Yggdrasil is an experimental implementation of a routing scheme which aims to
guarantee end-to-end reachability between all network nodes in decentralised and
unpredictable network topologies, without relying on central points of
authority.

A globally-agreed spanning tree is formed between interconnected nodes to
provide each node with a dynamic locator, reducing the amount of knowledge that
a node must maintain about the network as a whole and eliminating the need for
large routing tables. A distributed hash table (DHT) is used to facilitate
discovering the locators of a given node where the address is already known.

The routing scheme is name-independent - that is, that the permanent address of
a given node is not permanently associated with the location of the node on the
network. This allows a certain amount of node mobility without relying on
subnet-based routing.

Yggdrasil is a greedy routing algorithm. All nodes on the network have the
capacity to forward traffic on behalf of other nodes so long as doing so brings
the traffic closer to the intended destination, using only local knowledge. In
practice, only nodes that are directly connected to more than one peer will
forward traffic on behalf of other nodes.

Knowing the destination locator of the traffic in question, and the locators of
all directly connected peers, a node can work out which egress peering will take
the traffic closer to the intended destination.

---

## Identities

In order for Yggdrasil to operate a zero-trust model, node identities are
generated cryptographically. These same cryptographic identities are then used
for signing and encryption operations.

Although it may often be beneficial for these identities to be persistent, there
is no hard requirement for this. A node can either load existing persistent
identities or generate new identities at startup, depending upon whether
permanence is required.

### Keypairs

An Yggdrasil node has two identities: an `ed25519` keypair (referred to as the
"signing keys") and a `curve25519` keypair (referred to as the "encryption
keys"). While `ed25519` keys can indeed be converted into `curve25519` keys,
there is no requirement for the two keypairs to be related - using separately
generated keypairs is a fully supported configuration.

The signing keys are critical in participating in the global spanning tree, both
for performing root elections and signing switch root updates, as well as to
generate a Tree ID. The encryption keys are used to encrypt traffic between
nodes, as well as to generate a Node ID.

### Tree ID

The Tree ID is a 64-byte identifier which is calculated by taking the SHA512 sum
of the node's public signing key.

### Node ID

The Node ID is a 64-byte identifier which is calculated by taking the SHA512 sum
of the node's public encryption key. The node's permanent address is derived
from the Node ID.

### Strength

When calculating the "strength" of a given Tree ID or Node ID, the entire
identifier is treated as a 512-bit integer. The "strongest" ID is the one with
the "highest" value, e.g. the highest number of leading one bits set.

---

## Wire protocol

The Yggdrasil protocol is a binary protocol. Every message has one or more
fields, matching one of the below base types. The first bytes of every message
represent a `varu64` (see below) which defines the message type code. The
structure of the message following that message type code must comply with that
message type as defined.

The protocol does not otherwise frame fields - that is, there are no markers
within a message which denote the start or end of a given field. It is the
responsibility of an implementation to know the format for a defined message
type and to be able to determine the end of a given field based on the
characteristics of that field type. Further details on this are available in the
base type definitions below.

### Base types

The Yggdrasil protocol defines the following base types:

- `varu64`: Variable-length unsigned 64-bit integer
- `vari64`: Variable-length signed 64-bit integer
- `bytes`: Variable-length byte array
- `coords`: Variable-length byte array containing coordinates

#### Unsigned 64-bit integers

All unsigned integers on the wire are encoded as big-endian `varu64` which, with
variable-length encoding, encode into a `bytes` array of at most 10 bytes. This
variable-length encoding mechanism ensures that small values will use less bytes
on the wire, and that in future, larger integer types can be used if
appropriate.

Each byte is composed of 8 bits where:

1. The most-significant bit (MSB) is a "continuation byte" - set to 1 if there
   is another byte following this one, or set to 0 if this is the last byte
2. The remaining 7 bits contain the next 7 least-significant bits of the source
   `varu64` value

##### Encoding

To encode a `varu64`, start with an empty byte array. Take the least-significant
7 bits of the source value and place them into the least-significant 7 bits of
your first byte, and then set the most-significant bit of the first byte to 0.

Repeat the process by right-shifting 7 bytes from the source value. If the
shifted source value is still greater than zero, append the next byte by taking
the next 7 least-significant bits and set the most-significant bit to 1.

The resultant bytes will be big-endian individually, however the order of the
byte array will be reversed as the beginning of the array now contains the
source value's least-significant bits. Therefore, to complete the encoding, the
order of the bytes must be reversed (preserving the order of the bits in each
byte) so that the entire sequence is big-endian and the last byte in the
sequence now has the zero most-significant bit.

##### Decoding

To decode a `varu64`, start with a target zero-value unsigned 64-bit integer.
Take the 7 least-significant bits of the first byte and copy them onto the 7
least-significant bits of the target. If the most-significant bit of the byte is
1, left-shift the target value by 7 bits and then move onto the next byte,
repeating this process until the most-significant bit is 0.

#### Signed 64-bit integers

All signed integers on the wire are encoded as `vari64`, using the same
compression as `varu64`, specified above. However, for signed integers, zig-zag
encoding is also used in order to preserve whether a value is positive or
negative.

Positive values of `x` are encoded as `2*x+0` and negative values of `x` are
encoded as `2*(^x)+1`, allowing negative numbers to be complemented - in this
case, bit 0 denotes whether to complement or not.  

#### Byte arrays

Byte arrays are written to the wire unmodified. There is no encoding or decoding
to be done when working with byte arrays.

As they are not prefixed with a length value, all byte arrays that occupy fields
before the last field of the message **must** be fixed-length and that length
must be specified and known. Byte arrays **must** only be variable-length if
they are the last field in the message.

#### Coordinates

Coordinates are encoded into byte arrays, however, they have special behaviour.
Coordinates are variable-length arrays of `varu64`s, therefore encoding
coordinates is a three-step process:

1. Encode each `varu64` element of the coordinates into a `bytes` array, as
   above
2. Concatenate all resultant `bytes` arrays into one contiguous `bytes` array
3. Prefix the contiguous `bytes` array with a `varu64` which contains the
   final array length

To decode `coords`, perform the same process in reverse:

1. Decode the beginning of the bytes array as a `varu64` which will show the
   length in bytes for this set of coordinates, advancing by the number of bytes
  used by the `varu64` length field
2. Continue to decode the next bytes as a `varu64`, appending the result to the
   discovered coordinates
3. Continue through the bytes until the initial length value has been reached

### Message types

Yggdrasil implements the following top-level message types:

| Code | Name                  | Scope  | Defined in |
|:-----|:----------------------|:-------|:-----------|
| 0    | Traffic               | Global | YS001      |
| 1    | Protocol message      | Global | YS001      |
| 2    | Link protocol message | Link   | YS001      |

The following top-level message scopes are defined:

1. **Global**: The node **must** forward this message type onto other nodes as
   specified in the destination coordinates
2. **Link**: The node **must not** forward this message to any other nodes
   outside of the current link

Additional message types may be defined by later extension specifications.

#### Protocol message types

Encapsulated within a **protocol message**, Yggdrasil implements the following
second-level message types:

| Code | Name         | Defined in |
|:-----|:-------------|:-----------|
| 4    | Session ping | YS001      |
| 5    | Session pong | YS001      |
| 6    | DHT request  | YS001      |
| 7    | DHT response | YS001      |

#### Link protocol message types

Encapsulated within a **link protocol message**, Yggdrasil implements the
following second-level message types:

| Code | Name          | Defined in |
|:-----|:--------------|:-----------|
| 3    | Switch update | YS001      |

### Top-level message formats

The following message formats are defined for each message type:

#### Traffic

A traffic message contains a payload which can be sent between any two nodes on
the network, being forwarded by intermediate nodes if necessary in order to
reach its destination.

This message type **must** only be used to send session traffic. This message
type **must not** be used to encapsulate Yggdrasil protocol traffic.

The session handle (field 3) identifies the session to which the traffic
belongs, allowing the remote node to determine which shared session key to use
to decrypt the payload (field 5), which is encrypted using shared session keys
known only to the two endpoints of the session.

The nonce (field 4) contains the one-time value required by `curve25519` to
prevent secret key leakage and to add replay resistance to the payload.

The contents of the payload (field 5) **must not** exceed the maximum supported
MTU size of the session to which the handle (field 3) refers.

| Field | Type     | Description                                | Length   | Encrypted |
|:------|:---------|:-------------------------------------------|:---------|:----------|
| 1     | `varu64` | Message code: **must** have a value of `0` | 1 byte   | No        |
| 2     | `coords` | Target coordinates                         | Variable | No        |
| 3     | `bytes`  | Session handle                             | 8 bytes  | No        |
| 4     | `bytes`  | Encryption nonce                           | 24 bytes | No        |
| 5     | `bytes`  | Encrypted message payload                  | Variable | Yes       |

#### Protocol traffic

A protocol message is a control message which can be sent between any two nodes
on the network, being forwarded by intermediate nodes if necessary in order to
reach its destination.

The nonce (field 5) contains the one-time value required by `curve25519` to
prevent secret key leakage and to add replay resistance to the payload.

The payload (field 6) is encrypted **once** using the destination node's public
`curve25519` keys with the above nonce. Once decrypted, the payload contains one
of the below second-level protocol messages.

| Field | Type     | Description                                | Length   | Encrypted |
|:------|:---------|:-------------------------------------------|:---------|:----------|
| 1     | `varu64` | Message code: **must** have a value of `1` | 1 byte   | No        |
| 2     | `coords` | Target coordinates                         | Variable | No        |
| 3     | `bytes`  | Target public encryption key               | 32 bytes | No        |
| 4     | `bytes`  | Sender public encryption key               | 32 bytes | No        |
| 5     | `bytes`  | Encryption nonce                           | 24 bytes | No        |
| 6     | `bytes`  | Encrypted message payload                  | Variable | Yes       |

#### Link protocol traffic

A link protocol message is a control message which can be sent between two nodes
which are directly peered. It is link-scoped - that is, it should never be
forwarded onto another link or peer.

The payload (field 3) is encrypted **twice**:

1. The inner layer, encrypted using ephemeral `curve25519` keys, ensures that
   the message has not been replayed and for forward secrecy
2. The outer layer, encrypted using the node's permanent `curve25519` keys,
   ensures that the message has come from the node that we expect it to come
   from, and has not been captured and replayed from another network node in an
   attempt to impersonate another node

Once decrypted, the payload contains one of the below second-level link protocol
messages.

The nonce (field 2) contains the one-time value required by `curve25519` to
prevent secret key leakage and to add replay resistance to the payload.

| Field | Type     | Description                                | Length   | Encrypted |
|:------|:---------|:-------------------------------------------|:---------|:----------|
| 1     | `varu64` | Message code: **must** have a value of `2` | 1 byte   | No        |
| 2     | `bytes`  | Encryption nonce                           | 24 bytes | No        |
| 3     | `bytes`  | Encrypted message payload                  | Variable | Yes       |

### Second-level protocol message formats

#### Session ping

A session ping is a message which is used to open, or to update, an open session
between two given network nodes. It **must** be encapsulated within a protocol
message.

A session ping is sent in response to the following conditions:

1. A node wishes to establish a new session with a remote node
2. A session is already established with a remote node and one of the following
   conditions occurs:
   1. The node's coordinates have changed
   2. The local maximum supported session MTU has changed

When establishing a new session, a node **must** generate an ephemeral set of
`curve25519` session keys, which are kept for the lifetime of the session. The
ephemeral session public key is then sent to the remote side and is used to
agree a shared session key, with which all session traffic is encrypted.

| Field | Type     | Description                                | Length        |
|:------|:---------|:-------------------------------------------|:--------------|
| 1     | `varu64` | Message code: **must** have a value of `4` | 1 byte        |
| 2     | `bytes`  | Session handle                             | 8 bytes       |
| 3     | `bytes`  | Sender ephemeral session public key        | 32 bytes      |
| 4     | `vari64` | Timestamp                                  | 1 to 10 bytes |
| 5     | `coords` | Sender coordinates                         | Variable      |
| 6     | `varu64` | Sender maximum supported session MTU       | 1 to 2 bytes  |

#### Session pong

A session pong is a message which is sent in response to a session ping. It
**must** be encapsulated within a protocol message.

If the node wishes to participate in a session with a remote node, the node
**must** respond to the session ping by sending a matching session pong in order
to acknowledge the session.

When responding to a session ping in an attempt to establish a new session, a
node **must** generate an ephemeral set of `curve25519` session keys, which are
kept for the lifetime of the session. The ephemeral session public key is then
sent to the remote side and is used to agree a shared session key, with which
all session traffic is encrypted.

| Field | Type     | Description                                | Length        |
|:------|:---------|:-------------------------------------------|:--------------|
| 1     | `varu64` | Message code: **must** have a value of `5` | 1 byte        |
| 2     | `bytes`  | Session handle                             | 8 bytes       |
| 3     | `bytes`  | Sender ephemeral session public key        | 32 bytes      |
| 4     | `vari64` | Timestamp                                  | 1 to 10 bytes |
| 5     | `coords` | Sender coordinates                         | Variable      |
| 6     | `varu64` | Sender maximum supported session MTU       | 1 to 2 bytes  |

#### DHT request

A DHT request is sent when a node on the network wishes to find the
coordinates of a remote node. It **must** be encapsulated within a protocol
message.

The sending node **must** know at least a partial Node ID to search the DHT for.

| Field | Type     | Description                                 | Length   |
|:------|:---------|:--------------------------------------------|:---------|
| 1     | `varu64` | Message code: **must** have a value of `6`  | 1 byte   |
| 2     | `coords` | Sender coordinates                          | Variable |
| 3     | `bytes`  | Known bytes of target Node ID to search for | Variable |

#### DHT response

A DHT response is sent in response to a DHT request when a node on the DHT has
knowledge that helps us to reach the target node. It **must** be encapsulated
within a protocol message.

A response may contain more than one set of target candidates, therefore the
fields n<sub>1</sub> and n<sub>2</sub> may be repeated multiple times.

| Field         | Type     | Description                                | Length   |
|:--------------|:---------|:-------------------------------------------|:---------|
| 1             | `varu64` | Message code: **must** have a value of `7` | 1 byte   |
| 2             | `coords` | Sender coordinates                         | 8 bytes  |
| 3             | `bytes`  | Known bytes of target Node ID searched for | 64 bytes |
| n<sub>1</sub> | `bytes`  | Target candidate public encryption key     | 32 bytes |
| n<sub>2</sub> | `coords` | Target candidate coordinates               | Variable |

### Second-level link protocol message formats

#### Switch update

A switch update is a message that contains a fully signed path containing all
hops from the root node down to a given node. Each hop also contains the port
number that the update was sent to at each node, which allows a node to
construct its own coordinates.

The root node sends a switch update message to all directly connected peers,
containing a timestamp and its own signing key and exactly one signature.

An update message that has been relayed by any non-root nodes will contain
information about more than one hop, therefore the fields n<sub>1</sub>,
n<sub>2</sub> and n<sub>3</sub> will be repeated for each hop.

When a node receives a switch update message, the node **must** append its own
update and then relay to all directly connected peers.

Since the port number must be specified in the update (which **must** be set to
the port number that the update will be sent to), this means that the node
**must** sign the update once for each peer, resulting in each peer receiving a
unique switch update. 

This process continues until the switch update has flooded the entire network
and all nodes have received a switch update that contains `n` number of
signatures, where `n` is the number of hops from the node to the root.

| Field         | Type     | Description                                | Length        |
|:--------------|:---------|:-------------------------------------------|:--------------|
| 1             | `varu64` | Message code: **must** have a value of `3` | 1 byte        |
| 2             | `bytes`  | Root node public signing key               | 32 bytes      |
| 3             | `vari64` | Timestamp                                  | 1 to 10 bytes |
| n<sub>1</sub> | `varu64` | Update port number                         | 1 to 10 bytes |
| n<sub>2</sub> | `bytes`  | Update node public signing key             | 32 bytes      |
| n<sub>3</sub> | `bytes`  | Update signature                           | 64 bytes      |

---

## Peerings

Peerings are connections between the switches of any two given Yggdrasil nodes.

### Stream semantics

All peering connections rely strictly on stream semantics in order to prevent
routing loops. At present, TCP peering connections satisfy all requirements
outlined below:

1. **Guaranteed delivery of packets**: All packets that traverse a specific
peering **must** be retransmitted in the event that a packet (or any fragments
of the packet) are lost, mangled or otherwise damaged in transit.

2. **Ordered delivery of packets**: All packets that traverse a peering, whether
they are protocol-level messages or network forwarded traffic **must** be
delivered in-order. This is necessary so that routing loops do not occur as a
result of protocol messages.

3. **Fragmented delivery of packets**: The connection **must** be able to
fragment a larger packet into smaller packets before being sent to a peered
node. This is required as Yggdrasil may be forwarding packets that are larger
than a given link's maximum transmission unit (MTU) size.

### Multiple peerings

An Yggdrasil node **should** accommodate for the fact that multiple peerings may
exist between a given pair of nodes. They may not necessarily take the same
physical path or exhibit the same connection characteristics such as bandwidth
or latency. They may exist over different network interfaces, address families,
etc.

Note that link aggregation is not a specified behaviour, therefore each peering
**must** be evaluated individually when making a forwarding decision.

---

## Spanning Tree

The spanning tree is one of the major underlying components of the Yggdrasil
design. All nodes on the network participate in the maintenance of the spanning
tree.

### Root selection

The root node is typically the node on the network that has the "strongest" Tree
ID - that is, with the most leading 1 bits set. However, each and every node on
the network is expected to independently make a decision about which node is
acting as the root node.

Since all locator coordinates are relative to the selected root node, it is
therefore critically important that every node on the network **must** follow
the same rules when making this decision so that all nodes eventually agree on
the same root.

The network relies on this convergence to function. Without it, locators will
not necessarily make sense between nodes and traffic may not be delivered.

### Root timestamp updates

It is the responsibility of the root node to advertise a timestamp update
message to all directly connected peers. The update **must** be
cryptographically signed using the `ed25519` keypair.

These updates are then, in turn, flooded to all connected peers at each hop so
that every node on the network receives the timestamp update.

##### Update interval

The root node **should** send this update every 30 seconds, but **must** send
this update at least once a minute.

##### Cool-off period

Nodes on the network **should** ignore any timestamp update that takes place
within 15 seconds of receiving the last timestamp update. This is known as the
"cool-off period".

The cool-off period is important to reduce bandwidth usage and to prevent the
network from being flooded by root timestamp updates too often. During the
cool-off period, root timestamp updates **should not** be relayed to direct
peers.

##### Blacklisting

Each node on the network maintains its own root node blacklist. If the root node
fails to send valid timestamp update messages within the required timeframe,
other nodes on the network **must** blacklist the root node.

Root timestamp updates from blacklisted nodes **must not** be retransmitted to
other directly connected peers.

The node **must** remain blacklisted until one of the two conditions occurs:

1. A new announcement is received from the root node that is valid
2. A new announcement is received from another node that is a better candidate
   to be root, e.g. by possessing a stronger Tree ID

### Parent selection

Each node on the network selects one of its peers to be the "parent" upon which
its own coordinates will be derived. That is, the node's coordinates **must** be
prefixed with the coordinates of the chosen parent node.

Since the node will receive root timestamp updates from all peers, the node
**should** select the node that relays the root updates quickest as the parent.
Doing so will help to minimise the latency of the path from the node to the root
node.

When evaluating whether a root timestamp is eligible for parent selection:

1. All signatures from the peer up to the root node **must** be valid
2. The path from the peer up to the root node **must not** contain any loops,
   e.g. the same signing keys may not appear twice within the same timestamp
   update message

In the event that the peering to the selected "parent" node goes down, the node
**must** select a new parent and update its own coordinates appropriately. In
the event that no peerings are connected, the node becomes isolated from the
rest of the network and **may** become the root node of this new isolated
network.

### Coordinates

Coordinates are an ephemeral identifier which represent the switch's location on
the network graph, relative to the root node. They are not a fixed value and can
change at any time depending on the node's own peers, or changing connectivity
on the path from the node up to the root node.

Coordinates are an array of unsigned 64-bit integers. The root node is noted as
an empty array, e.g. `[]`. All other nodes are noted as an array where each
element represents a switch port ID on the path from the root of the network
down to a specific node, e.g. `[3 6 1 24]`.

---

## Distributed Hash Table (DHT)

Yggdrasil uses a DHT to discover the coordinates of a node on the network where
either a partial or complete Node ID are already known. All nodes in the network
**must** participate in the DHT, responding to searches for other network nodes.

### Neighbours

The DHT used by Yggdrasil is derived from Chord, where each node in the network
**must** maintain a list of DHT neighbours including:

1. The direct predecessor of the node in the DHT
2. The direct successor of the node in the DHT
3. All directly connected peers

Nodes are identified in the DHT by their Node ID.

In addition, nodes **should** maintain a Chord "finger table" of approximately
`O(logn)` other nodes in the ring, at points on the ring which ensure that
searches should probabilistically require only `O(logn)` iterations of DHT
requests/responses. This **may** be approximated by including a node's immediate
successor in the ring, and any other successors which are encountered and are
strictly closer via the tree metric than any successors already tracked, and
likewise in the direction of the predecessor.

### Bootstrap

When a node starts and one or more peering connections are established, the DHT
should bootstrap.

### Requests

At any point, a node may send a DHT request for a given Node ID. The DHT request
**must** contain the following:

1. The current coordinates of the requesting node
2. The known bits of the target Node ID (where all unknown bits **must** be set
   to `0`)

The DHT request is then sent to the locally-known/cached DHT neighbour that has
the closest Node ID to the target Node ID in the request. Note that a request is
not guaranteed to yield a response, therefore the node **should** time out
waiting for a response after a given timeframe.

### Responses

Having received a DHT request, a node **must** return a DHT response
containing:

1. The coordinates and public encryption key of the responding node
2. The target Node ID, exactly as it was sent in the original DHT request
3. Zero or more sets of candidate public encryption keys and coordinates,
   referring to both the closest and the furthest away nodes that the node knows
   about to the target Node ID

In this context, "closest" is defined as the successor of the target Node ID
(the node which appears at or immediately after the target Node ID in the ring),
and "furthest" is defined to be the predecessor (the node which appears immediately
before the target in the ring).

This definition of "closest" is required to ensure that a partial Node ID with
unknown bits set to 0 is owned by a node with the full Node ID, which would not
be the case if the keyspace distance metric was reversed. Nodes **may** search
with the keyspace distance metric reversed, but **must** set unknown bits of the
Node ID to `1` in such a case.

### Searches

To look up a specific node on the network, a search is started by sending a DHT
request targeting a specific Node ID.

The requesting node **must** keep a record of each Node ID that is being
searched for, until a time that either the search has completed, either by
contacting a node with a key which matches the known bits of the target Node ID
or by running out of nodes to check.

The node will then:

1. Receive a DHT response containing information about zero or more closer nodes
   to the target Node ID
2. Receive no response at all, at which point the search should time out

Once the requesting node receives the DHT response from the responding node, the
requesting node **must** verify that the Node ID in the DHT response matches a
search that we initiated. If it does not match an existing search, the response
**must** be dropped and ignored.

If the search is valid, the node **must** then check if the Node ID generated
from the sender's public encryption key in the protocol message headers matches
the searched target Node ID (after applying a bitmask to account for any
unknown bits of the Node ID).

If it does, the search is considered to be completed as the response has come
directly from the node that we were searching for. From this point, the node
**should not** send any additional search requests related to this search.

If not, then the node which handled the request **must** be added to a list of
visited nodes, so that it can be ignored if included in any future responses.
Any information about closer nodes, included in the response, **must** be
checked to confirm that they are closer to the target Node ID than the request
handler. Any nodes which satisfy this condition and do not appear on the list of
nodes already visited in this search **should** be added to a list of known
nodes to contact in later search iterations.

After any response from a node which is not the target of a search, or after a
timeout in the event of no response, the node **must** select the node from the
list of unchecked destinations which is closest to the target Node ID, send a
request to that node, and remove the node from the list. This process **must**
repeat iteratively until either the a node matching the known bits of the target
Node ID is contacted, or until the list of nodes to visit is exhausted and a
timeout passes (to give the last node a chance to respond).

When a search is initialised, the list of visited nodes **must** be empty, and
the list of nodes to visit in later search iterations **must** be filled by the
nodes closest to the destination which are already present in the searching
node's DHT.

### Caching

In a Chord-like DHT, a node's predecessor or successor on the DHT may not be
close to the node in metric space. As searches take place, a node may discover
other nodes which are closer in metric space than the node's predecessor or
successor.

In order to help to speed up lookups, an Yggdrasil node **may** cache these
closer nodes as additional DHT neighbours to use during searches. This allows us
the node to maintain a small database of DHT neighbours that are relatively
close to the node on the tree.

---

## Forwarding

All nodes in an Yggdrasil network have the capacity to forward traffic on behalf
of other network nodes. This happens if:

1. The node has peering connections to more than one node
2. The current node is on an optimal path between two other network nodes which
   are exchanging traffic

### Next-hop selection

When a message is received for forwarding, the node **must** make a decision as
to which node to forward onto next.

The node **must** evaluate each direct peering and determine which peerings will
take the message closer to the target destination coordinates. This is done by
calculating the metric distance between the coordinates of each peer from the
target coordinates and verifying if they are closer to the destination than the
nodes own coordinates are.

In order to prevent routing loops, a strict distance rule applies. A node **must
not** forward traffic to any other node which is not closer to the destination
coordinates in metric space.

Where feasible, a node **should** forward traffic to the node which is closest
to the destination. A node **may** wish to select a node which is closer but not
necessarily closest in order to avoid poor-quality or congested links.

In the event that multiple peered nodes are the same distance to the destination
node, a tie-break may be required - see the "Distance tie-break" section below.

Traffic destined for the current node will fail the distance rule, therefore in
this scenario, the node **should** attempt to decrypt the message using its own
keys and handle it as if it were destined for that node. Note that a node **must
not** attempt to handle any traffic as if it is destined locally if the distance
rule has not failed and a peer can take the traffic closer to its destination.

If the above distance rule fails and decryption also fails, then it is safe to
assert that the packet was not destined for that node and therefore **must**
drop the packet.

### Distance metric

To calculate the distance in hops between two nodes:

1. Find the node which shares the most common leading coordinate elements with
   both node `A` and node `B`
2. Calculate the distance from node `A` to node `L` by subtracting the length
   of node `L`'s coordinates from the length of node `A`'s coordinates
3. Calculate the distance from node `B` to node `L` by subtracting the length
   of node `L`'s coordinates from the length of node `B`'s coordinates
4. Add the two distances together to find the total distance

For example, in the case where node `A` has coordinates `[1 4 2 6 4 2]` and
node `B` has coordinates `[1 4 2 9 6]`, the common ancestor is the node `L` at
coordinates `[1 4 2]`.

The length of `L`'s coordinates is 3, the length of `A`'s coordinates is 6 and
the length of `B`'s coordinates is 5. By subtracting 3 from 6, and then
subtracting 3 from 5, and adding together both results, the total distance
calculated between node `A` and node `B` is 5 hops.

### Distance tie-break

In some cases, multiple peers may result in the same metric distance for a given
set of target coordinates. Therefore, the node **should** implement a tie-break
algorithm in order to determine which of the "best" peers should be selected to
receive the forwarded traffic.

Although a tie-break algorithm is deliberately not defined here, an
implementation **should** consider one or more of the following factors when
deterministically selecting the best peer to forward to:

1. The uptime of each peering
2. The relative latency of each peering, based on which peerings deliver switch
   update messages from the root node first
3. The average utilisation of each peering

Alternatively, a non-deterministic approach **may** be used, e.g. by selecting
from the candidate peers randomly, although doing so may have unintended
negative side-effects on performance.

---

## Sessions

Sessions are a cryptographic agreement between two nodes to allow the exchange
of end-to-end encrypted traffic.

### Initiating a session

In order to open a session with a remote node, a node **must** know the
destination coordinates in order to send a session ping. If the destination
coordinates are not already known, the node **should** start a DHT search and
wait for the response first.

Once the coordinates are known, the node **must** generate and store the
following:

1. A set of ephemeral `curve25519` encryption keys, which will be used to
   generate the shared `curve25519` session key
2. An 8-byte session handler, which **must** be locally unique, to identify the
   session

Note that a node **should not** reuse `curve25519` encryption keys across
sessions, nor should the permanent node encryption keys be used for session
traffic.

The node then **must** send a session ping to the remote side containing:

1. The locally-generated session handle
2. The locally-generated public ephemeral encryption key
3. A timestamp showing the time that the session ping was sent
4. The coordinates of the current node
5. The maximum supported session MTU size in bytes

### Responding to a session

At any time, a node **may** receive a session ping from another node on the
network. The session ping **may** belong to an existing session, or it **may**
be from a node from which no session is currently open.

If the session ping contains an unknown session handle, the node **must** treat
the session ping as an incoming request to open a new session.

However, it is important to note that a node is not obliged to respond to an
initial session ping if it does not wish to open a session with that node and
therefore **may not** send a session pong in response. An example of this may
include whitelisting sessions based on the public encryption key in the protocol
header.

If the session handle is known and belongs to an existing open session, the node
**should** examine the session ping to see if the maximum supported session MTU
has changed and update the local session record if so. In response to a session
ping for a known session, a node **must** respond with a session pong.

Before sending a session pong in response to a session ping, the node **must**
generate and store the following:

1. A set of ephemeral `curve25519` encryption keys, which will be used to
   generate the shared `curve25519` session key
2. An 8-byte session handler, which uniquely identifies the session locally

Importantly, the session handles generated on each node **may not** be the
same, however, both nodes are required to store the handle chosen by the remote
side. Once these elements are generated, the session pong **must** contain:

1. The locally-generated session handle
2. The locally-generated public ephemeral encryption key
3. A timestamp showing the time that the session pong was sent
4. The coordinates of the current node
5. The maximum supported session MTU size in bytes

### Session establishment

A session is established once the initiating node has received a session pong,
and the responding node has sent a session pong. At this point, both nodes are
aware of each other's public ephemeral encryption keys, allowing each node to
compute the shared ephemeral session key, and the session handle on each side.

For all sessions, a node **must** store:

1. The shared ephemeral session key, as computed from both the local and the
   received public ephemeral encryption keys in the received session ping/pong
2. The local session handle, as generated locally
3. The remote session handle, as provided in the received session ping/pong
4. The local maximum supported MTU size of the session, as configured locally
5. The remote maximum supported MTU size of the session, as provided in the
   received session ping/pong

A node **must not** consider a session to be established if it has chosen to not
respond to a session ping, or if a session pong has not been received in
response to a session ping.

The shared `curve25519` session key is computed from both the local and the
remote public ephemeral encryption keys. Assuming that the key exchange from the
session pings/pongs has taken place successfully, the local and the remote node
should arrive at the same computed shared key, which both nodes **must** keep
secret.

### No response

If no session pong is received from the remote side, the node that sent the
original session ping **may** resend the original session ping at an interval
that **should not** be more frequent than once per second, until it becomes
obvious that the remote node is not responding either deliberately, or because
it is offline.

### Traffic exchange

Once a session is open, a node can begin sending traffic over the session. To do
so, the traffic should be encrypted using the ephemeral shared session key and
encapsulated in a traffic packet, before being sent to the target coordinates
with the correct remote session handle.

A node **must not** send session traffic to a remote node until a session has
been established, as it is not possible to know which session handle to use
until a session has been agreed.

A node **may** wish to buffer traffic during session setup so that it can be
sent to the remote node once the session is established.

A node can also receive session traffic from a remote node. The session handle
in the traffic headers is used to determine which shared ephemeral session key
should be used to decrypt the payload.

In the event that a message is received with a handle that is unknown to the
node, the node **must** drop the message.

### Session MTU

During the exchange of session pings and session pongs to set up a session, both
nodes send their maximum supported MTU size for the session to the other node.
Once a round-trip including both a session ping and a session pong have taken
place, both nodes will know what the maximum supported value is for both nodes.

The agreed maximum supported MTU of the session **must** equal the lowest of the
two exchanged values.

The contents of an individual traffic message **must not** exceed this agreed
size. If a node receives a traffic message that exceeds the maximum agreed size
for the session, the node **should** drop the message.

### Session Expiry

Sessions do not have a specific lifetime.

A node **may** consider that a session has expired after a period of inactivity,
e.g. from having received no traffic nor session pings from the remote side
after a defined interval, at which point the node **should** stop accepting
future traffic for that session.

### Session Closure

When closing a session, a node **should** remove the association between the
session handles and the ephemeral shared session key.

The node **must not** send any traffic for a closed session and **must** drop
any received traffic that no longer matches an active session.
