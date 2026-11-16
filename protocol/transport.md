# Convene transport protocol, version 1

**Status: draft for review, 2026-11-16 — rebased on Aegis.** Nothing here is frozen. Until this
draft, this document specified everything from keys to cells to the swarm. What convene shares
with any other product built the same way — identities, lineages, sealing, cells, the swarm and
connections — is now the **Aegis protocol**, drafted in `kopilati/cairn` at `protocol/aegis.md`
and due a repository of its own, and this document specifies only what is convene's: the records
of a conversation, the two charters convene founds lineages under, and the rules a client
follows to send, acknowledge, withdraw and catch up. A reference written "Aegis 9.2" is to that
document's section; a bare "6.1" is to this one. Sections here keep the numbers they had, so a
section that moved is a pointer at the number it used to have.

The behaviour this protocol realises is specified as DCI drafts in `kopilati/convene`, in the
vocabulary of the people who own each layer. Where this document and those drafts disagree, the
drafts are the specification and this document has a defect to be fixed. **Those drafts and the
reference floors still realise the 2026-11-14 draft**: they speak `convene/v1/` on the wire for
everything, and a client built from this draft does not interoperate with them. Section 16 says
what changed.

## 1. Scope

This document specifies how sealed records move between the members of a convene group, on top
of Aegis: the records and which key seals each, and the rules a client follows to send,
acknowledge, withdraw, collect, catch up, and change a group's membership. It is written so
that a client can be built from it and from the Aegis protocol, in any language, that
interoperates with every other client built from them.

It does not specify what a message says or how it is shown. That is the rendering protocol
(`rendering.md`). The transport treats message content as opaque bytes.

**Conventions.** As Aegis 1: MUST, SHOULD and MAY per RFC 2119; CBOR is RFC 8949 in its
deterministic encoding with text-string keys; a reader MUST ignore keys it does not know.

## 2. The shape of the whole thing

- **Convene is a product on Aegis.** Nothing in the middle reads anything, an identity is a
  keypair, one identity has many devices, and every device runs the same mechanics: all of that
  is Aegis's (Aegis 2), and this document does not restate it.
- **A group is a lineage under convene's charter** (6.2): anyone proposes; the founder or a
  majority accepts; the proposer enacts; anyone rotates; every change of membership opens a new
  epoch, so a newcomer reads nothing from before and a removed member nothing after.
- **A member is an identity, never a persona.** People who meet exchange their own cards
  (10.2); being the same person in every group is what a chat is for.
- **Order rides inside the seal.** Every message names every tip its author had seen; two people
  writing at once stand as siblings and the next message closes the fork. What a message
  *answers* is a separate, single, optional pointer that moves nothing in the order and is what a
  reading quotes. Peers keep no order and refuse nothing. Readers assemble the same timeline from
  the same records.
- **Nothing in the swarm acknowledges.** A recipient says what it holds in a receipt, sent at
  its own cadence; a sender keeps every message of its own until the receipts cover it, and may
  withdraw a message nobody has yet. A message anyone has received is part of the conversation.
- **Delivery is at least once and eventual.** Readers deduplicate by id. A custodian keeps a cell
  for days, not forever; members keep everything they received and serve each other what they
  missed; and a sender keeps its own until it is acknowledged.
- **Chat needs no connection.** Everything here reaches a recipient at its pigeonholes, at its
  cadence (Aegis 13 is for what that is too slow for).

## 3. Primitives

As Aegis 3. **Domain separation.** A signature over one of convene's own records is Ed25519 over
`convene/v1/<record>` ‖ the deterministic CBOR of the record with its signature fields absent:
`convene/v1/message`, `convene/v1/catchup`, `convene/v1/receipt`, `convene/v1/reaction`.
Everything Aegis defines — cards, revisions, proposals, approvals, wrapped keys, cells, pair
secrets, pigeonholes — uses `aegis/v1/`, as Aegis says.

## 4. Identifiers

4.1 **Ids.** A message id is 16 bytes from the CSPRNG.
Its textual form is base58. When the protocol says one id *sorts first*, compare the 16 raw bytes
as unsigned big-endian integers; the lower sorts first.

Identity ids, lineage ids, key ids, edges and pigeonholes are Aegis 4.2 to 4.5.

## 5. Identities, devices and cards

As Aegis 5. A convene member is the identity itself: a client MUST NOT present a persona
(Aegis 5.6) as a member of a group. A card link is Aegis 5.4's, `aegis://card?…`; a client also
reads the 2026-11-14 form `convene://card?…`, which carries the same card.

## 6. Records

Every record is a CBOR map. Records never travel in the clear: a record is sealed (section 7)
under an epoch key, or wrapped (Aegis 6.5) to one recipient. `(sig)` marks an Ed25519 signature
as defined in section 3.

6.1 **Message**

```
{ "id": <16 bytes>,
  "author": <identity id, 32 bytes>,
  "parents": [ <16 bytes>, ... ],
  "answers": <16 bytes>,                                 optional
  "sent": <unix seconds, UTC>,
  "content": <bytes, as the rendering protocol defines>, absent in a withdrawal
  "sig": (sig, `convene/v1/message`) }
```

`sent` is when the author's device made the message, in seconds since the epoch, UTC, by the
author's own clock. It is the author's claim: a client shows it as local time and MUST NOT order
messages by it — order is the parents' (section 5 of the rendering protocol). A correction carries
its own `sent`; a withdrawal carries the message's, since it empties the place and does not move
it. (The rendering protocol's `at` is withdrawn in favour of this field; a reader ignores it.)

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

6.2 **Revision, proposal, approval, wrapped key.** These are Aegis 6.2 to 6.5. Convene founds
lineages under two charters (Aegis 10):

| | `purpose` | `governs` | `writes` | `roll` | `admitting` | `keys` | `presence` |
|---|---|---|---|---|---|---|---|
| a group | `group` | `founder-or-majority` | `members` | `members` | `opens` | `delivered` | false |
| a device set | `device-set` | `founder-or-majority` | `members` | `members` | `opens` | `delivered` | true, with `identity` |

So in a convene lineage every member sees the roll and holds every other member's agreement key,
every change of membership opens a new epoch, and a key reaches a member when the roll changes,
never by asking: there is no origin to ask later, and a member must be able to read offline.

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
point. A client sends a receipt on its cadence (Aegis 9.6), never at the moment of arrival, to every
member of the current epoch, its own inbox among them, and MUST NOT send one whose frontier is
empty. A sender reads receipts against its own messages: a message covered by every other member's
receipt is **received**; by some, **partly**; by none, **queued**.

**Withdrawal and receipts.** A client MUST refuse to withdraw a message any receipt covers. A
receipt that arrives after a withdrawal and covers the message means someone had the words: the
author's device MUST then **restore** it — send the same `id` again with the pre-withdrawal
content and the current frontier as parents — so that everyone, those who got the withdrawal
included, reads what was in fact received. A withdrawn message nobody ever covered leaves no
trace for anyone.

6.8 **Reaction**

```
{ "lineage": <16 bytes>,
  "author": <identity id, 32 bytes>,
  "message": <16 bytes>,
  "mark": <text>,
  "image": <cid>,                                        custom reactions only
  "present": true | false,
  "sent": <unix seconds, UTC>,
  "sig": (sig, `convene/v1/reaction`) }
```

A member's mark on a message: `mark` is an emoji, or a custom reaction's name, in which case
`image` names the picture — the cid of a blob (6.9) shared with the lineage. `present` says
whether the mark stands or this record takes it back. A reaction is a toggle, not a history: of
one author's records with one `(mark, image)` on one message, **the latest `sent` stands**, and
at a tie the record with `present: false` wins, so a reader never shows a mark its author took
back in the same second. A client sends a reaction to every member of the current epoch, its own
inbox among them, sealed under the current epoch's key of the lineage, once — it is a record,
not a message: it is not placed again, and catch-up (10.8) covers what a custodian no
longer holds. A reaction on a message the reader does not hold is kept until the message
arrives. `mark` MUST NOT exceed 64 bytes of UTF-8.

6.9 **Blob**

```
{ "media": "image/png" | "image/webp",
  "bytes": <bytes> }
```

A small picture a lineage shares — a custom reaction's image — named by its content. Its **cid**
is the CIDv1 of the raw codec over the SHA-256 of `bytes`, in lowercase base32 with the multibase
prefix `b` (`bafkrei…`): the name IPFS would give the same bytes, computed by every reader from
the bytes it opens and never trusted from a record. `bytes` MUST NOT exceed 32,768 and the
picture MUST NOT exceed 128 by 128 pixels; a reader shows a blob that breaks either rule as the
reaction's name, not as a picture. A blob is sealed under the current epoch's key of the lineage
(7.2) and sent to every member of the epoch once, on the first reaction that names it; a client
that holds a reaction naming a cid it does not hold shows the reaction's name in its place and
asks for the blob as it asks for anything it missed (10.8). No client fetches a blob from
anywhere but its own inbox: not from IPFS, not from a server — a group's picture is the group's.

(Aegis 6.9 defines a block, sealed once and held by the swarm for anyone who can name it. A
convene blob is deliberately not that: it is a group's picture, sealed under the group's key and
sent to its members' inboxes, and no client fetches one from anywhere else. Whether larger
attachments become blocks is with the rendering protocol's section 8.)

## 7. Sealing and envelopes

7.1 **An envelope** is Aegis 7.1's. Convene's kinds are `message`, `catchup`, `receipt`,
`reaction` and `blob`, beside Aegis's `revision`, `proposal`, `approval` and `key`.

7.2 **Which key.** A message is sealed under the current epoch's key of its lineage at the
moment of sending. A catch-up request is sealed under the key of the newest epoch the requester
holds for the lineage. A receipt, a reaction and a blob are sealed under the current epoch's key
of their lineage. Revisions, proposals and approvals are as Aegis 7.2 says.

Per-recipient copies and what a reader does with an envelope it cannot open are Aegis 7.3 and
7.4. An envelope that opens but whose record fails its own checks is evidence (Aegis 10.5).

## 8. Cells

As Aegis 8, unchanged in size and layout: 1129 bytes, keyed per pair, a wrapped key under the
recipient's hail keys, an envelope of at most 64 chunks.

## 9. The swarm

As Aegis 9: discovery, custody, the chain, the cadence and the balance are Aegis's, and a
convene client follows them as any client does. What convene adds is what a sender keeps and
when a receipt goes out.

9.2 **What a sender keeps.** A sender keeps the cells of a message until receipts cover it
(10.11) or it withdraws the message. At each emission (Aegis 9.6) the one frame it sends is the
oldest kept cell, placed again with whoever the mark's custodians are now; and when the slot
turns it places every kept message again under the current mark, so a recipient who keeps up
finds it where it is asking (Aegis 8.4). Any other record — a revision, a proposal, an
approval, a receipt, a reaction, a blob, a catch-up request — is placed once; catch-up (10.8)
covers what the custodians no longer hold.

9.3 **Depth.** A convene client's depth (Aegis 9.3) is one unless the person chooses more.

9.6 **Receipts and the cadence.** Receipts (6.7) go out as ordinary cells at an emission, never
at the moment of arrival, so that a retry, a receipt and an asking cannot be told from silence.

## 10. Flows

Founding a group, proposing, approving, enacting, rotating, adopting, assembling the chain,
admitting and evicting a device, and leaving are Aegis 10.1 and 10.3 to 10.9, under the
charters of 6.2. What follows is convene's own.

10.2 **Meeting someone.** Identities exchange card links (Aegis 5.4) outside the protocol: shown as a QR
code and scanned, read over a call, pasted. A client MUST verify the card's signature and SHOULD
let the person compare a short fingerprint of the signing key with the card's holder when the link
did not arrive screen to camera. The card is what an admission proposal names. Starting a chat with
someone is one gesture: the client founds the lineage, waits for its own founding to come back,
proposes the admission, and enacts it as soon as it carries — the founder's at once, a member's
when the others have approved. A client MUST NOT ask the person to enact.

10.5 **A message under a voided epoch.** A message sealed under an epoch that was later voided in a race is unreadable to clients that
settle on the other successor. The author's client, on seeing its epoch voided, MUST send the
message again under the standing epoch, with the same message id.

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

10.10 **Sending, and what the person sees.** A sender composes a message with the whole frontier
as parents (6.1) and the chosen answer, seals a copy per member, cuts each copy into cells, and
places each cell (Aegis 9.2) and keeps the message until receipts cover it. The person sees one of
three states on their own line: **queued** while no other member's receipt covers it, **partly**
with the names that have, **received** when every other member has; the sender's own receipt
counts for nothing here, since the sender holds its own words from the start. A queued message
needs no words beyond that: it is in custody, and a receipt is what moves it. **Withdraw** is
offered only on a queued message; it sends the withdrawal (6.1) to every member, tells the
transport to let go of every cell of the message still kept, and removes the message from the
sender's reading. A
receipt that later covers it restores it (6.7) and the client says so in one line. A withdrawn
message leaves no trace for the sender.

10.11 **Acknowledging.** At an emission, a device whose frontier in a lineage has moved since the
receipt it last sent there sends a receipt (6.7) naming the frontier to every current member of
the lineage — for every lineage it holds, whether or not the person is looking at it. A device
MUST send a receipt at least once per slot in which it holds something new, MUST NOT send one
more often than its rate allows, and SHOULD NOT send one whose frontier it has already sent: a
receipt at every emission fills a pair's mark with nothing new. A sender that reads a receipt covering one of its messages tells its
transport that the copy to that member is delivered, so nothing of it is carried further.

## 11. What a client keeps

Beside what Aegis 11 lists:

- Every envelope it received, for every lineage it is in, keyed by lineage and id, for as long as
  the person keeps the conversation. This is the store that catch-up serves from; a client that
  discards history cannot serve it.
- Its ring (Aegis 5.5), sealed at rest.
- The assembled chain per lineage, which it may recompute at any time from the revisions.
- Its own messages not yet covered by every member's receipt, with each member's state, and
  the pre-withdrawal content of anything it withdrew, in case a receipt restores it.
- Its keeping: the cells of its messages no receipt covers yet (Aegis 9.2), each with its pigeonhole
  and the message it belongs to, so a withdrawal can let a message's cells go together, and the
  message itself, so the slot turning can place it again.
- A log of the records it read, in order, the routes it learned, and when it last asked its
  marks — what a relaunch re-reads instead of asking the swarm for everything.
- What it holds for others as a custodian (Aegis 9.2), under each mark, for the retention.
- The receipts it has opened per lineage, by holder.
- For deduplication: the message ids it has seen per lineage, with the latest sealing of each.
- A key only it holds, for its loop cells (Aegis 9.6).

## 12. Security and metadata considerations

As Aegis 12, for everything Aegis carries. For convene's own records:

- **Authenticity.** Messages, receipts, reactions and catch-up requests are signed by their
  makers, and a reader verifies against the cards the lineage's chain holds. Who a card belongs
  to in the world is settled when cards are exchanged (10.2), not by this protocol.
- **Replay.** Message ids deduplicate; a correction is the causally latest sealing of an id.

- **Receipts and withdrawal.** A receipt is signed and sealed, so nobody can forge "received" for
  a message; a withdrawal is a signed correction, so nobody can withdraw another's words; and a
  message anyone received cannot be withdrawn, so the record a recipient holds is never silently
  rewritten against them.

## 13. Connections

Aegis 13. Chat needs none of it.

## 14. Versioning

Every envelope carries `v` (Aegis 14). New record kinds are added as new `kind` values; a v1
reader drops an envelope of unknown kind silently.

## 15. Not yet settled

- everything Aegis 15 lists, which a convene client inherits;
- the one-cell-per-emission claim in the floors: a post places at once, and the slot turning
  places every kept message again;
- a device set carrying the sender's keeping, so a second device of the same person can deliver
  what the first was carrying;
- whether attachments (rendering 8) are Aegis blocks, and if so under which key a group seals
  them;
- when the drafts and the floors move from the 2026-11-14 wire to this one, and whether a
  client speaks both for a time.

## 16. What changed from the 2026-11-14 draft

Everything in this list is a change on the wire.

- **Domain strings.** Every derivation that Aegis now defines begins `aegis/v1/` in place of
  `convene/v1/`: cards, revisions, proposals, approvals, wrappings, key ids, pair secrets, cell
  and detect keys, pigeonholes, hail keys, reach, connections. Convene's own four records keep
  `convene/v1/`.
- **The chain replaces mixing** (Aegis 9.3). A frame no longer goes `via` one tumbler that picks
  the next: its sender chooses the whole chain, the recipient stands anywhere in it, each peer
  changes the cell with a cipher pass, and the frame travels on to the chain's end. The `via`
  and `to` frames are gone; an ask carries a return path in place of a reply token.
- **The cadence is random at a rate** (Aegis 9.6). The fixed beat of 3 seconds is the mean of a
  random gap. A device may raise its rate for a period.
- **Peers keep a balance** (Aegis 9.8). A neighbour in debt beyond a tolerance pays in work, a
  stamp on the frame; a hello carries a peer's terms.
- **A custodian may always decline** (Aegis 9.2), beyond an allowance or for reasons of its own.
- **The charter is wider** (Aegis 10). `acceptance` is replaced by `governs`, `writes`, `roll`,
  `admitting` and `keys`; convene's two charters are the rows of 6.2 and behave as before.
- **A cell's hops byte is its count** (Aegis 8.5), and is what the peer before owes for it.
- **Dead drops are gone.** A leaf hands its frames to a peer in full it knows, which leads for
  it (Aegis 9.3); the `register`, `to`/`cell` and `relayed` frames of the 2026-11-12 draft are
  withdrawn.
- **Also in Aegis and unused by chat:** personas, postings, notices, chained keys, blocks,
  answering on the way, and the use case named when a connection is opened.
