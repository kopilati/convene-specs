# Convene transport protocol, version 1

**Status: draft for review, 2026-11-12.** Nothing here is frozen. The reference implementation is
the convene tree in `kopilati/nord` (`examples/convene`); the behaviour this protocol realises is
specified there as DCI drafts, in the vocabulary of the people who own each layer. Where this
document and those drafts disagree, the drafts are the specification and this document has a
defect to be fixed.

## 1. Scope

This document specifies how sealed records move between the members of a convene group: the
identities and their keys, the records and how they are sealed, the cells they travel in, the
relays that carry cells, and the rules a client follows to send, collect, catch up, and change a
group's membership. It is written so that a client can be built from it, in any language, that
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
  recipient and drops it at the recipient's edge. A recipient collects only at its own edge. A
  relay sees edges and fixed-size cells, nothing else.
- **Order rides inside the seal.** Every message names the messages it follows. Relays keep no
  order and refuse nothing. Readers assemble the same timeline from the same records.
- **An identity is a keypair.** Its public key, in base58, is its name, its edge, and what an
  invitation carries. There is no directory and no account.
- **One identity, many devices.** A device set is a group whose members are devices and whose
  conversation carries the identity's secrets. Chat groups see the identity only.
- **Membership is governed.** Anyone proposes; the founder or a majority accepts; the proposer
  enacts; anyone rotates. Every epoch has its own key, so access is bounded by epoch.
- **Delivery is at least once and eventual.** Readers deduplicate by id. A relay keeps a cell for
  days, not forever; members keep everything they received and serve each other what they missed.

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

4.5 **Edge.** The string a recipient registers at a relay and a sender addresses. In v1 an
identity's edge is its identity id, and a device's edge inside its device set is its device id.
This is the recorded metadata concession of v1: a relay learns which edges receive and when. A
later version derives edges pairwise so that they cannot be linked; the relay frames do not change
for that.

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

5.4 **A card link** is `convene://card?c=<base64url CBOR card>&r=<relay url>[&r=...]`. It says
"this is who I am, and where I can be reached". A link carries no secret and may be shown as a QR
code, pasted, or printed. A client MUST verify the card's signature before offering the link's
holder as someone to admit.

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
  "content": <bytes, as the rendering protocol defines>,
  "sig": (sig, `convene/v1/message`) }
```

`parents` are the ids of the messages the author's device had seen as the tips of the conversation
when the message was made; empty for a conversation's opening message. A **correction** is a later
message carrying the same `id` with new content and parents that include the tips at the time of
the correction; readers keep the causally latest sealing of an id and show it in the place the id
first appeared. A message id is minted once and never re-minted for a different message.

6.2 **Revision**

```
{ "id": <16 bytes>,
  "lineage": <16 bytes>,
  "parent": <16 bytes>,                                absent on the founding revision
  "key": <key id>,
  "creator": <identity id>,
  "founder": <identity id>,
  "members": [ <member>, ... ],
  "charter": { "purpose": "group" | "device-set",     founding revision only
               "acceptance": "founder-or-majority",
               "presence": true | false },
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

The founding revision has no `parent`, has `id` equal to its `lineage`, carries the `charter`,
stands on a founding warrant, and names its creator as founder. Every later revision names its
parent, carries the founder copied from its parent, and stands on a rotation or an enactment. The
creator is among the members, except for the one revision that enacts the creator's own leaving.
In a device set (`purpose` = `device-set`) the members are device cards and `presence` is true;
in a group they are identity cards and `presence` is false.

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
envelopes, every record it holds for the lineage that the requester did not list and that the
requester is entitled to (10.8).

## 7. Sealing and envelopes

7.1 **An envelope** is what one record becomes on its way to one recipient:

```
{ "v": 1,
  "kind": "message" | "revision" | "proposal" | "approval" | "catchup",
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
newest epoch the requester holds for the lineage.

7.3 **Per-recipient copies.** The sender seals the same record afresh for each recipient, with a
fresh nonce, so that no two recipients hold the same bytes. The sender's own copy is a recipient's
copy like any other: a sender's own inbox is where its own record lives.

7.4 **What a reader does with an envelope it cannot open** — a key it does not hold, or an
authentication failure — is nothing. It is noise on a public dead drop, dropped without an alarm.
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

8.2 **Keys.** For an envelope sealed under an epoch key *K*: cell key = HKDF-SHA256(ikm = *K*,
salt = empty, info = `convene/v1/cell`), detect key = HKDF-SHA256(*K*, empty, `convene/v1/detect`).
Every holder of the epoch key can therefore recognise and open every member's cells of that
epoch; the per-recipient fan-out buys unlinkability of copies, not secrecy among members, and the
drafts say so.

For a wrapped key (6.5), the recipient shares no epoch key with the sender yet. Its cells use the
recipient's **hail keys**: cell key = HKDF-SHA256(ikm = recipient id bytes, empty, `convene/v1/hail-cell`),
detect key = HKDF-SHA256(recipient id bytes, empty, `convene/v1/hail-detect`). The hail keys are
public by construction; what protects a wrapped key is the HPKE inside it. A hail cell's content
MUST be a wrapped-key envelope and nothing else; a recipient MUST drop a hail cell that opens to
anything else.

8.3 **Chunking.** The envelope's deterministic CBOR is split into 1024-byte chunks; `count` is the
number of chunks, at most 64 in v1 (an envelope of at most 64 KiB; the rendering protocol bounds
content well under that). The reassembled envelope is the concatenation of the chunks' payloads,
each cut at its `length`.

8.4 **Collection.** For each cell at its edge, a device tests the tag against every detect key it
holds — one per epoch key in its ring, plus its hail detect keys — and unseals a match with the
matching cell key. It groups opened contents by reassembly id, reassembles each group whose every
index from 0 to count−1 is present, and hands the envelope to section 7. Cells that match nothing
are discarded untried. An incomplete group waits for a later collection and is discarded after
7 days.

8.5 **Hops.** Reserved for the mixing transport the drafts specify (`mix/`): a cell with hops
above zero is forwarded, one hop fewer, through a pool of tumblers that batch and reorder. The
hops byte is outside the AEAD so that a tumbler can spend a hop without the key; a wrong hop
count misroutes a cell and reveals nothing. A v1
client sets hops to 0 and a v1 relay ignores the field. The layout is fixed now so that v1 cells
and mixed cells are indistinguishable on the wire when the mix arrives.

## 9. The dead-drop relay

A relay is a mailbox per edge. It holds cells for an edge nobody attends, hands them over when
someone does, and forgets them. It reads nothing, orders nothing across edges, acknowledges
nothing, and trusts nobody, including other relays.

9.1 **Transport.** WebSocket (RFC 6455), `wss://` across the public network, `ws://` permitted on
a local network between native clients. Any path. Each WebSocket text frame carries exactly one
JSON object. A cell rides as a base64url string without padding.

9.2 **Frames**

| frame | direction | meaning |
|---|---|---|
| `{"register": <edge>, "at": <unix seconds>, "proof": <b64>}` | client → relay | Attend the edge. `proof` is base64url of the Ed25519 signature over `convene/v1/register` ‖ edge (as its UTF-8 text) ‖ `at` (8 bytes), by the key whose base58 public key is the edge. Everything waiting flushes as separate `cell` frames, in arrival order; later cells for the edge are handed straight over. A later registrant displaces an earlier one. |
| `{"to": <edge>, "cell": <b64>}` | client → relay | Leave a cell for an edge. Handed at once if the edge is attended, otherwise kept. |
| `{"cell": <b64>}` | relay → client | A delivery. A flush and a live hand-over are indistinguishable. |
| `{"relayed": {"to": <edge>, "cell": <b64>}}` | relay → relay | A cell a fellow relay passes on. Kept exactly as a client's would be, and passed no further. |

9.3 **Rules.** A relay MUST ignore, silently, a `register` whose proof does not verify or whose
`at` differs from the relay's clock by more than 300 seconds; it MUST ignore any frame it cannot
parse, any frame with an empty edge or cell, and any cell that is not exactly 1129 bytes once
decoded. A relay MUST NOT acknowledge anything, and MUST NOT reorder the cells it holds for one
edge. Registering is idempotent: registering again from the same socket flushes whatever has arrived
since and keeps the attendance.

9.4 **Retention and limits.** A relay MUST keep an unclaimed cell for at least 24 hours and SHOULD
keep it for 7 days. A relay MAY refuse, silently, cells beyond a per-edge or total limit it
publishes to its operator; it SHOULD hold at least 4096 cells per edge. A relay restart MAY start
empty; members' own holdings and catch-up are what recover loss.

9.5 **Fellowship.** A relay MAY keep company with other relays. A cell nobody attended at this
relay is passed on to each fellow as a `relayed` frame, a copy and never a move; a cell that
arrived as `relayed` is passed no further. Whom a relay keeps company with is its operator's
configuration. Delivery across a fellowship is therefore at least once, and a client may be
handed the same cell by two relays.

9.6 **Client behaviour.** A client holds a set of relay URLs: its own configuration, plus the
relays named in the card links of everyone it has admitted or been admitted by. It shuffles the
set and dials until one answers, registers each edge it attends, and **re-registers every 60
seconds** while connected: registering is idempotent, and it is how a device displaced by a
sibling regains attendance once the sibling has gone. A client reconnects with backoff when the
socket goes, keeps an outbox for cells it could not send, and sends them on reconnection. A
client SHOULD register at more than one relay when the relays are not known to keep company,
because fellowship is one hop and a cell left at a relay that keeps company with nobody waits
there alone.

9.7 **What the relay learns.** Which edges are attended, from which addresses, when; which edges
receive cells, how many, when. Cells are fixed-size, so nothing about content, grouping, sender,
or lineage. A relay that lies can delay, lose, duplicate or invent cells, and a client already
lives with all four.

## 10. Flows

Every flow below is a client's own doing. No step asks a relay anything, and no relay answers.

10.1 **Founding.** A device holding an identity mints a lineage: a fresh epoch key, a founding
revision (6.2) with the identity as sole member, creator and founder, sealed under the new key
and dropped at the identity's own edge; the ring adopts the key. A group of one is a working
group.

10.2 **Meeting someone.** Identities exchange card links (5.4) outside the protocol: shown as a QR
code and scanned, read over a call, pasted. A client MUST verify the card's signature and SHOULD
let the person compare a short fingerprint of the signing key with the card's holder when the link
did not arrive screen to camera. The card is what an admission proposal names.

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
  revision at its own edge without a wrapped key; its ring already holds the key. An enactor who
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
shows a device-admission code: a card link (5.4) whose card is a device card, with the relays the
device will attend. An existing device of the set scans it and proposes the admission, with
`presence` signed by its own presence key; the set's founding device may enact at once on its own
word, any other device needs the majority of the set's devices, the admitted device not counted.
Enacting wraps the set's new epoch key **and the identity's secrets** to the new device
(`entrusted`, 6.5) and drops them at the new device's edge, its device id. The new device adopts,
stores the identity's secrets in its ring, registers the identity's edge, and asks the admitting
device to rotate every lineage the identity belongs to (10.3). The new device holds no epoch key
of any group from before its admission: it reads nothing from before, by design.

10.7 **Evicting a device.** A presence-signed eviction proposal, carried as in 10.6 with the
evicted device not counted, enacted by its proposer. The remaining devices rotate every lineage
the identity belongs to. A device that cannot produce a presence signature cannot propose or
approve an eviction or an admission, which is the point.

10.8 **Catching up.** A device that may have missed deliveries — it was off, its relay expired a
mailbox, it was just admitted — drops a catch-up request (6.6) at the edge of any member of the
lineage. A responder MUST verify the request's signature against a card the lineage's chain
holds, and MUST then drop at the requester's edge every revision, proposal and approval it holds
for the lineage that the request did not list, and every message it holds for the lineage sealed
under an epoch the requester was a member of that the request did not list. A responder MUST NOT
serve a requester the chain does not name, and MUST NOT serve epochs the requester was not in.
Within a device set, a device that collects anything at the identity's edge MUST forward the cells
it collected, unchanged, to each sibling device's edge; siblings hold the same keys and open them
as if they had collected them. A responder SHOULD rate-limit catch-up to one response per
requester per minute.

10.9 **Leaving.** A proposal to evict oneself, enacted by oneself. The leaver mints the one
revision it is not in, seals it under a key it then discards, delivers to the remaining members,
and keeps every key it held from before: what it could already read, it can still read.

## 11. What a client keeps

- Every envelope it received, for every lineage it is in, keyed by lineage and id, for as long as
  the person keeps the conversation. This is the store that catch-up serves from; a client that
  discards history cannot serve it.
- Its ring (5.5), sealed at rest.
- The assembled chain per lineage, which it may recompute at any time from the revisions.
- Its outbox: records sealed but not yet handed to a relay, retried until they are.
- For deduplication: the message ids it has seen per lineage, with the latest sealing of each.

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
- **Metadata.** A relay learns edges, timing and counts (9.7). In v1 an edge is an identity id,
  so a relay can tell which identities receive and when, and can link an identity's traffic across
  time. Fixed-size cells hide sizes and grouping. Pairwise edges and the mixing transport are the
  two later evolutions that close this; both keep the frames of section 9.
- **Replay.** Message ids deduplicate; approvals bind to a proposal id; proposals bind to a parent
  revision and are void once it moves; a `register` carries a time and a signature.
- **Downgrade.** `v` is inside every envelope's AAD and every card's signature; a reader MUST
  refuse an envelope whose `v` it does not implement.
- **A stolen device** can read what its ring holds. It cannot admit or evict a device without a
  presence signature, and its own eviction by a sibling, followed by rotation, ends its reading
  from that epoch on. An identity whose every device is lost has no recovery in v1.
- **A hostile relay** can delay, lose, duplicate or invent cells and can refuse an edge. Clients
  register at more than one relay and members serve each other by catch-up.

## 13. Versioning

Every card, envelope and cell content carries `v`. A reader ignores map keys it does not know and
refuses a version it does not implement. New record kinds are added as new `kind` values; a v1
reader drops an envelope of unknown kind silently. The cell layout is fixed for v1; the reserved
bytes are zero and a reader ignores them.

## 14. Not yet settled

These are decided in direction and not in detail, and a client built today should expect them:

- the mixing transport (`mix/` in the drafts): tumblers, hops above zero, cover traffic;
- pairwise-derived edges, so an identity's traffic cannot be linked at a relay;
- one-time prekeys for forward-secret invitations, and per-message ratcheting;
- re-keying an identity after suspected compromise, and recovery when every device is lost;
- the exact form of a relay operator's limits and how a client learns them;
- test vectors for every record, envelope and cell, to be published with the reference
  implementation.
