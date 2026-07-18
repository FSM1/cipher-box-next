# Spec title

<!--
TEMPLATE — every as-built spec follows this structure.
Kind "part" = a package/crate/app; kind "flow" = a cross-cutting system behavior.
Sections marked (part) or (flow) apply only to that kind; drop the marker in real specs.
Document flows, data structures, and invariants — NOT implementation. A reader should be
able to reimplement the behavior from this spec without opening the source.
Describe what IS, including bugs and drift — idealizing belongs in the blueprint, not here.
-->

| | |
| --- | --- |
| **Kind** | part \| flow |
| **Sources** | `path/in/cipher-box`, … |
| **Verified against** | cipher-box `<commit sha>` |
| **Status** | draft \| agreed |

## Purpose and scope

One or two paragraphs: what this part/flow is for, and the boundary of this spec. Link the
specs that cover adjacent ground instead of restating them.

## Vocabulary

Terms this spec introduces or leans on, one line each. Reuse the project's canonical
terminology (`publicKey`, `ipnsName`, `keyEpoch`, …); if the implementation uses a
divergent name, note both.

## Actors and trust boundaries

Who participates — web client, desktop client, API, TEE worker, IPFS network, database —
and, for each: what plaintext or key material it can see, and what it must never see.
This section is load-bearing in a zero-knowledge system; every flow below must be
consistent with it.

## Data structures

For each structure this spec owns: name, fields (with types), encoding (CBOR / JSON /
protobuf / DB row), where it lives (IPFS block, IPNS record, DB table, memory only),
who writes it, who can read it, and who can decrypt what. Prefer field tables over prose.
For mutable structures, include the **write discipline**: who may write, under what
concurrency gate (CAS on which field, equality check, tombstone), and what happens on a
lost race — this lives here, with the structure, not scattered through the flows.
Structures owned by another spec are linked, not restated. When this spec *constrains* a
structure or plane another spec owns (e.g. it must never advance a sequence number),
state the constraint as an invariant here and link the owning spec.

## Interface (part)

The part's public surface: what it exposes, to whom, and the responsibilities it owns.
Exports/endpoints at the level of capability ("seals a child ref", "relays an IPNS
publish"), not function signatures.

## Dependencies (part)

Which other parts this one depends on and for what — one line each.

## Flows (flow) / Behaviors (part)

The heart of the spec. For each operation, a numbered sequence narrative:

### Operation name

- **Trigger** — what starts it (user action, timer, event).
- **Preconditions** — what must already hold.
- **Steps** — numbered, one actor per step, naming the data structures exchanged.
  Use a Mermaid `sequenceDiagram` when three or more actors interact. No semicolons in
  Mermaid message text — `;` is a statement terminator and breaks the parse.
- **Postconditions** — what is true after; what observers can now see.
- **Failure modes** — what happens on each realistic failure; what state is left behind.

## Runtime variants

Deployment or configuration modes that change behavior — production vs simulator, env
flags that alter a flow, defaults that differ per environment. Behavior-affecting
variants only; this is not an ops runbook. Omit the section when there are none.

## Testing posture

Optional. What actually gates this part/flow in CI versus what is skipped, quarantined,
or main-push-gated — as fact, not aspiration. Include when the gap between apparent and
real coverage would surprise a reader.

## Invariants

Numbered MUST / MUST NEVER statements — the checkable properties a reimplementation has
to preserve. Each invariant stands alone; reference them elsewhere as `INV-n`.

## Known gaps and quirks

Where the implementation deviates from intent: bugs, dead code, half-wired features,
platform-specific workarounds. Cite the evidence (file, ticket, todo) so a reader can
judge freshness.

## Rewrite notes

The pain this area caused and why — what a redesign must not repeat, what it could drop.
This section feeds the blueprint design tickets; it is opinion, clearly separated from
the as-built record above.
