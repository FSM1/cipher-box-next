# CipherBox v2 — Ubiquitous Language

Glossary only. Implementation detail lives in `specs/`; decisions in issue resolutions and `docs/adr/`.

## Read-plane key model (seeded derivation)

- **Node seed** — the per-node secret from which a node's keys derive, flat within a scope: `nodeSeed(X) = KDF(scopeSeed, X.id)`. Location-independent, so intra-scope moves are pure relinks; membership is the envelope's epoch tag.
- **Scope** — a derivation domain: the set of nodes keyed from one random scope seed. The vault root is the initial scope; every grant on an interior folder and every rotation creates or re-seeds one.
- **Scope root** — the node a scope is anchored at, carrying its grant blobs, owner blob, ascent link, and history links. Granting an interior folder makes it a scope root (with a subtree sweep into the new scope).
- **Content key** — a random per-version key stored inline in the sealed read-body. Rotation re-wraps it for free via the metadata re-seal and never re-encrypts content bytes; the next version is fresh-keyed. There is no pending-re-key state.
- **Cross-scope move** — moving a node between scopes re-seals the moved subtree at the destination scope's epoch (and is a scope exit for the source when the source is granted).
- **Override seed** — the fresh random seed minted at a scope root by a rotation, replacing its derived (or prior override) seed and starting a new epoch.
- **Epoch** — a scope-relative rotation counter. Every sealed body carries a plaintext, AAD-bound **epoch tag** `{scope, epoch}`.
- **Ascent link** — the current override seed sealed to an asymmetric keypair derived from the *parent's* node seed (never the child's own): the public half rides plaintext in the scope root's envelope so any writer can re-seal the link; unsealing requires the parent seed, so only ancestor-scope readers descend. Deliberately unreadable to holders of the scope's own prior seed.
- **Owner blob** — the current override seed ECIES-wrapped to the vault owner's identity key, present at every scope root and maintained by whoever rotates. The owner is an implicit, permanent, unrevokable grantee of every scope; this is also the owner's O(1) recovery entry into any scope.
- **History link** — the previous epoch's seed sealed under the current one; the key-regression ratchet. Current-seed holders walk backward to read not-yet-resealed nodes; old-seed holders can never walk forward.
- **Lazy wave** — post-rotation re-sealing of a scope's descendants at the new epoch, carried by ordinary writes and/or a background sweep rather than an eager cascade.
- **Epoch lag** — a node whose envelope epoch is behind its scope's current epoch; readable via history links, queued for the lazy wave. Replaces v1's "dirty edge".
- **Eager set** — the nodes a rotation must republish before it counts as cut: the rotated scope root plus every descendant scope root carrying a live grant. Interior nodes ride the lazy wave.
- **Epoch-converged** — a subtree with no epoch-lagged nodes. Granting a subtree requires converging it first (a new grantee cannot regress through an ancestor scope's history).

## Write-plane key model

- **Write seed** — the write plane's parallel derivation tree: `writeSeed(child) = KDF(writeSeed(parent), child.id)`. A node's Ed25519 IPNS keypair, its `ipnsName`, and its `writeKey` all derive from its write seed. Read grants carry read seeds only; write grants carry both.
- **Name wave** — the background, child-first republish of a write-rotated scope under freshly derived names, root re-pointed last. Survivor write-grantees derive new names locally; read-only survivors follow republished mirrors (root via mailbox re-point).
- **Forgery window** — accepted residual: during the name wave a revoked writer can still seal plausible old-epoch records at not-yet-rotated old names. Bounded by wave duration; old names retire from the republisher inventory (tombstones advisory only).
