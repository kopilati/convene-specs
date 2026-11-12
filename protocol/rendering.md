# Convene rendering protocol, version 1

**Status: draft for review, 2026-11-12.** Nothing here is frozen. The presentation this protocol
fixes is specified as DCI drafts in the convene tree in `kopilati/nord` (`examples/convene`,
the `surface` crate). Where this document and those drafts disagree, the drafts are the
specification and this document has a defect to be fixed.

## 1. Scope

This document specifies the **content** of a message — the bytes the transport protocol carries
as `content` and treats as opaque — and the rules every client follows to **present** a
conversation, so that two people on two clients read the same thing. It covers what a client
encodes when a person writes, what a client shows when a message arrives, and what a client does
with content it does not understand.

It says nothing about keys, cells, relays or membership. That is the transport protocol
(`transport.md`).

In version 1 a message carries text, written in Markdown. Images and video are reserved: the
envelope is shaped so that they can be added without a v1 client breaking, and section 7 says how
they will arrive.

**Conventions.** As in the transport protocol: MUST, SHOULD, MAY per RFC 2119; CBOR is RFC 8949
deterministic encoding with text-string keys; unknown keys are ignored on read.

## 2. The content envelope

`content` is the deterministic CBOR of

```
{ "v": 1,
  "at": <unix milliseconds>,            optional: when the author composed it, the author's claim
  "re": <16 bytes>,                     optional: the message this one answers, for presentation
  "parts": [ <part>, ... ] }
```

A `part` is a CBOR map with a `"type"` key holding a media type, and the keys that type defines.
A message MUST carry at least one part. Version 1 defines one part type:

```
{ "type": "text/markdown", "text": <text> }
```

Reserved types, defined in section 7 and not to be sent by a v1 client: `image/*`, `video/*`, and
`application/x-convene-retraction`.

**Rules for every client, every version.**

- A client MUST render every part type it implements and MUST show a placeholder for a part type
  it does not, saying that there is something it cannot show. It MUST NOT drop a message because
  one of its parts is unknown, and MUST NOT hide that a part exists.
- `at` is the author's claim and is shown as such. The transport carries no clock; a client MUST
  NOT reorder messages by `at`. Order comes from the transport's timeline fold (section 5).
- `re` is presentation only: a client MAY show the answered message's first line above this one.
  Causal order still comes from the message's parents, which the transport owns.
- Content size: the CBOR of `content` MUST NOT exceed 32,768 bytes in v1, and the `text` of a
  text part MUST NOT exceed 16,384 bytes of UTF-8. A client's composer refuses longer text; a
  client that receives longer content shows it truncated with a visible mark.

## 3. Text

3.1 **Encoding.** UTF-8, lines separated by LF. A sender SHOULD normalise to NFC; a receiver MUST
accept any form. A client MUST strip nothing on receipt except what section 3.2 says is not
interpreted; the text a person wrote is the text the others read.

3.2 **Dialect.** CommonMark 0.31, as rendered in a chat line, with these constraints, every one of
them mandatory:

- **Raw HTML is never interpreted.** HTML blocks and inline HTML are rendered as the literal
  characters the author typed.
- **Nothing in content causes a fetch.** A client MUST NOT retrieve any resource that content
  refers to, by URL or otherwise, without an explicit act by the person reading. An image written
  as `![alt](url)` is rendered as its alt text with the URL beside it, as a link, not as an image.
  Link previews, favicons, embeds and tracking pixels are all fetches and are all forbidden.
- **Links** are rendered, and opened only by the person's explicit act, with the full destination
  visible before it opens. Schemes `https`, `http`, `mailto` and `convene` are links; any other
  scheme is rendered as text. Bare URLs SHOULD be recognised as links (GFM autolinks).
- **A single newline is a line break.** Chat is written line by line; a soft break renders as a
  hard break. A blank line separates paragraphs.
- **Allowed and rendered:** paragraphs, emphasis, strong emphasis, strikethrough (`~~`, as in
  GFM), inline code, fenced and indented code blocks, block quotes, ordered and unordered lists,
  thematic breaks, headings.
- **Headings** are rendered inside the line, as emphasis at most two steps above body text, never
  as document headings; a chat line is not a page.
- **Code** is rendered in a monospace face. A client MAY highlight syntax and MUST NOT execute,
  evaluate or interpret anything in it.
- **Tables, footnotes, task lists, math and other extensions** are not in v1 and are rendered as
  CommonMark renders their source text.
- **Mentions** of members are not in v1. A later version may define a `convene:` link form for
  them; until then an identity id in text is text.

3.3 **Plain-text projection.** For notifications, accessibility, previews and search, a client
derives plain text from a text part by rendering the Markdown and taking its text content, with
one space where a line break was. A client MUST NOT put any of a message's content into a
notification unless the content was unsealed on that device; a notification raised before
unsealing says only that something arrived.

## 4. What a line shows

A client presents a lineage's conversation as lines, in the order the transport's timeline fold
gives (section 5), each line one of two shapes:

- **A spoken line:** who said it, what they said, and whose voice it is. The speaker is the
  author's shown name as the reader's client knows it from the members named across the
  revisions the reader has opened — a speaker from a closed epoch keeps the name the reader saw
  them under, and a speaker no held revision names is shown by their identity id, honestly, never
  by an invented name. The voice is *mine* when the author is the reader's own identity, on any of
  the reader's devices, and *theirs* otherwise; a client lays the two voices out so that they can
  be told apart at a glance.
- **An awaited line:** a message the reader holds names a parent that has not arrived. The hole
  is shown, in its place, as waiting, and it is not an accusation: the parent may be travelling,
  withheld, or from before the reader's time. A client MUST NOT collapse an awaited line into its
  neighbours.

A **correction** — a later sealing of a message id already shown — replaces the line's content
in the place the id first appeared, marked as edited; the earlier content is not shown and MAY be
kept for the person to inspect. A client MUST NOT show a correction as a new line.

The **chat itself is named by its people**: the shown names of the current revision's members
other than the reader, or the reader's own shown name when they stand alone. Nothing on the wire
carries a group name in v1. A client MAY let the person give a chat a local name, and MUST make
clear it is local.

## 5. Order

Order is the transport's: a message stands after every message it names as a parent; messages
that are concurrent — neither names the other, directly or through others — stand in the order
their ids sort (transport 4.1), so every member reads the same sequence; a correction stands
where the id first appeared; an id is shown once however many copies arrived. A client MUST NOT
sort by `at`, by arrival, or by any clock of its own. A client MAY show `at` beside a line.

## 6. Reading state and receipts

Version 1 carries no read receipts and no delivery receipts, and a client MUST NOT infer either
from the transport. A client MAY keep local reading state (what the person has seen on this
device) and MAY share it between the devices of one identity through the device set; it MUST NOT
send it to other members. A later version may define receipts as a content part; a group's
members will opt in.

## 7. Reserved: images, video, retraction

These part types are defined in direction so that a v1 client shows the right placeholder and a
later client sends the right thing. They are not to be sent by a v1 client.

7.1 **Attachments.** An image or video does not ride inside `content`. It is a **blob**: bytes
sealed under the lineage's epoch key at the time of sending, chunked into cells and delivered to
each recipient exactly as an envelope is (transport, section 8), identified by the SHA-256 of
the sealed bytes. A part refers to its blob:

```
{ "type": "image/jpeg" | "image/png" | "image/webp" | "video/mp4",
  "blob": <32 bytes, SHA-256 of the sealed blob>,
  "size": <bytes>,
  "width": <pixels>, "height": <pixels>,
  "duration": <milliseconds>,                     video only
  "alt": <text>,                                   optional
  "thumb": <bytes, a JPEG of at most 16,384 bytes> optional, carried inline }
```

A client renders the thumbnail at once, if present, and the blob when it has reassembled it; a
blob that has not arrived is a placeholder with the part's size and type, and the person may ask
for it, which is a catch-up for the blob from any member who holds it. A client MUST NOT play
video automatically and MUST NOT fetch anything from outside the protocol to render a part.
Limits will be set with the v2 transport; the working figures are 8 MiB for an image and 64 MiB
for a video.

7.2 **Retraction.** `{ "type": "application/x-convene-retraction" }` as the sole part of a
correction (a later sealing of the same message id) withdraws the message. A client shows a
withdrawn line in the message's place, saying that it was withdrawn, and shows no earlier
content. A retraction that arrives before the message it withdraws stands as a withdrawn line
from the start.

## 8. Versioning

`v` names the content envelope's version. A client that receives a `v` it does not implement
shows a placeholder saying the message needs a newer client, and keeps the message. New part
types are added as new `type` values under the rule in section 2; new fields are added as new
keys, ignored by older clients. The text dialect of section 3 is versioned with the envelope.
