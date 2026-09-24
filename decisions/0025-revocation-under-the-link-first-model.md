# ADR 0025 — Revocation under the link-first model

- **Status:** Proposed
- **Date:** 2026-09-25
- **Relates to:**
  [wayfinder: sharing as a link-first flow with no approve step (#1945)](https://github.com/FSM1/cipher-box/issues/1945),
  [sharing: what revocation means under the link-first model (#1949)](https://github.com/FSM1/cipher-box/issues/1949),
  [sharing: prototype the share dialog with the link as the main path (#1948)](https://github.com/FSM1/cipher-box/issues/1948)
  (the revoke confirmation and the "can" control),
  [Design: sharing and grant delivery architecture (#25)](https://github.com/FSM1/cipher-box-next/issues/25)
  D4 (the immediate cut), ADR 0023 (the entry kind, the deadline and the via-link reference),
  ADR 0024 (link holders), the `blueprint/engine.md` "Grants and ledger" ("Revocation is
  discovered, not delivered") and "Rotation primitives" ("Triggers") sections, the
  `blueprint/web-client.md` "Composition (apps/web)" section, and the `CONTEXT.md` "Cut epoch",
  "Revocation floor" and "Write-scope cut" terms

## Context

A person revoke runs `revoke_grant`, then `cut_recipient`, and finds the recipient only in the
device-local contact book (`crates/engine/src/facade.rs:8079-8093`, `:8223-8233`). So a person
that device A converted cannot be revoked from device B. `cut_and_rotate` steps the cut epoch and
republishes the scope root and every descendant scope root under a fresh seed
(`facade.rs:8249-8313`, `crates/engine/src/rotation/trigger.rs:795`). "Revoke link" cuts the link
row only, and the grants that came from its claims stay (`facade.rs:9018-9055`). Nothing cuts a
link at its deadline. The deadline sits in the sealed ledger, where a read holder cannot see it.

ADR 0024 adds link holders, who read through a key that every holder of one link shares. This ADR
states what each revoke removes and what the removed side sees. The engine word for a revoke
stays **cut**. File and line citations come from the research reports (`main` at `d30509e48`).

## Decision

**D1 — "Revoke link" cuts the link row.** Every link holder of that link loses access at once,
because the rotation leaves no blob at the link tag. A member who came through the link keeps
access by default. The confirmation carries one checkbox, "also remove the N people who joined
through this link", for a leaked link. The engine finds those members by their via-link
reference (ADR 0023 D2).

**D2 — The owner's tick cuts expired links.** On each tick the owner engine cuts every link whose
commitment deadline (ADR 0023 D2) is not later than the injected `now`. The cut is a rotation,
the same as a manual revoke. An honest grantee engine reads the same deadline and stops reading
through the link at it, so the UI says "expired" before the owner's tick runs.

**D3 — Revoke runs on any owner device.** The engine reads the person's encryption key from the
owner-signed ledger row in the folder record, after the row signature verifies
(`crates/core/src/seal/write_body.rs:202-255`). It does not read the contact book. The contact
book becomes a label cache (ADR 0027).

**D4 — One revoke is one cut.** Every row that one revoke removes leaves in one cut set, with one
cut-epoch step and one rotation. The set is a person and the link that admitted the person
(ADR 0024 D3), or a link and the members that the checkbox of D1 names.

**D5 — The removed side sees one of three messages.**

| Message               | When the engine shows it                                           |
| --------------------- | ------------------------------------------------------------------ |
| The owner removed you | A member finds no blob at the personal tag                         |
| The link expired      | The deadline of the link entry it last verified is not after `now` |
| The link was revoked  | A link holder finds no blob at the link tag before the deadline    |

The engine knows whether it read as a link holder or as a member from its bookmark
(ADR 0024 D2).

**D6 — The owner can change a member's permission.** The "can" column of the people table is a
control with view and edit.

- An upgrade mints write material for that person. When the folder is not a write scope yet, a
  write-scope cut runs first.
- A downgrade is a write revoke: a write rotation renames the subtree. The person keeps read, with
  a read row minted at the new name.

**D7 — A link's permission is fixed at creation.** To change it, the owner revokes the link and
creates a new one. The link chips carry no permission control.

## Alternatives considered

**(a) A link revoke also removes its members by default.** Rejected. Most link revokes end a link
that did its job. The checkbox covers a leaked link.

**(b) Keep the deadline in the sealed ledger.** Rejected. A read holder cannot see it, and a
committed write grantee can change it.

**(c) Sync the contact book so that any device finds the recipient.** Rejected. The owner-signed
row already carries the key, and ADR 0023 D8 keeps the contact book local.

**(d) One cut per removed row.** Rejected. It costs one rotation of the scope and every
descendant scope root for each row, and it steps the cut epoch more than once for one owner act.

## Consequences

1. **Each expired link costs one rotation.** D2 rotates the scope and every descendant scope root,
   as a manual revoke does.
2. **`blueprint/engine.md` changes.** In "Grants and ledger", "Revocation is discovered, not
   delivered" states D1, D3, D4 and D5, and D6 and D7 join it. In "Rotation primitives",
   "Triggers" gains the expired-link cut of D2.
3. **`blueprint/web-client.md` changes.** "Composition (apps/web)" gains the "can" control, the
   inline revoke confirmation that names the link and the other people who keep access, the
   checkbox of D1, and the three messages of D5.
4. **`CONTEXT.md` changes.** "Cut epoch" gains "one step per owner revoke, whatever the number of
   rows". "Write-scope cut" gains the upgrade of D6. "Revocation floor" does not change.
5. **The contact book stops being a trust input.** Revoke and conversion read only the record.

## Residuals

**E1 — A hostile link holder can read past the deadline until the owner's tick runs.** Its engine
ignores the deadline. The owner's next online tick cuts the link. An owner who stays offline keeps
the window open.

**E2 — A person who is removed keeps what the person saw.** This is the promise of #25 D4: "they
keep what they saw; they lose everything new, now."

The blueprint and glossary are maintained in the `FSM1/cipher-box` repository. The `blueprint/`
copies in this repository are the as-charted archive and are not edited by this ADR.
