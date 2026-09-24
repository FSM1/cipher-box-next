# ADR 0023 — The invite link is the primary sharing path, and conversion runs by itself

- **Status:** Proposed
- **Date:** 2026-09-25
- **Supersedes:** the "owner converts" part of
  [Design: sharing and grant delivery architecture (#25)](https://github.com/FSM1/cipher-box-next/issues/25)
  D6, and the invite-record store of
  [ADR 0006](https://github.com/FSM1/cipher-box-next/blob/main/decisions/0006-owner-local-sealed-store.md)
- **Relates to:**
  [wayfinder: sharing as a link-first flow with no approve step (#1945)](https://github.com/FSM1/cipher-box/issues/1945),
  [sharing: conversion on the owner tick on any owner device (#1950)](https://github.com/FSM1/cipher-box/issues/1950),
  [sharing: can any owner device recover a pending link from the published record (#1946)](https://github.com/FSM1/cipher-box/issues/1946),
  [Design: API residual role and surface (#34)](https://github.com/FSM1/cipher-box-next/issues/34)
  D6 (no people directory, unchanged), ADRs 0024 to 0027, the `blueprint/engine.md` "Grants and
  ledger" and "Mailbox logic" sections, the `blueprint/api.md` "Mailbox" section, the
  `blueprint/web-client.md` "Composition (apps/web)" section, and the `CONTEXT.md` "Grant-set
  commitment", "Grant ledger" and "Mailbox" terms

## Context

Today the owner converts a claim by a press on "convert claims" in the share dialog
(`apps/web/src/components/sharing/InviteLinkPanel.tsx:44-58`, `crates/engine/src/facade.rs:9190-9450`).
The tick leaves claim items alone (`crates/engine/src/grants/inbox.rs:11-15`). Only the minting
browser can convert, because the link records and the spent claim ids are device-local
(`crates/engine/src/grants/invite.rs:316-331`, `:512-530`). The record cannot say which rows are
links, so the code refuses to decide from it (`invite.rs:760-766`).

The research ticket
[sharing: can any owner device recover a pending link from the published record (#1946)](https://github.com/FSM1/cipher-box/issues/1946)
found gaps in the record. A link row has the shape of a personal row. The deadline `expiresAt` is
outside both owner signatures, so a committed write grantee can change it
(`crates/core/src/seal/write_body.rs:74-88`, `:202-255`). The spent claim ids and the link-to-grant
pairs are owner-local. It recommended a new vault-synced owner structure. The charting of the map
then decided that there is no approve step.

File and line citations come from the research reports (`main` at `d30509e48` or `ee147eac9`).

## Decision

**D1 — There is no approve step.** Every claim that passes D3 converts. The owner removes a
person later (ADR 0025). The contact code stays as an advanced path. There is no people directory,
and the decision #34 D6 stays.

**D2 — Conversion state lives in the record, under owner signatures.**

- The grant-set commitment entry gains a kind, `personal` or `link`, and a deadline for a `link`
  entry. The commitment signature covers both. The codec refuses a deadline on a `personal`
  entry, on decode and on encode.
- The ledger row `expiresAt` retires. The commitment entry holds the one deadline.
- A member's ledger row gains a via-link reference: the tag of the link row that admitted the
  member. The field joins the row signature preimage. A write wave re-mints every row at a new
  tag, so it re-maps the reference in the same pass.
- No vault-synced owner structure is introduced. The invite-record store kind retires.
- The invite secret is never stored on an owner device. A link shows only at its creation.

**D3 — The conversion rule.** An owner device converts a claim only when each check passes:

1. The claim opens, and its sender signature verifies, as today.
2. The claim names a scope root that this owner holds, and the record passes the adoption gate.
3. The sender is the owner-attested `recipientIdentityPk` of a ledger row whose commitment entry
   has the kind `link`.
4. The deadline of that entry is later than the injected `now`.
5. The claimant contact code passes its binding verify at import.

When a row for the claimant identity already exists in the scope, conversion is a no-op. Otherwise
the device mints a personal row at the link's permission with the via-link reference, re-signs
the commitment, publishes the root, and posts the share pointer.

**D4 — The trigger.** Any owner device converts on every tick, with one root publish per folder
per tick, and when the owner opens the share dialog of a folder.

**D5 — Ack first, convert second.** The mailbox delete answers whether this call removed the item.
The engine converts only on "removed". Today the delete always answers success
(`apps/api/src/mailbox/services/mailbox.service.ts:237-252`), and the engine accepts any 2xx
(`crates/engine/src/api/client.rs:647-655`). The order is: ack, write a durable op-queue entry that
holds the claim, convert, publish. For claim items this replaces "ack after the fact is durable".

**D6 — Retry on both sides.** After a failed publish, the owner engine retries the op-queue entry
on later ticks. The claimant engine posts its claim again when no personal blob lands inside a
window, and reads as a link holder meanwhile (ADR 0024).

**D7 — The owner signal.** The people list updates from the record on every owner device. The
device that converted shows one transient notice, "X joined <folder>". Nothing persists.

**D8 — A revoked person may join again through another live link.** It is a fresh admission. The
contact book does not sync, and conversion and revoke do not read it.

### Why this reverses the recommendation of the research ticket

| Gap that the research found                  | What closes it                                         |
| -------------------------------------------- | ------------------------------------------------------ |
| A link row looks like a personal row         | The owner-signed entry kind (D2), checked by D3 item 3 |
| The deadline is outside the owner signatures | The owner-signed deadline on the entry (D2)            |
| Spent claim ids are owner-local              | The ack lock (D5) and the no-op on a known identity    |
| Link-to-grant pairs are owner-local          | The via-link reference, and ADR 0024 D3                |
| A later revoke needs the claimant key        | The owner-signed ledger row (ADR 0025 D3)              |

The re-share attack of `invite.rs:763-770` fails at D3 item 3, because a personal entry never has
the kind `link`. The only loss is that a second owner device cannot show a link again.

## Alternatives considered

**(a) An owner-only vault-synced structure, as the research ticket recommended.** Rejected. The
table above closes every gap with owner-signed fields. The structure adds a second authority beside the
record, and it needs a carrier, merge rules and a monotone generation.

**(b) An approve step.** Rejected at the charting. A per-link "ask me first" switch, off by
default, stays on the map as an item that is not yet specified.

**(c) Conversion on the minting device only.** Rejected. The owner may not open it again.

**(d) Sync the invite secret, so that any owner device shows a link again.** Rejected. A stored
secret is a bearer capability.

## Consequences

1. **`crates/core` wire formats change.** The commitment entry gains the kind and the deadline. The
   ledger row gains the via-link reference inside the row signature and loses `expiresAt`. The
   KATs are pinned again. Each decode refusal has a release-active encode check (AGENTS.md rule 8).
2. **A public observer sees which commitment entries are links, and when each ends.** A mask
   under `pointerReadKey` needs a new edge in the frozen KDF catalog, and the entry already shows
   its permission in the clear.
3. **The blueprint changes.** `blueprint/engine.md` "Grants and ledger": the "Invites" bullet
   states D2 to D8, and "Authority" states that the owner engine converts with no click.
   `blueprint/engine.md` "Mailbox logic": "Lifecycle" states D5 for claim items.
   `blueprint/api.md` "Mailbox": the ack answers whether the call removed the item.
   `blueprint/web-client.md` "Composition (apps/web)": the "convert claims" control goes, and the
   notice of D7 comes.
4. **`CONTEXT.md` changes.** New terms: **claim** (the grantee side of a link: the sealed,
   ephemeral-key-signed mailbox request that carries the claimant contact code), **conversion**
   (the owner side: the owner engine mints the claimant a personal grant, on the tick, with no
   click), and **invite link**. "Grant-set commitment" gains the kind and the deadline, "Grant
   ledger" gains the via-link reference, and "Mailbox" gains the ack answer. **Cut** stays the
   engine word for a revoke, and **adoption** stays with the record gate only.
5. **ADR 0006 keeps its structure** for received shares and the contact book. **The pending
   conversion is a new op record,** and ADR 0020 applies to it.

## Residuals

**E1 — A lying API can report "removed" to two owner devices.** Both mint the same row, because the
tag derives from the owner secret and the claimant key. No extra grant results.

**E2 — The API can replay an acked claim through a live link.** It re-admits a person who once
claimed through that link and who can rejoin through it anyway. The owner can cut the link.

The blueprint and glossary are maintained in the `FSM1/cipher-box` repository. The `blueprint/`
copies in this repository are the as-charted archive and are not edited by this ADR.
