# Convene transport protocol, version 1

**Status: draft for review, 2026-11-14** (threads, receipts, withdrawal and the custodial swarm
added over the 2026-11-12 draft). Nothing here is frozen. The reference implementation is
the convene tree in `kopilati/nord` (`examples/convene`); the behaviour this protocol realises is
specified there as DCI drafts, in the vocabulary of the people who own each layer. Where this
document and those drafts disagree, the drafts are the specification and this document has a
defect to be fixed.

## 1. Scope

This document specifies how sealed records move between the members of a convene group: the
identities and their keys, the records and how they are sealed, the cells they travel in, the
the swarm of peers that carries and holds cells, and the rules a client follows to send,
acknowledge, withdraw, collect, catch up, and change a group's membership. It is written so that a client can be built from it, in any language, that
interoperates with the reference implementation and with every other client built from it.

It does not specify what a message says or how it is shown. That is the rendering protocol
(`rendering.md`). The transport treats message content as opaque bytes.

**Conventions.** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119. Byte strings are
written in hexadecimal. "Base58" is the Bitcoin alphabet with no checksum. "CBOR" is RFC 8949 in
its deterministic encoding (section 4.2.1) with text-string keys; a reader MUST ignore keys it does
not know. Integers are unsigned and big-endian unless stated. `‖` is concatenation. ASCII strings
used in derivations are written in backticks and carry no terminator.

## 2. The shape of the whole thing

- **Nothing in the middle reads anything.** A sender seals a fresh copy of a record for each
  recipient, cuts it into fixed-size cells, and leaves each cell at a pigeonhole only that pair can
  name, with the peers nearest it. A recipient asks its own pigeonholes. A peer sees marks and
  fixed-size cells, nothing else, and nobody in the middle can tell who is talking to whom.
- **Order rides inside the seal.** Every message names every tip its author had seen; two people
  writing at once stand as siblings and the next message closes the fork. What a message
  *answers* is a separate, single, optional pointer that moves nothing in the order and is what a
  reading quotes. Peers keep no order and refuse nothing. Readers assemble the same timeline from
  the same records.
- **Nothing in the swarm acknowledges.** A recipient says what it holds in a receipt, sent on its
  own cadence; a sender keeps every message of its own until the receipts cover it, and may
  withdraw a message nobody has yet. A message anyone has received is part of the conversation.
- **One set of mechanics, at every size.** A cell is placed with the reachable peers nearest its
  pigeonhole, up to k, the sender's own device always among the candidates; a swarm of two works
  exactly as a swarm of thousands, and what differs is only how many peers hold a cell, and so how
  likely one of them is up when the recipient asks. Nothing is ever held back for want of peers:
  fewer peers is worse availability, not different behaviour. How much an observer cannot tell
  is conditional on the swarm's size, and a client states the swarm's population, not a regime.
- **An identity is a keypair.** Its public key, in base58, is its name, its edge, and what an
  invitation carries. There is no directory and no account.
- **One identity, many devices.** A device set is a group whose members are devices and whose
  conversation carries the identity's secrets. Chat groups see the identity only.
- **Membership is governed.** Anyone proposes; the founder or a majority accepts; the proposer
  enacts; anyone rotates. Every epoch has its own key, so access is bounded by epoch.
- **Delivery is at least once and eventual.** Readers deduplicate by id. A custodian keeps a cell
  for days, not forever; members keep everything they received and serve each other what they
  missed; and a sender keeps its own until it is acknowledged.
- **Every device runs the same mechanics; what differs is reachability.** A desktop that stays
  up is a peer in full: reached directly, holding for others, mixing. A phone cannot be reached
  from outside and announces nothing, so only the peers that know it by its card name it as a
  custodian, and its custody is worth what its uptime is. Public drops are peers like any other,
  depended on by nobody in particular.

## 3. Primitives

| purpose | primitive |
|---|---|
| signatures | Ed25519, RFC 8032 |
| key agreement | X25519, RFC 7748 |
| wrapping a key to one recipient | HPKE base mode, DHKEM(X25519, HKDF-SHA256), HKDF-SHA256, ChaCha20-Poly1305 (RFC 9180) |
| sealing under a shared key | XChaCha20-Poly1305, 24-byte random nonce |
| hashing and key derivation | SHA-256, HKDF-SHA256 |
| detection tags | HMAC-SHA256, truncated |
| presence (device admission and eviction) | ECDSA P-256 over SHA-256, key held in platform hardware |
| randomness | the platform CSPRNG |
| encoding | deterministic CBOR; base64url without padding wherever bytes ride inside JSON |

**Domain separation.** Every signature input, HKDF info and HPKE info begins with the ASCII
string `convene/v1/` followed by a purpose word named where it is used. A signature over a record
is Ed25519 over `convene/v1/<record>` ‖ the deterministic CBOR of the record with its signature
fields absent.

## 4. Identifiers

4.1 **Ids.** A message id, revision id, proposal id or reassembly id is 16 bytes from the CSPRNG.
Its textual form is base58. When the protocol says one id *sorts first*, compare the 16 raw bytes
as unsigned big-endian integers; the lower sorts first.

4.2 **Identity id.** The 32-byte Ed25519 public signing key of the identity, in base58. A device
id is the same, from the device's signing key.

4.3 **Lineage id.** The revision id of the lineage's founding revision.

4.4 **Key id.** The first 16 bytes of SHA-256(`convene/v1/key-id` ‖ key material), in base58. A
holder of the material can verify it; nobody else learns anything from it.

4.5 **Edge and pigeonhole.** An edge is where a peer is reached — its identity id at a dead drop,
or a route the peer-to-peer transport knows (9.1); a pigeonhole is the mark a sender places a
cell under and a recipient asks for (9.2). In v1 an identity's edge is its identity id, and a
device's edge inside its device set is its device id. Nothing is ever delivered *to* an edge: a
record for a member is placed under the pair's pigeonholes with the mark's custodians, and the
edge is only where a `hold` or an `ask` travels. Where this document says a record is "dropped at
a member's edge", it means placed for that member this way.

## 5. Identities, devices and cards

5.1 **An identity** holds two keypairs, minted together on its first device: a signing keypair
(Ed25519) and an agreement keypair (X25519). The identity's *secrets* are the two private keys.

5.2 **A device** holds, per device set it belongs to, its own signing and agreement keypairs, and a
*presence* keypair where the platform offers hardware-bound keys. The presence key's grade is
`biometric` when the hardware refuses to sign without a biometric match, `password` when it
refuses to sign without the device password, and `none` when the device has no such hardware. A
device set SHOULD refuse to admit a device of grade `none`.

5.3 **A card** is the only form in which a public key reaches another party. There is no directory
to fetch one from. A card is the CBOR map

```
{ "v": 1,
  "kind": "identity" | "device",
  "id": <32 bytes, the signing public key>,
  "agreement": <32 bytes, the X25519 public key>,
  "presence": <33-byte compressed P-256 point>,      device cards only
  "grade": "biometric" | "password" | "none",         device cards only
  "display": <text>,                                   optional
  "sig": <64 bytes> }
```

with `sig` the Ed25519 signature `convene/v1/card` over the card without `sig`, by the card's own
signing key. A reader MUST verify `sig` before using any field. A card travels in a card link
(10.2), a device-admission code (10.6), or sealed inside a revision (6.2).

5.4 **A card link** is `convene://card?c=<base64url CBOR card>[&r=<rendezvous>...][&p=<route>]`.
It says "this is who I am, and how to find me". The card's identity id is the holder's name in
the swarm; each `r` is a rendezvous hint — a dead drop the holder attends, a local name — that
helps a client find the swarm when it knows no peer yet; `p` is the holder's own route (9.1),
where the device that made the link is reached directly, so the first connection needs no
lookup. Hints are optional, carry no authority, and a client MUST NOT depend on any particular
one: once it is in the swarm it finds custodians by pigeonhole, not by hint. A link carries no secret and may be shown as a QR code,
pasted, or printed. A client MUST verify the card's signature before offering the link's holder as
someone to admit.

5.5 **The ring.** A device keeps, sealed at rest under a key the platform holds (a passkey-derived
key, the Keychain, the Keystore), the identity secrets it holds, its own device secrets, and every
epoch key it was let into. The ring never leaves the device, and nothing in this protocol ever
asks it to.

## 6. Records

Every record is a CBOR map. Records never travel in the clear: a record is sealed (section 7)
under an epoch key, or wrapped (6.5) to one recipient. `(sig)` marks an Ed25519 signature as
defined in section 3.

6.1 **Message**

```
{ "id": <16 bytes>,
  "author": <identity id, 32 bytes>,
  "parents": [ <16 bytes>, ... ],
  "answers": <16 bytes>,                                 optional
  "content": <bytes, as the rendering protocol defines>, absent in a withdrawal
  "sig": (sig, `convene/v1/message`) }
```

`parents` are the ids of **every** tip of the conversation the author's device held when the
message was made — the frontier, so that two messages written at once are siblings and the next
message names both; empty for a conversation's opening message. A client MUST name the whole
frontier it holds, never only the latest line. `answers` is the one message the author meant to
reply to, if any; it MUST NOT be the message's own id, it MUST be a message the author held, it
does not affect order, and a reading quotes it when the answered message is not the line directly
above. A client defaults it to the latest line the person had read when they began to write, and
lets them choose another.

A **correction** is a later message carrying the same `id` with new content and parents that are
the frontier at the time of the correction; readers keep the causally latest sealing of an id and
show it in the place the id first appeared. A **withdrawal** is a correction with no `content`: the
author taking the words back; readers keep the place as a withdrawn entry so that what followed it
still follows something (6.7). A message id is minted once and never re-minted for a different
message.

6.2 **Revision**

```
{ "id": <16 bytes>,
  "lineage": <16 bytes>,
  "parent": <16 bytes>,                                absent on the founding revision
  "key": <key id>,
  "creator": <identity id>,
  "founder": <identity id>,
  "members": [ <member>, ... ],
    "charter": { "purpose": "group" | "device-set",
               "acceptance": "founder-or-majority",
               "presence": true | false,
               "identity": <identity id> },                device sets only
  "warrant": <warrant>,
  "sig": (sig, `convene/v1/revision`, by creator) }
```

A `member` is `{ "id", "agreement", "display" }` in a group and
`{ "id", "agreement", "display", "presence", "grade" }` in a device set: the public half of the
member's card. Every member holds every other member's agreement key from the revision, which is
what lets any member wrap a rotated key to every other, and in a device set every device holds
every sibling's presence key, which is what lets it verify a presence signature. A `warrant` is one of

```
{ "kind": "founding" }
{ "kind": "rotation" }
{ "kind": "enactment", "proposal": <proposal>, "approvals": [ <approval>, ... ] }
```

The founding revision has no `parent`, has `id` equal to its `lineage`, stands on a founding
warrant, and names its creator as founder. Every revision carries the `charter` and the `founder`,
copied from the founding revision down the chain, so that a member holding only a suffix knows
both. Every later revision names its parent and stands on a rotation or an enactment. The
creator is among the members, except for the one revision that enacts the creator's own leaving.
In a device set (`purpose` = `device-set`) the members are device cards, `presence` is true, and
`identity` names the identity whose devices they are; in a group the members are identity cards,
`presence` is false, and there is no `identity`.

6.3 **Proposal**

```
{ "id": <16 bytes>,
  "lineage": <16 bytes>,
  "parent": <16 bytes, the revision it stands on>,
  "proposer": <identity id>,
  "change": "admit" | "evict",
  "subject": <card> for admit, <identity id> for evict,
  "presence": <ECDSA signature>,                        when the charter demands presence
  "sig": (sig, `convene/v1/proposal`) }
```

A proposal whose `change` is `evict` and whose subject is its proposer is a **leaving**. A proposal
to admit MUST carry the subject's full card, because the enactor and every member will wrap keys
to it. `presence` is ECDSA P-256 over SHA-256(`convene/v1/presence` ‖ CBOR of the proposal
without `presence` and `sig`), by the proposer's presence key as recorded in the device set.

6.4 **Approval**

```
{ "proposal": <16 bytes>,
  "approver": <identity id>,
  "presence": <ECDSA signature>,                        when the charter demands presence
  "sig": (sig, `convene/v1/approval`) }
```

6.5 **Wrapped key**

```
{ "v": 1,
  "kind": "key",
  "key": <key id>,
  "recipient": <id>,
  "wrapping": "invitation" | "rotation" | "entrusted",
  "enc": <32 bytes, the HPKE encapsulated key>,
  "ct": <bytes> }
```

`ct` is the HPKE base-mode sealing, to the recipient's agreement key, with info
`convene/v1/wrap` ‖ wrapping ‖ key id and AAD the CBOR of the map without `ct`, of

```
{ "material": <32 bytes, the epoch key>,
  "identity": { "signing": <32 bytes>, "agreement": <32 bytes> } }   entrusted only
```

`invitation` brings a newcomer their first epoch of a lineage; `rotation` hands a continuing
member the next epoch; `entrusted` is used only inside a device set and carries, beside the set's
epoch key, the identity's secrets. A device MUST refuse an `entrusted` wrapping that arrives for a
lineage whose charter purpose is not `device-set`, and MUST refuse an `entrusted` wrapping from
any sender that the set's current revision does not name.

6.6 **Catch-up request**

```
{ "lineage": <16 bytes>,
  "requester": <id>,
  "revisions": [ <16 bytes>, ... ],        every revision id the requester holds
  "messages": [ <16 bytes>, ... ],         the ids of the newest 64 messages it holds
  "sig": (sig, `convene/v1/catchup`) }
```

There is no catch-up response record: the responder drops at the requester's edge, as ordinary
envelopes, every revision and every message it holds for the lineage that the requester did not
list and is entitled to (10.8). Pending proposals and approvals are not served: a proposal is its
proposer's to re-ask on the epoch that stands.

6.7 **Receipt**

```
{ "lineage": <16 bytes>,
  "holder": <identity id, 32 bytes>,
  "frontier": [ <16 bytes>, ... ],
  "sig": (sig, `convene/v1/receipt`) }
```

A holder's word for what it holds: `frontier` is the ids of the newest messages it holds for the
lineage, those no message it holds follows. A receipt **covers** a message when the message is in
the frontier or is an ancestor of one; one receipt therefore acknowledges everything up to a
point. A client sends a receipt on its cadence (9.6), never at the moment of arrival, to every
member of the current epoch, its own inbox among them, and MUST NOT send one whose frontier is
empty. A sender reads receipts against its own messages: a message covered by every other member's
receipt is **received**; by some, **partly**; by none, **queued**.

**Withdrawal and receipts.** A client MUST refuse to withdraw a message any receipt covers. A
receipt that arrives after a withdrawal and covers the message means someone had the words: the
author's device MUST then **restore** it — send the same `id` again with the pre-withdrawal
content and the current frontier as parents — so that everyone, those who got the withdrawal
included, reads what was in fact received. A withdrawn message nobody ever covered leaves no
trace for anyone.

## 7. Sealing and envelopes

7.1 **An envelope** is what one record becomes on its way to one recipient:

```
{ "v": 1,
  "kind": "message" | "revision" | "proposal" | "approval" | "catchup" | "receipt",
  "key": <key id>,
  "nonce": <24 bytes>,
  "ct": <bytes> }
```

`ct` is XChaCha20-Poly1305 under the epoch key named by `key`, with the random `nonce` and AAD
the deterministic CBOR of `{ "v", "kind", "key" }`, over the deterministic CBOR of the record. A
wrapped key (6.5) is its own envelope: it carries `"kind": "key"`, no `nonce`, and its `ct` is the
HPKE ciphertext of 6.5, not a sealing under an epoch key.

7.2 **Which key.** A message is sealed under the current epoch's key of its lineage at the moment
of sending. A revision is sealed under the key of the epoch it begins. A proposal or approval is
sealed under the key of the epoch it stands on. A catch-up request is sealed under the key of the
newest epoch the requester holds for the lineage. A receipt is sealed under the current epoch's
key of its lineage.

7.3 **Per-recipient copies.** The sender seals the same record afresh for each recipient, with a
fresh nonce, so that no two recipients hold the same bytes. The sender's own copy is a recipient's
copy like any other: a sender's own inbox is where its own record lives.

7.4 **What a reader does with an envelope it cannot open** — a key it does not hold, or an
authentication failure — is nothing. It is noise at a pigeonhole, dropped without an alarm.
An envelope that opens but whose record fails its own checks (a bad signature, a revision that
names the reader in no member list it should, a warrant that does not carry) is evidence, and
section 10.5 says what evidence raises.

## 8. Cells

Every envelope travels as one or more cells of exactly **1129 bytes**, so that nothing on the
wire can be sorted by size, and the grouping of a message's cells is visible only to a holder of
the key.

8.1 **Layout**

| offset | size | field |
|---|---|---|
| 0 | 32 | detection tag: salt (16) ‖ HMAC-SHA256(detect key, `convene/v1/tag` ‖ salt)[0..16] |
| 32 | 1 | hops remaining; 0 in v1 |
| 33 | 24 | nonce |
| 57 | 1072 | XChaCha20-Poly1305(cell key, nonce, content, AAD = tag) |

`content` is exactly 1056 bytes:

| offset | size | field |
|---|---|---|
| 0 | 16 | reassembly id, fresh per envelope |
| 16 | 2 | chunk index, from 0 |
| 18 | 2 | chunk count |
| 20 | 2 | length of this chunk's payload, 1 to 1024 |
| 22 | 1024 | the chunk, zero-padded past `length` |
| 1046 | 10 | reserved, zero |

8.2 **Keys.** Cells are keyed **per pair**, not per epoch. For a sender *S* and a recipient *R*,
the pair secret *P* = HKDF-SHA256(ikm = X25519(*S*'s agreement private key, *R*'s agreement
public key), salt = empty, info = `convene/v1/pair` ‖ min(*S* id, *R* id) ‖ max(*S* id, *R* id)) —
the same from either side. Then cell key = HKDF-SHA256(*P*, empty, `convene/v1/cell`), detect key
= HKDF-SHA256(*P*, empty, `convene/v1/detect`), and the pigeonhole of slot *t* and index *i* is
base58 of HKDF-SHA256(*P*, empty, `convene/v1/pigeonhole` ‖ *t* ‖ *i* as 2 bytes)[0..16]. Only the
two of them can recognise, open, or name the places of their cells; another member of the same
epoch cannot. (The 2026-11-12 draft keyed cells per epoch; that is withdrawn, and the drafts'
`SealKey` is now the pair's material.) *Slot* is the UTC hour, written `YYYY-MM-DDTHH`; a sender
the index is the time within the slot at the **step**, 5 minutes unless configured otherwise, so
an hour's slot has indices 0 to 11, read from the sender's clock when a cell is placed and from
the recipient's when it is asked for. The cells a pair exchanges within one step share a mark and
are told apart by the reassembly id inside the seal; nothing is counted, so a device that restarts
leaves its next cell where its clock says and loses nothing. (The 2026-11-13 draft numbered a
sender's cells within a slot from 0 and asked a fixed breadth of indices; a fifth cell in an hour
was never found, and a restart began the count again under marks already asked past.)

For a wrapped key (6.5), the recipient shares no epoch key with the sender yet. Its cells use the
recipient's **hail keys**: cell key = HKDF-SHA256(ikm = recipient id bytes, empty, `convene/v1/hail-cell`),
detect key = HKDF-SHA256(recipient id bytes, empty, `convene/v1/hail-detect`). The hail keys are
public by construction; what protects a wrapped key is the HPKE inside it. A hail cell's content
MUST be a wrapped-key envelope or the revision that key opens — a stranger cannot name the pair
until it has read who invited it — and nothing else; a recipient MUST drop a hail cell that opens
to anything else, and MUST, once it holds the revision, derive the pair secret with the inviter
and collect under it from then on.

8.3 **Chunking.** The envelope's deterministic CBOR is split into 1024-byte chunks; `count` is the
number of chunks, at most 64 in v1 (an envelope of at most 64 KiB; the rendering protocol bounds
content well under that). The reassembled envelope is the concatenation of the chunks' payloads,
each cut at its `length`.

8.4 **Collection.** On every beat (9.6) a device asks, for every pair secret and every hail key
it holds, the pigeonholes of the current index and the one before it — across the slot boundary
when the hour has just turned — and that is the whole of its asking while it keeps up. A device
that last asked longer ago than that asks, once, every index of every slot since, back to the
custodians' retention (9.2); that is how a device that was away reads what was placed for it,
and a client keeps when it last asked (11) so a relaunch asks for what it missed rather than for
everything or for nothing. For each cell that comes back — copies of one cell from several
custodians among them — it tests the tag against every detect key it holds and unseals a match
with the matching cell key; it groups opened contents by reassembly id, a chunk handed twice
counted once, reassembles each group whose every index from 0 to count−1 is present, and hands
the envelope to section 7. A cell that matches nothing is discarded untried: every mark a device
asks is derived from a key it holds, so a mismatch is a collision or noise. An incomplete group
is discarded after 7 days. A device that restarts re-reads the records it wrote down (11), not
cells; the invitation's revision and wrapped key ride under the hail key (8.2), so the pair secret
a device needs is always learned from a record it can already open.

8.5 **Hops.** Every cell — placed, carried or looped — travels through the tumblers (9.3): a cell
with hops above zero is forwarded, one hop fewer, through a pool of peers that batch and reorder.
The hops byte is outside the AEAD so that a tumbler can spend a hop without the key; a wrong hop
count misroutes a cell and reveals nothing. A client sets hops to the swarm's configured depth,
3 unless the swarm says otherwise; a cell that arrives at a custodian or an edge with hops above
zero is still held.

## 9. The swarm

There is no relay. There is a swarm of peers, each of which may mix cells, hold cells for
pigeonholes it is nearest, and answer who is nearest a mark; a device that cannot stay up is a leaf
that uses the swarm without being part of it. No peer is trusted, no peer acknowledges anything,
and no peer is asked who a pigeonhole belongs to.

9.1 **Peers, routes and discovery.** A peer is named by its **endpoint id** — on a peer in full
the iroh endpoint id, on a leaf an id minted from its identity id — and reached at a **route**:
its identity id at a dead drop, or an iroh endpoint written
`iroh:<endpoint id in z-base-32>?a=<ip:port>,…&r=<relay url>` — the id, the addresses it was last
seen at, and its home relay, so a card that carries the route (5.4) lets the first connection
happen with no lookup. An endpoint id says nothing about an identity: a sub maps to an endpoint
only through a card or a hello, so a table of endpoints can be shared without saying who knows
whom, and a stranger can hold for a stranger. The reference binding is iroh; the drafts name the
seam `Wire`. Between peers in full, frames travel one per unidirectional QUIC stream under the
ALPN `convene/frames/1`; to and from a leaf they travel in a dead drop's `to`/`cell` frames
(9.5). A frame is unit-separated text, and there are eight:

| frame | meaning |
|---|---|
| `hold ␟ mark ␟ tag ␟ hops ␟ content` | hold this cell under the mark |
| `ask ␟ reply ␟ mark…` | hand `reply` a copy of what you hold under each mark |
| `held ␟ mark ␟ tag ␟ hops ␟ content` | a copy of what I hold under the mark |
| `hello ␟ endpoint ␟ subs ␟ route` | this endpoint, the subs it speaks for (to an acquaintance only), a route it is reached at |
| `peers ␟ endpoint ␟ route…` | endpoints I know that a stranger may reach, each with an `iroh:` route; never a sub, never a leaf |
| `via ␟ hops ␟ endpoint ␟ route ␟ inner` | mix `inner` and send it on toward the endpoint at that route (9.3); with no endpoint, on toward the custodians you name for the marks inside — a leaf's way in (9.4) |
| `to ␟ token ␟ inner` | hand `inner` to whoever asked with this token (9.3) |
| `near ␟ route ␟ target` | hand `route` the endpoints you know nearest `target`, as a `peers` (9.1) |

`reply` is `<endpoint>@<route>`, or `<tumbler endpoint>/<token>@<tumbler route>` when the ask
came through a tumbler. A frame carries the route of whoever is to receive it next and of whoever
is to answer it, because a tumbler need not know the peers it passes frames between.

Peers keep a table of the endpoints they know — from cards, from hellos, from tables other peers
shared — with their routes, the subs an acquaintance speaks for, and when each was last heard
from, and answer two questions: *how many peers are reachable* and *which k reachable peers are
nearest this mark*, nearness being the distance between the hashed mark and the hashed endpoint
id; the asker's own endpoint is always a candidate, so the answer is never empty. The drafts name
this seam `Discovery`. A device joins by dialing any peer it knows — a route from a card, a public
drop, a peer on the local network — and greets each peer it knows with a `hello` carrying its
routes, the moment it learns the peer, whenever a route of its own moves, and every five minutes
after; a peer in full binds the same port each time it starts, because a route without a relay
URL is only as durable as its port; a peer answers a hello with
the `peers` it can share and a hello of its own, and greets any endpoint it learns from a table,
so that two acquaintances' tables grow into one swarm. A leaf has no route a stranger could use,
so it is in no shared table; the peers that know it by its card name it. A peer MUST NOT answer a
nearness question with an endpoint it has not heard from within 10 minutes, and SHOULD keep no
more than a few thousand, letting the strangers silent longest go first.

Tables shared on hello reach the acquaintances of acquaintances, which fills a swarm of hundreds.
Beyond that a peer in full **walks**: on each tend it asks the two peers in full nearest its own
endpoint id, with a `near`, whom they know near it and near a mark drawn at random, and learns
from the `peers` they answer with — up to 8 endpoints each, nearest first, the answerer itself
among them when a stranger may reach it. So each table grows toward the neighbourhood its own
id falls in and toward one more neighbourhood a tend, and the k nearest a mark converge across
peers that never shared a table. And every naming of a mark's custodians is a step of a **lookup**
for that mark: when the k nearest a peer knows for the mark differ from what it knew when it last
asked, it asks the two nearest of them, with a `near`, whom they know nearer, and the next naming
reads their answers — the Kademlia step, spread over beats rather than awaited — until the naming
stops changing, after which it asks no more for that mark; while the nearest keep changing it asks
no oftener than every thirty seconds for one mark. A new mark in a settled swarm costs two `near`
and two `peers`; a mark in a swarm of millions converges in as many steps as the distance halves.
A leaf neither walks nor seeks: nobody could answer it, and its carrier names its custodians.

9.2 **Custody.** For each cell, the sender asks for the k peers nearest the cell's pigeonhole and
places the cell at each one it is handed — a `hold` of the mark and the cell, carried through the
tumblers like any cell; its own node holds without a hop. It is handed k when the swarm has k, and
fewer when it has fewer, never none. The sender keeps the cells of a message until receipts cover
it (10.11) or it withdraws the message, places the oldest kept cell again on every beat with
whoever the mark's custodians are now (9.6), and when the slot turns places every kept message
again under the current mark, so a recipient who keeps up finds it where it is asking (8.4). Any
other record is placed once; catch-up (10.8) covers what the custodians no longer hold. A custodian keeps a
cell under its mark for at least 24 hours and SHOULD keep it for 7 days, hands a **copy** to
whoever asks for the mark, and periodically asks who the mark's custodians are now and places the
cell at any of them it has not placed that mark with before — it remembers, per mark, whom it
last placed with, so a swarm that has not changed is handed nothing again, a peer arriving among
the nearest is handed what it is now nearest, and a peer leaving does not take the only copies
with it. A
custodian MAY bound what it holds under one mark — the reference floors keep the newest 64
cells and let the oldest go, which in a step full of loops is the noise — and MAY refuse,
silently, beyond a limit its operator publishes. **k is 5** unless the
swarm's operators publish another value with its derivation.

9.3 **Mixing.** Every peer is a tumbler, and every frame to another peer — a `hold`, an `ask`,
a `held` — travels `via` one: the sender picks a reachable peer at random, which may be the
destination itself, and hands it the frame with one hop to spend. A frame that arrives with hops
above zero is held with others until the tumbler has gathered its batch, then the whole batch
leaves at once, reordered, each frame one hop shorter, each toward a fresh peer; the last hop
delivers to the endpoint the frame names, at the route it carries. **How a tumbler withholds is
its own setting**: a batch leaves at `withhold` frames or after `seconds` with anything in the
pen, whichever comes first, 8 and 5 unless its operator sets otherwise (`swarm.toml` beside the
seat on a Mac). A recipient's ask for a pigeonhole travels the same way, and is answered the
same way: the tumbler keeps the asker's route under a token the asker chose — the same for one
custodian within an hour, so a custodian can tell a repeated ask from a new one without learning
who asks — rewrites the ask's reply to its own endpoint and that token, and hands the custodian's
`held` answers back through `to`. A custodian sees a tumbler asking, never the recipient. A
hello, a table, a `near` and a `to` go straight, not via: they are about routes, not about
anyone's words.

A **leaf** cannot place or ask for itself beyond the peers in full it knows: the custodians a
mark falls to are endpoints it never learns, since it is in no shared table and answers no
nearness question. So a leaf hands each `hold` and each `ask` to a peer in full it knows, as a
`via` with one hop and **no endpoint**, and that peer places the cell with, or splits the ask
among, the custodians *it* names for the marks inside, each on through a tumbler as if its own.
The leaf's reply on an ask is its own endpoint and route at the drop, under the carrier's token,
so the answers come back the way a tumbler's do (`to`), and a custodian sees the carrier's
tumbler asking, never the leaf. A leaf's holds and asks for marks whose custodians it can name
in full — one of its acquaintances, itself — need no carrier.

9.4 **The same mechanics at every size.** There is no small-swarm regime. In a swarm of two the
custodians of every mark are the two of them: the sender places the cell with the recipient and
with itself, and the cell is there for the recipient whenever either of them is up — the
recipient from its own depot the moment the hold reaches it, or from the sender's as soon as both
are up. In a swarm of thousands the custodians are strangers who hold it for days. One rule,
per cell, and the only thing that scales is availability. What does change with size is the
strength of the claims of 12: with few peers the mixing of 9.3 hides little and a custodian can
guess whose mark it holds; the honest claim in a swarm of two is sealed content and reliable
delivery, not unlinkability, and a client states the swarm's population rather than presenting
more. (The 2026-11-13 draft held a cell back when fewer than k peers were reachable and carried
it to the recipient's edge instead; that was two sets of mechanics, and it is withdrawn.)

9.5 **Public drops.** A public drop is a peer anyone may run and nobody need trust: it mixes,
holds and answers like any other, and a card may name it as a rendezvous hint. Nothing in this
protocol depends on any particular drop existing. The 2026-11-12 dead-drop frames (`register`,
`to`/`cell`, `cell`, `relayed`) remain as the way the frames of 9.1 reach a device that cannot
speak the peer-to-peer transport: a `hold`, an `ask`, a `held` or a `hello` for such a device
travels as the `cell` of a `to` frame addressed to its identity id, and the drop queues it until
the device attends. The drop reads nothing and holds nothing under a mark; it is a leaf's way in,
not a custodian.

9.6 **The cadence.** Every device keeps a beat, 3 seconds unless configured otherwise, and on
every beat does exactly two things, whether or not it has anything to say or expect: it sends
**one** cell — the oldest cell of a message it still keeps (9.2), placed again with the mark's
custodians now, else a **loop cell**: a cell sealed under a key only it holds, placed with the
custodians of one of its own pigeonholes, that only it will recognise when it comes back — and it
asks the pigeonholes of 8.4, one `ask` per custodian carrying every mark it has for that
custodian. Receipts (6.7) go out as ordinary cells on the beat, never at the moment of arrival. A
loop that comes back is recognised by its key and is not kept at rest; a cell handed over twice is
kept once. A device MUST NOT send more than one cell per beat, MUST NOT skip the ask on a beat,
and MUST NOT vary what it asks while it keeps up; the beat is what makes a retry, a receipt and a
poll indistinguishable from silence. What it asks is the current index and the one before it, per
key (8.4); a device that was away asks what it missed on its first beat back, which is observable
and is the cost of having been away. Nothing but the beat sends: what arrives between beats is
taken in at once — collected, read, shown — and nothing goes out for it, and a gesture sends only
what it places (a message, a record) and no beat of its own. A floor that beat on every arrival
was a floor beating on every tick under steady traffic, and the traffic it made kept it there.

9.7 **What a peer learns.** As a tumbler: cells in, a shuffled batch out, from and to peers. As a
custodian: that a mark was written, from a tumbler; that a mark was asked for, from a tumbler;
when; and that every cell is 1129 bytes. Never a pair, a lineage, a sender or a recipient, and
never whether a device had anything to say. A peer that lies can delay, lose, duplicate or invent
cells, and a client already lives with all four; a peer that drops what it was given to hold is
why k is more than one and members keep what they received.

## 10. Flows

Every flow below is a client's own doing. No step asks a peer anything but nearness, and no peer
answers anything else.

10.1 **Founding.** A device holding an identity mints a lineage: a fresh epoch key, a founding
revision (6.2) with the identity as sole member, creator and founder, sealed under the new key
and dropped at the identity's own edge; the ring adopts the key. A group of one is a working
group.

10.2 **Meeting someone.** Identities exchange card links (5.4) outside the protocol: shown as a QR
code and scanned, read over a call, pasted. A client MUST verify the card's signature and SHOULD
let the person compare a short fingerprint of the signing key with the card's holder when the link
did not arrive screen to camera. The card is what an admission proposal names. Starting a chat with
someone is one gesture: the client founds the lineage, waits for its own founding to come back,
proposes the admission, and enacts it as soon as it carries — the founder's at once, a member's
when the others have approved. A client MUST NOT ask the person to enact.

10.3 **Proposing, approving, enacting, rotating.** As the drafts specify:

- Any member of the current revision may **propose** an admission (naming the newcomer's card),
  an eviction, or their own leaving. The proposal is sealed under the current epoch key and
  dropped at every member's edge, the proposer's own among them.
- Any member of the revision the proposal stands on, the subject excepted, may **approve** it; the
  approval is sealed and dropped at every member's edge likewise.
- A proposal **carries** when the founder is its proposer or among its approvers, or when the
  proposer together with the distinct approvers drawn from the electorate are more than half of
  the electorate, where the electorate is the members of the revision the proposal stands on
  without the subject. A leaving carries on the proposer's word alone.
- The proposer **enacts** a carried proposal: mints a fresh epoch key and the successor revision
  (members adjusted, warrant = enactment with the proposal and the approvals it holds), seals the
  revision under the new key, wraps the new key to each member of the new epoch under their
  agreement key (`invitation` for the admitted, `rotation` for the rest), and drops the revision
  and the wrapped key at each member's edge. The evicted get nothing. The enactor drops the
  revision at its own edge with a rotation wrapped key of its own — not for its ring, which
  adopts the key it minted, but for the other devices of its set, which read the identity's edge
  as their own and held no key of the epoch (10.6). An enactor who
  is leaving drops the key at nobody's edge for itself and discards the key it minted.
- Any member may **rotate**: a successor with the same members, a fresh key, warrant = rotation,
  delivered as above. Clients SHOULD rotate at a random interval and MUST rotate every lineage an
  identity belongs to when the identity admits a device, so that a rotation carries no information
  about why.
- A proposal stands on one revision. When that revision is no longer current, the proposal is
  void; a client MUST NOT enact it and MUST propose again on the current revision if the change is
  still wanted.

10.4 **Adopting.** A device that collects a wrapped key unwraps it, opens the revision that
arrived beside it with the key, and keeps the key only if the revision names the device's
identity (or, in a device set, the device) among its members. A key whose revision does not name
the recipient MUST NOT be kept, and is evidence (10.5).

10.5 **Assembling the chain, and what is evidence.** From the revisions it has opened for a
lineage, a client assembles the chain eldest first by parent references. Where several held
revisions name one parent, a successor *stands* when its warrant holds against that parent: a
rotation that keeps the parent's members exactly, or an enactment whose proposal carries against
the parent's electorate and the lineage's founder, with signatures that verify against the cards
the parent holds. Of the standing successors, the one that is settled on is the enactment whose
proposal id sorts first, before any rotation, and between rotations the revision id that sorts
first; the rest are void, with everything that succeeds them. A successor that does not stand is
**substitution evidence**: the client MUST refuse the lineage's assembly and raise an alarm that
carries the offending revision, and MUST NOT guess. A key delivered for an epoch that does not
name its recipient is **impossible-delivery evidence**. The eldest revision a client holds is
taken as it stands, because a later joiner holds no parent to judge it against.

A message sealed under an epoch that was later voided in a race is unreadable to clients that
settle on the other successor. The author's client, on seeing its epoch voided, MUST send the
message again under the standing epoch, with the same message id.

10.6 **Admitting a device.** The new device mints its device keypairs and a presence keypair and
shows a device-admission code: a card link (5.4) whose card is a device card, with the rendezvous
hints the device knows. An existing device of the set scans it and proposes the admission, with
`presence` signed by its own presence key; the set's founding device may enact at once on its own
word, any other device needs the majority of the set's devices, the admitted device not counted.
Enacting wraps the set's new epoch key **and the identity's secrets** to the new device
(`entrusted`, 6.5) and drops them at the new device's edge, its device id. The new device adopts,
stores the identity's secrets in its ring, registers the identity's edge, and asks the admitting
device to rotate every lineage the identity belongs to (10.3) — in the reference floors the admitting
device rotates every chat the moment it enacts, and every rotation and enactment leaves the actor
a wrapped key of its own at the identity's edge, which is how a sibling device comes to hold the
epoch. The new device holds no epoch key of any group from before its admission: it reads nothing
from before, by design. The identity's edge holds the identity's whole key history for as long as
custodians keep it, and the new device asks that edge's marks, so the rule is the new device's
to keep, not the step's: of the keys waiting for it there, it adopts only those of revisions no
revision it can read — held already, or waiting beside them — names as its parent, and an epoch
the chain shows superseded at its admission is refused however the step falls. The seat remembers
that it came to the identity by entrustment (`a ␟ sub` in its ring at rest), because the same
rule applied to a member who was merely away across two rotations would refuse an epoch that was
theirs. On a Mac the new device is a seat of its own beside the identity's, over one
endpoint and one depot: it shows its admission code — its device card link, its route on it —
until the entrusted delivery arrives, then opens the identity's seat with the card the set's
founding revision names; the device's seat stays, for the set's later business. A phone admits
from *Me, Add a device*; a Mac from *Your card, Add a device*.

10.7 **Evicting a device.** A presence-signed eviction proposal, carried as in 10.6 with the
evicted device not counted, enacted by its proposer. The remaining devices rotate every lineage
the identity belongs to. A device that cannot produce a presence signature cannot propose or
approve an eviction or an admission, which is the point.

10.8 **Catching up.** A device that may have missed deliveries — it was off longer than custody keeps a cell,
it was just admitted — drops a catch-up request (6.6), sealed under the newest epoch key it holds for the lineage, at the
edge of every current member of the lineage, its own among them. Every holder answers and the
requester's fold discards the duplicates. A device that comes back after a custodian's retention
has passed catches up this way: custody covers days, members cover the rest. A responder MUST verify the request's signature against
a card the lineage's chain holds, and MUST then drop at the requester's edge every revision it
holds of an epoch the requester was a member of that the request did not list, and every message
it holds sealed under such an epoch whose id the request did not list. A responder MUST NOT
serve a requester the chain does not name, and MUST NOT serve epochs the requester was not in.
Within a device set, every device asks the identity's pigeonholes itself; siblings hold the same
keys and the custodians hand each of them a copy, so nothing is forwarded between them. A
responder SHOULD rate-limit catch-up to one response per requester per minute.

10.9 **Leaving.** A proposal to evict oneself, enacted by oneself. The leaver mints the one
revision it is not in, seals it under a key it then discards, delivers to the remaining members,
and keeps every key it held from before: what it could already read, it can still read.

10.10 **Sending, and what the person sees.** A sender composes a message with the whole frontier
as parents (6.1) and the chosen answer, seals a copy per member, cuts each copy into cells, and
places each cell (9.2) and keeps the message until receipts cover it. The person sees one of
three states on their own line: **queued** while no other member's receipt covers it, **partly**
with the names that have, **received** when every other member has; the sender's own receipt
counts for nothing here, since the sender holds its own words from the start. A queued message
needs no words beyond that: it is in custody, and a receipt is what moves it. **Withdraw** is
offered only on a queued message; it sends the withdrawal (6.1) to every member, tells the
transport to let go of every cell of the message still kept, and removes the message from the
sender's reading. A
receipt that later covers it restores it (6.7) and the client says so in one line. A withdrawn
message leaves no trace for the sender.

10.11 **Acknowledging.** On its beat, a device whose frontier in a lineage has moved since the
receipt it last sent there sends a receipt (6.7) naming the frontier to every current member of
the lineage — for every lineage it holds, whether or not the person is looking at it. A device
MUST send a receipt at least once per slot in which it holds something new, MUST NOT send one
more often than its beat allows, and SHOULD NOT send one whose frontier it has already sent: a
receipt a beat fills a pair's mark with nothing new. A sender that reads a receipt covering one of its messages tells its
transport that the copy to that member is delivered, so nothing of it is carried further.

## 11. What a client keeps

- Every envelope it received, for every lineage it is in, keyed by lineage and id, for as long as
  the person keeps the conversation. This is the store that catch-up serves from; a client that
  discards history cannot serve it.
- Its ring (5.5), sealed at rest.
- The assembled chain per lineage, which it may recompute at any time from the revisions.
- Its own messages not yet covered by every member's receipt, with each member's state, and
  the pre-withdrawal content of anything it withdrew, in case a receipt restores it.
- Its keeping: the cells of its messages no receipt covers yet (9.2), each with its pigeonhole
  and the message it belongs to, so a withdrawal can let a message's cells go together, and the
  message itself, so the slot turning can place it again.
- A log of the records it read, in order, the routes it learned, and when it last asked its
  marks — what a relaunch re-reads instead of asking the swarm for everything.
- What it holds for others as a custodian (9.2), under each mark, for the retention.
- The receipts it has opened per lineage, by holder.
- For deduplication: the message ids it has seen per lineage, with the latest sealing of each.
- A key only it holds, for its loop cells (9.6).

## 12. Security and metadata considerations

- **Confidentiality** rests on the epoch keys and on HPKE to the agreement keys. Every change of
  membership rotates; a removed member reads nothing after, a newcomer nothing before. Rotation on
  a schedule bounds what a compromised epoch key exposes.
- **Forward secrecy** is at epoch granularity. An invitation or rotation is wrapped under the
  recipient's long-term agreement key, so a later compromise of that key exposes the epoch keys
  it received. Per-message ratcheting and one-time prekeys are not in v1; the drafts record both
  as later evolutions.
- **Authenticity** rests on the signing keys: records are signed, revisions by their creator,
  proposals and approvals by their makers, and a reader verifies against the cards the chain
  holds. Who a card belongs to in the world is settled when cards are exchanged (10.2), not by
  this protocol.
- **Metadata.** A custodian learns that a mark was written and asked for, by a tumbler, when
  (9.7); a tumbler learns cells in and a batch out. Pigeonholes are pairwise and per cell, so
  nothing a peer sees links two cells, a cell to a pair, or a pair to a time; the beat (9.6) makes
  a device's traffic constant whether or not it has anything to say. A tumbler learns the
  endpoints and routes on either side of it, and a token; a custodian learns a tumbler's route and
  a token, never who asked. **How much of this holds is conditional on the swarm's size**, though
  the mechanics are not: in a swarm of two every mark's custodians are the two of them, the
  tumbler is the other party, and a custodian can guess whose mark it holds; a client states the
  population rather than claiming unlinkability it does not have. A shared table names endpoints
  and routes, never a sub or a leaf; who speaks for an endpoint is told to acquaintances only.
- **Replay.** Message ids deduplicate; approvals bind to a proposal id; proposals bind to a parent
  revision and are void once it moves; a `register` carries a time and a signature.
- **Downgrade.** `v` is inside every envelope's AAD and every card's signature; a reader MUST
  refuse an envelope whose `v` it does not implement.
- **A stolen device** can read what its ring holds. It cannot admit or evict a device without a
  presence signature, and its own eviction by a sibling, followed by rotation, ends its reading
  from that epoch on. An identity whose every device is lost has no recovery in v1.
- **A hostile peer** can delay, lose, duplicate or invent cells, drop what it holds, and answer
  nearness falsely. k custodians, re-placement among them, the sender's own keeping until a
  receipt, and members' catch-up are the four answers; none of them trusts the peer.
- **Receipts and withdrawal.** A receipt is signed and sealed, so nobody can forge "received" for
  a message; a withdrawal is a signed correction, so nobody can withdraw another's words; and a
  message anyone received cannot be withdrawn, so the record a recipient holds is never silently
  rewritten against them.

## 13. Versioning

Every card, envelope and cell content carries `v`. A reader ignores map keys it does not know and
refuses a version it does not implement. New record kinds are added as new `kind` values; a v1
reader drops an envelope of unknown kind silently. The cell layout is fixed for v1; the reserved
bytes are zero and a reader ignores them.

## 14. Not yet settled

These are decided in direction and not in detail, and a client built today should expect them:

- the numbers, with their derivations: k, the tumbler threshold, the beat, the step, the
  retention — each stated here with a default and owed a published formula over the swarm's
  measured size and availability, and a way for a client to learn the swarm's published values;
- the lookup's first step in a swarm of millions: a mark's first naming is from a table that
  does not yet hold its custodians, so the first hold of a new mark lands near rather than at
  them until the lookup converges (9.1); whether a sender should await the convergence before
  the first placement, or place again as the naming moves (the beat does), is owed a measured
  answer;
- more than one hop, and a tumbler that reaches a destination it cannot route to: today a frame
  whose last tumbler reaches neither the route it carries nor the endpoint is lost, and the
  sender's beat places it again by another tumbler;
- the one-cell-per-beat claim in the floors: a post places at once, and the slot turning places
  every kept message again;
- a device set carrying the sender's keeping, so a second device of the same person can deliver
  what the first was carrying;
- one-time prekeys for forward-secret invitations, and per-message ratcheting;
- re-keying an identity after suspected compromise, and recovery when every device is lost;
- a custodian's and a tumbler's limits and how a client learns them;
- test vectors for every record, envelope and cell, to be published with the reference
  implementation.
