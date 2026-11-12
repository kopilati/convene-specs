# convene — specifications

Convene is a sealed group chat with no server that reads anything: every message is sealed on the
member's device and delivered to each recipient it is for, membership is sealed too, and the
relays in between see fixed-size cells and nothing else.

Two kinds of specification live here, and they answer different questions.

**The protocol** — what any client, in any language, must do to interoperate:

- [`protocol/transport.md`](protocol/transport.md) — identities, keys, records, sealing, cells,
  relays, and the flows: founding, proposing and enacting a membership change, rotating, adopting,
  admitting a device, catching up.
- [`protocol/rendering.md`](protocol/rendering.md) — the content of a message and how a
  conversation is presented: the Markdown dialect, what a line shows, order, and the reserved
  shapes for images, video and retraction.

**The behaviour** — the DCI drafts the reference implementation is compiled from, published as
linked pages at [nordhavn.info/specs/convene](https://nordhavn.info/specs/convene/). They are
written in the vocabulary of the people who own each layer: the comms domain (`core`), the
transport domain (`mix`), the presentation domain (`surface`). Where a protocol document and a
draft disagree, the draft is the specification and the protocol document has a defect.

Both protocol documents are **version 1 drafts, under review**. Their status lines say what is
settled and what is not.

The reference implementation is `examples/convene` in `kopilati/nord`.
