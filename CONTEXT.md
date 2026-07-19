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

- **Write seed** — the write plane's parallel derivation, flat within a write scope: `writeSeed(X) = KDF(writeScopeSeed, X.id)`. A node's Ed25519 IPNS keypair, its `ipnsName`, and its `writeKey` all derive from its write seed. Read grants carry read seeds only; write grants carry both.
- **Write-scope cut** — granting write on an interior folder anchors a fresh write scope there: flat derivation means one node's write seed cannot derive its children's, so the grant mints a write scope seed and runs a name wave over the subtree.
- **Name wave** — the background, child-first republish of a write-rotated scope under freshly derived names, root re-pointed last. Survivor write-grantees derive new names locally; read-only survivors follow republished mirrors (root via mailbox re-point).
- **Forgery window** — accepted residual: during the name wave a revoked writer can still seal plausible old-epoch records at not-yet-rotated old names. Bounded by wave duration; old names retire from the republisher inventory (tombstones advisory only).

## Envelope and grants

- **Envelope** — the published plaintext wrapper of a node: format version, node id, epoch tag, the sealed bodies, and (at scope roots only) the grant section. Kind-uniform: whether a node is a file or a folder is sealed inside the read-body.
- **Write-body** — the sealed write-plane payload, present only at scope roots: the grant ledger plus the write-plane history link. Interior nodes publish none; all their write material is derived.
- **Blinded tag** — the opaque per-pair label a grant blob is keyed by, derived from an owner–recipient shared secret and the scope root's name. Observers learn only blob count; the recipient self-locates their blob.
- **Grant blob** — a scope's current seed(s) sealed to one recipient and keyed by their blinded tag, carried in the scope root's envelope. Read grants carry the read seed; write grants carry both seeds.
- **Grant ledger** — the authoritative record of a scope's grants, `(recipient, permission, tag)`, sealed in the scope root's write-body. Writers re-wrap blobs for the recorded set during re-seals but cannot change the set; grant changes are owner-only.
- **Grant-set commitment** — the owner-signed `(tag, permission)` list in the scope root's envelope. Recipients verify their tag is committed before trusting a grant. Deliberately epoch-free, so grantee-triggered rotation needs no owner signature.
- **Encryption subkey** — the sealing key each user derives from their login secret, published signed by their identity key. The identity key only signs; the subkey only seals.
- **Vault pointer** — the stable IPNS name derived from the owner's login secret whose record points at the current vault root. The cold-boot entry point; root write rotation re-points it.

## Sync and refresh

- **Sync timing profile** — the environment-scoped bundle of sync timing policy: record TTL, poll cadence, staleness thresholds, offline staging budget, escalation window. Records always set TTL explicitly; TTL is independent of EOL.
- **Focus window** — the set a poll tick refreshes: the vault pointer, the open folder, and its ancestor chain to root. Everything else refreshes on access.
- **Pending-op overlay** — the client state law: rendered state is the last-known-good remote snapshot with queued ops applied on top; the op queue is the only local divergence.
- **Op queue** — the durable FIFO journal of intent ops every mutation rides. Offline replay and online CAS rebase re-apply ops through the same path.
- **Adoption gate** — the trust pipeline every resolved record passes before adoption: signature verify, strictly-newer sequence vs floor, epoch at or above the scope floor, unseal success. A failure is a fail-closed trust violation, never mere staleness.
- **Conditional delete** — a delete op snapshots its target's own record sequence; on rebase it is dropped if the target advanced. Edit wins in both directions: a rebased edit resurrects a concurrently deleted node.
- **Observed repair** — cleanup of an invariant violation noticed at read time by any write-capable client, e.g. removing the losing link of a dual-linked child.
- **Dual-link** — one child linked in two parents, the benign crash residue of a dest-first move; repaired deterministically via the child ref's monotonic link counter.
- **Dead-letter** — a queued op that terminally fails rebase, surfaced to the user with any staged content preserved rather than silently dropped.
- **Withheld-update escalation** — the shared-scope warning raised when a name stays pinned past a profile window while other resolves succeed; the reader-side signal for a stale-view pin.
