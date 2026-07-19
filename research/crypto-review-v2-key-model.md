# Adversarial crypto review — v2 derivation and rotation model

| | |
| --- | --- |
| **Ticket** | [#35](https://github.com/FSM1/cipher-box-next/issues/35) |
| **Reviews** | [#26 rotation](https://github.com/FSM1/cipher-box-next/issues/26), [#25 sharing/grants](https://github.com/FSM1/cipher-box-next/issues/25), [#27 schemas/envelope](https://github.com/FSM1/cipher-box-next/issues/27); context [#23](https://github.com/FSM1/cipher-box-next/issues/23)/[#24](https://github.com/FSM1/cipher-box-next/issues/24) |
| **Baseline** | `CONTEXT.md`, `specs/flows/rotation.md`, `specs/flows/sharing-grants.md`, `specs/parts/crypto.md` (v1) at `435a861` |
| **Stance** | Adversarial. Goal is to break the design; sound surfaces are listed at the end as evidence of coverage. |

## Threat model recap (what the design promises)

- Server/relay and IPFS/IPNS are **untrusted**; the relay hosts the pointer mailbox,
  the people directory, the republisher inventory, and serves resolution.
- The client is trusted; the Web3Auth secret is the identity root ("identity compromise
  is game-over").
- Grants are owner-only; write-grantees may re-wrap **committed** grants during a
  self-triggered read rotation but "can neither extend nor shrink the set."
- Anti-rollback: mandatory client floors on `{scope, epoch}` + per-name IPNS seq, seeded
  from the grant blob's "owner-vouched epoch"; the design claims this "structurally
  closes v1's optional-injection hole."
- The owner is "an implicit, permanent, unrevokable grantee of every scope" (owner blob =
  O(1) recovery).
- Revocation promise: "they keep what they saw; they lose everything new, now"; the
  revokee residual is "stale interior metadata for a sweep-length window — never a live
  grant, never anything sealed after the cut."
- Forgery-window residual is "bounded by wave duration."

The review targets the gap between these claims and what the pinned constructions
actually enforce.

---

## Findings

### F-1 [HIGH] Grant blobs and other seed-bearing sealed structures are unauthenticated and unbound from the owner commitment

**Design decisions touched:** #27 grant-blob format (`tag → HPKE{seeds, epoch}`, base-mode
HPKE); #25 D7 owner-signed commitment "generalizes and replaces the recipient-pin
anti-substitution defense against relay, grantee, and URL-holder alike"; #26 D4 floors
"seeded from the grant blob's owner-vouched epoch."

**The gap.** The owner-signed commitment binds `{ipnsName, [(tag, permission)]}` — the
tag SET and permissions — but **not the contents of each grant blob** (the sealed seed
and epoch). Grant blobs are RFC 9180 HPKE in base mode, which provides recipient
confidentiality and ciphertext integrity but **no sender authentication**. The blob is
sealed to the recipient's X25519 encryption subkey, which is **public** (published in the
people directory). Therefore anyone who can place a blob at the recipient's tag — the
relay serving the envelope, or any writer — can mint a *valid* HPKE seal to the recipient
carrying **attacker-chosen `{seed', epoch'}`**. A read-only recipient never sees the
write-body ledger (no `writeKey`), so their only anti-substitution check is
"is my tag in the commitment?" — which a forged blob at the correct tag passes.

**Attack (relay forgery → floor poisoning + DoS):**

1. Owner grants Carol read on scope root `R`. Envelope carries Carol's real blob at
   `tagCarol` and commitment `C = sign{R, [(tagCarol, read)]}`.
2. Carol's client (new device, cold floor) resolves `R`. The relay serves the envelope
   but swaps the blob at `tagCarol` for a forgery: `HPKE(CarolEncPub, {seed'=random,
   epoch'=2^40})`.
3. Carol checks the commitment: `tagCarol` is committed → she trusts the blob. She
   HPKE-unseals it (valid — sealed to her public key), obtains `seed'` and `epoch'`.
4. Carol seeds her durable `{scope=R}` floor to `epoch'` (a huge value).
5. Carol derives keys from `seed'`, resolves `R`'s nodes → every AEAD unseal fails
   (real nodes are under the real seed). Carol cannot read the scope.
6. Even after the relay stops interfering, every *real* record (`epoch << 2^40`) is
   now **below Carol's floor** and is rejected fail-closed. Carol is **persistently
   denied** until her floor store is manually reset.

**Attack escalation (confidentiality for write-grantees):** If Carol is a **write**-
grantee and the forged blob carries `seed'` chosen by the attacker, and Carol's client
writes a new node *before* a successful read (or ignores read failures), the new node is
sealed under `KDF(seed', id)` and published to `writeSeed`-derived names that the attacker
(knowing `seed'`) can also derive and read. Carol's newly authored content leaks to the
attacker. This is conditional on client write-before-read ordering, but confidentiality
must not rest on client call order.

**Why the commitment does not help.** #25 D7 advertises the commitment as the
anti-substitution defense "against relay, grantee, and URL-holder alike." It defends
against a wrong-*tag* substitution but not a wrong-*blob-at-the-right-tag* substitution,
which is the one the relay can always perform because the sealing target is public. The
"owner-vouched epoch" characterization in #26 D4 is therefore inaccurate: the epoch a
cold device seeds its floor from is asserted by whoever wrote the blob (or forged it),
with no owner signature over it.

**Recommended amendment:**

- Authenticate the blob by its writer. Use **HPKE auth mode** (sender = the rotator's
  write-plane key) or attach a signature by the rotator over the blob, and record the
  authorized rotator identity so the recipient can verify the blob came from a party
  holding write authority for `R` (owner or a committed write-grantee) — not from the
  relay. This blocks relay forgery outright while preserving grantee re-seal.
- Seed the anti-rollback floor **only from an epoch confirmed by a successful node
  unseal** under the derived seed, never from the raw blob field. A forged blob (wrong
  seed) then never yields a confirmed read and cannot poison the floor; it degrades to an
  availability event indistinguishable from the relay withholding service.
- Keep the epoch-free commitment for grantee-triggered rotation, but stop treating the
  commitment as blob-integrity: state explicitly that it binds tag membership only.

---

### F-2 [HIGH] Write-rotation root re-point is deliverable only over the untrusted mailbox — a colluding relay plus a revoked writer pins read-only survivors to the old root name indefinitely

**Design decisions touched:** #26 D3 name wave, "read-only survivors follow republished
mirrors plus the mailbox re-point for the root"; #25 D2 "write rotation re-points
root-granted survivors via the same [relay] mailbox"; #24 keyless re-PUT + inventory
retirement; the accepted forgery-window residual "bounded by wave duration."

**The gap.** Under write rotation every name changes. Write-grantees derive the new names
locally, but **read-only survivors cannot derive names** and learn the new root name
**only** from the relay-hosted pointer mailbox. The mailbox is explicitly integrity-
untrusted ("the relay can neither forge nor usefully replay" — but it can **withhold**).
Separately, IPNS re-PUT is keyless by design (#24 D1/D2), so a revoked writer who retains
the old root's Ed25519 signing key can keep the *old* root record alive on any endpoint
and keep signing new-sequence **old-epoch** records at it.

**Attack:**

1. Owner write-rotates scope `R` to revoke writer Mallory. Owner posts the new-root
   re-point to the mailbox for read-only survivor Sam, retires the old root name from the
   CipherBox republisher inventory, and considers the rotation done.
2. The relay (untrusted) **drops Sam's mailbox re-point**. Sam's vault bookmark still
   holds the *old* root name; Sam has no other signal that it changed.
3. Mallory (revoked) keeps the old root record alive: she keyless-re-PUTs it to public
   delegated endpoints and signs fresh **higher-sequence** records at the old name with
   the old Ed25519 key and old read seeds — forged old-epoch envelopes with content of
   her choosing (drawn from the pre-rotation seed she still holds).
4. Sam resolves the old root name → higher sequence wins → Sam gets Mallory's forged
   record. Sam's floor for `R` is at the pre-rotation epoch (he never saw the new epoch),
   so it does not fire. Sam sees a Mallory-controlled view of `R` **indefinitely**.

**Why the claimed bound is wrong.** The residual is documented as "bounded by wave
duration," but the effective bound for a read-only survivor is "until the mailbox
re-point is delivered *and* processed." Since that channel is untrusted and withholdable
by a single relay message drop, and the old name is kept resolvable by the revokee's own
keyless re-PUT, the window is unbounded. No confidentiality of *new* content is lost
(Mallory cannot seal the new epoch), but a **revoked** party controls a **surviving**
user's view of the scope forever — a direct defeat of the write-revocation guarantee.

**Recommended amendment:**

- Do not make the mailbox the sole re-point channel. Give the surviving reader an
  owner-authenticated forwarding record: e.g., the *old* root name's final owner-signed
  record carries an owner-signed `movedTo: newRootName` pointer (a cryptographic tombstone
  the revokee cannot forge because it is signed by the owner's identity key, not the IPNS
  key). Readers treat a signed `movedTo` as authoritative and migrate; the revokee's
  unsigned forgeries at the old name are then rejected.
- Alternatively, bind the read-only survivor's bookmark to an owner-identity-signed name
  chain so a name change is verifiable without the mailbox.
- At minimum, require the reader to treat "old name still resolving with no owner-signed
  liveness/forwarding proof past its retirement" as a hard staleness signal rather than
  silently trusting higher-sequence records.

---

### F-3 [MEDIUM] A malicious write-grantee can poison or withhold co-grantee blobs, the owner blob, and ascent links during a self-triggered rotation — deniably revoking co-grantees and locking the owner out of O(1) recovery

**Design decisions touched:** #26 D5 grantee re-seal "for the committed set verbatim...
they can neither extend nor shrink the set"; CONTEXT.md owner blob "the owner is an
implicit, permanent, unrevokable grantee of every scope... O(1) recovery"; ascent link
"any writer can re-seal them."

**The gap.** A write-grantee's scope-exit rotation re-writes, without owner mediation:
every grant blob (for committed tags), the owner blob, and the ascent link. None of these
are owner-signed or writer-authenticated (see F-1). The owner-signed commitment fixes the
tag *set* but cannot compel the grantee to write a *correct seed* into each blob. So while
the grantee "cannot extend or shrink the set" on paper, they fully control the effective
access each committed party receives.

**Attack (deniable co-grantee revocation):**

1. Owner grants Wanda write and Carol read on `R`. Commitment lists both tags.
2. Wanda legitimately triggers a scope-exit read rotation (she moved a file out). She
   mints the new override seed and must re-seal blobs for `tagWanda` and `tagCarol`.
3. Wanda writes her own blob correctly but writes **garbage** into `tagCarol`'s blob
   (or omits it). The commitment still lists Carol; the owner believes Carol is granted.
4. Carol looks up `tagCarol`, unseals a blob that yields a bad seed, and can read
   nothing new. She cannot distinguish this from corruption/staleness. Wanda has
   unilaterally revoked Carol — an owner-only act — undetectably.

**Attack (owner lock-out of a sub-scope):**

1. As above, Wanda additionally writes a bad seed into the **owner blob** and re-seals
   the **ascent link** to a public key she controls (both are unauthenticated).
2. The owner's O(1) recovery (owner blob) now yields a wrong seed; the owner's descent via
   the ascent link fails (sealed to Wanda's key). The owner cannot derive `R`'s current
   seed.
3. The owner can mint a fresh override seed (rotation needs no old seed) but cannot
   *re-seal existing interior nodes* without first reading them under the seed she has
   lost — so the interior content is stranded. The "unrevokable owner grantee / O(1)
   recovery" invariant is broken by a write-grantee.

This is within a write-grantee's destructive power (they could also delete content), but
it is **deniable** (looks like a bug), it **contradicts stated invariants** ("owner-only
grant authority," "unrevokable owner grantee"), and it **defeats recovery** rather than
being visibly destructive.

**Recommended amendment:**

- Authenticate seed-bearing structures by their writer (same HPKE-auth/signature fix as
  F-1), and record the authorized rotator so the owner/co-grantees can attribute a
  poisoned structure to a specific write-grantee (accountability + the ability to reject
  a structure from a party not currently authorized).
- Do not let the owner's recovery depend on grantee-authored owner blobs. The owner
  should keep an independent, self-authored record of each scope seed it has granted (or
  re-assert its own owner blob on every visit), so a grantee cannot strand the owner.
- Have ancestor-scope readers **derive** the expected ascent public key from the parent
  seed and reject a mismatched plaintext public half, rather than trusting it; and have
  the owner cross-check owner-blob seed vs ascent-link seed vs actual node decryption,
  flagging disagreement as abuse rather than failing silently.

---

### F-4 [MEDIUM] The eager set excludes grant-less descendant scope roots, so their stale ascent links leak the current seed (hence post-cut content) to an ancestor-scope revokee for the sweep window

**Design decisions touched:** #26 D2 / CONTEXT.md eager set = "the rotated scope root plus
every descendant scope root **carrying a live grant**"; accepted residual "never anything
sealed after the cut."

**The gap.** The eager-set rationale is precisely "an ancestor-scope revokee knows their
override seeds" — i.e., a revokee of a parent scope can descend into descendant scope
roots via ascent links until those links are re-sealed. But the eager set is restricted to
descendant scope roots **with a live grant**. A descendant scope root `S` that has **no
live grant** (e.g., a folder granted then fully revoked — its scope persists) still has an
ascent link derivable by parent-scope readers, and it is left to the **lazy** sweep.

**Attack:**

1. Uncle holds a read grant on parent scope `P` (so he holds `scopeSeed_P`). `P` contains
   a descendant scope root `S` with no current grantee.
2. Owner revokes Uncle: `P` rotates (new `scopeSeed_P'`). The eager set re-seals `P`'s
   root and descendant scope roots *with live grants*, but **not** `S`.
3. Uncle, using his old `scopeSeed_P`, derives `nodeSeed_P(S)` → the ascent private key →
   unseals `S`'s not-yet-re-sealed ascent link → obtains `S`'s **current** override seed.
4. Any content written to `S` during the sweep window is sealed under that seed and is
   therefore readable by Uncle — content "sealed after the cut," which the accepted
   residual promises Uncle can never read.

The leak is bounded by sweep duration (not unbounded) and requires a grant-less
descendant scope plus a writer active during the window, but it **violates the stated
bound** of the accepted residual and concerns the owner's own content in `S`.

**Recommended amendment:** Include **every** descendant scope root reachable via ascent
links by the revokee in the eager set (re-seal all descendant ascent links eagerly), not
only those carrying a live grant. The "live grant" optimization is unsound because a
grant-less sub-scope still holds owner-confidential content and still exposes a stale
ascent link to the revokee.

---

### F-5 [MEDIUM] The vault-pointer blob is encrypted-not-authenticated and the pointer key never rotates — pointer-key compromise short of full-secret compromise redirects the owner to a forged vault root

**Design decisions touched:** #27 D9 bootstrap "vault-pointer IPNS name whose Ed25519
keypair derives from the Web3Auth secret (stable, never rotates)... points to an
owner-sealed `{rootIpnsName}`."

**The gap.** The pointer record's target blob is described as "owner-sealed," i.e.
HPKE-encrypted **to** the owner's public subkey. Sealing *to* the owner requires only the
owner's **public** key (public), so anyone can craft a blob the owner can open. The blob
carries no owner **signature**. The only integrity on the pointer→blob binding is the IPNS
record signature by the pointer's Ed25519 key. Because that key never rotates, its
compromise is permanent and unrecoverable.

**Attack:**

1. Attacker obtains the pointer Ed25519 private key (needed only at publish time; the
   never-rotate policy maximizes exposure over the vault lifetime).
2. Attacker publishes a higher-sequence pointer record pointing to a blob they craft:
   `HPKE(ownerSubkeyPub, {rootIpnsName = attackerRoot})`.
3. On next cold boot the owner resolves the pointer, unseals the blob (it is sealed to
   the owner, so it opens), and navigates to `attackerRoot` — a forged vault the attacker
   populated. The owner is phished into trusting/writing against attacker content; owner
   writes may be sealed under attacker-influenced material.

Full Web3Auth-secret compromise is already game-over (agreed), but **pointer-key-alone**
compromise should not be, and today it yields owner-root redirection, not mere DoS.

**Recommended amendment:**

- Make the pointer blob **owner-identity-signed** (ECDSA over det-CBOR, same suite as the
  commitment). The client verifies the signature against the owner's identity key before
  trusting `rootIpnsName`; a pointer-key-only attacker can then only DoS (publish junk the
  client rejects), never redirect.
- Reconsider "never rotates": provide a recovery/rotation path for the pointer (e.g., a
  secret-derived successor pointer name) so a pointer-key compromise is not a permanent,
  unfixable SPOF.

---

### F-6 [MEDIUM] Sealing-to-a-person depends on a directory subkey binding whose verification and identity-key authentication are not mandatory

**Design decisions touched:** #27 D3 X25519 subkey "published in the people directory
signed by their secp256k1 identity key"; #25 D7 "people-directory... with
fingerprint-verification UX as **optional** hardening against directory substitution."

**The gap.** Every seal-to-a-person (grant blobs, owner blob, mailbox pointers) targets
the recipient's X25519 subkey fetched from the relay-hosted directory. Two checks must
hold or grants are redirected to an attacker: (a) the subkey binding signature must be
verified against the recipient's **identity** key at seal time, and (b) the identity key
itself must be authentic. v1 established authenticity by out-of-band paste ("the only
moment the recipient's identity is authentic"). v2 introduces a directory; if directory
lookup becomes the default and fingerprint verification stays "optional," a malicious
directory can return an attacker identity key + a self-consistent attacker subkey binding,
and the owner wraps grants to the attacker.

**Attack:**

1. Owner shares with "Bob" by directory lookup on a display name / handle.
2. Relay returns attacker `identityPk_A` and `subkey_A` with a valid self-signature
   (attacker signed its own binding).
3. Owner (no out-of-band check) HPKE-seals the grant to `subkey_A`. The attacker, not
   Bob, receives readable access.

**Recommended amendment:**

- The seal-to-a-person flow MUST verify the subkey binding signature against the
  recipient identity key (non-optional).
- The **identity key** must be authenticated out-of-band (paste or mandatory fingerprint
  confirmation) before a first grant; directory lookup may accelerate discovery but must
  not be the sole trust root. Present directory-only grants as bearer-of-substitution-risk
  in the UI, consistent with how bearer invite links are surfaced.

---

### F-7 [LOW] Cold-start floors have no owner-vouched current-epoch anchor — the accepted "within-epoch" staleness residual is actually across-epoch and effectively unbounded on cold devices

**Design decisions touched:** #26 D4 mandatory floors "structurally close v1's
optional-injection hole"; #27 D7 accepted residual "within-epoch cold-first-contact
staleness."

**The gap.** Floors prevent *regression* once seeded, but on a cold device the attacker
(transport-controlling relay) chooses the first record served, hence the initial floor
seed. Nothing the device can verify tells it that a higher epoch exists: the commitment is
epoch-free (F-1), the grant-blob epoch is unauthenticated (F-1), and IPNS gives no trusted
sequence reference to a device with no prior state. So a cold device can be seeded a
**pre-revocation** epoch and shown a rolled-back view spanning a revocation boundary, not
merely within one epoch. This is largely the accepted residual, but the documented scope
("within-epoch") understates it: it is across-epoch and, against a transport that serves
only stale records, does not self-heal.

**Recommended amendment:** Provide a cold device a verifiable freshness anchor that does
not break grantee-triggered rotation. Options: an owner-identity-signed
`{scope → minEpoch}` high-water carried in the mailbox pointer / vault bookmark and bumped
by the owner on each rotation (owner-only, so grantee rotations do not need it but cannot
lower it); or anchor first-contact freshness to the IPNS EOL/sequence plus a trusted
timestamp. At minimum, re-scope the documented residual to "across-epoch, self-heals only
on an honest resolve," so the "structurally closed" language is not over-claimed.

---

### F-8 [LOW] UUID uniqueness is security-critical but relies on unenforced global uniqueness; node name and ipnsName are not AAD-bound

**Design decisions touched:** flat derivation `nodeSeed(X)=KDF(scopeSeed, X.id)`,
`writeSeed(X)=KDF(writeScopeSeed, X.id)`; #27 D4 AAD = `(v, id, scope, epoch, structure
tag)`; child refs `{id, name, ipnsName, kind}`.

**The gap.** `X.id` is the **sole** discriminator of a node in both the key derivation and
the AAD. The node's `name` and `ipnsName` are not bound into the AAD. So two nodes sharing
`(id, scope, epoch, structure)` have byte-interchangeable sealed bodies (identical key,
identical AAD), and a cross-scope move onto a pre-existing colliding id (or a maliciously
crafted duplicate id within a scope) collapses two nodes onto one key. Random 122-bit
UUIDs make accidental collision negligible, but the invariant is unstated and unenforced,
and a malicious writer inside a scope can create duplicate ids deliberately.

**Recommended amendment:** State global UUID uniqueness as a normative invariant; have
decoders reject duplicate ids within a scope (and duplicate `ipnsName`s) fail-closed.
Consider binding `ipnsName` into the AAD so the sealed body is pinned to its published
name, closing id-collision transplant regardless of uniqueness enforcement.

---

### F-9 [INFO] Enumerate and pin the KDF edge/context catalog; specify id as a length-delimited keyed-hash input, not a variable derive_key context

**Design decisions touched:** #27 D3 "BLAKE3 `derive_key` with a distinct context string
per edge."

**Observation.** The read/write/structure/scope separation is sound *iff* every edge uses
a distinct, fixed context and node ids enter as unambiguous, length-delimited inputs.
BLAKE3 `derive_key` contexts are meant to be hardcoded global constants; folding a
variable `X.id` into the context string (rather than passing it as keyed-hash message
material) invites concatenation ambiguity if any variable-length field is ever added
adjacent to it. The read plane (`nodeSeed`, `readKey`), write plane (`writeSeed` →
Ed25519/ipnsName/writeKey), ascent-key derivation from the parent node seed, the
per-structure sealing keys, and the X25519-subkey-from-secp256k1 derivation should be
enumerated in one frozen catalog with a cross-language KAT (the v1 `hkdf.json` +
count-guard discipline is the proven pattern to inherit).

**Recommended amendment:** Publish the edge catalog (context string + input layout per
edge) in the schemas spec; derive per-node material as
`keyed_hash(key=derive_key("<fixed edge context>", scopeSeed), message=id16)` so ids are
fixed-length message input, never variable context; add a both-sides KAT that asserts
read/write/scope/structure separation (no two edges produce equal output for equal
inputs).

---

## Attack lines checked and found sound

The following were probed adversarially and hold up under the pinned constructions;
their soundness is evidence of coverage, not omission.

- **Read↔write plane separation.** `nodeSeed` and `writeSeed` use independent random
  scope seeds and distinct KDF edges; even if a scope's read and write seeds were equal,
  distinct contexts prevent collision. No cross-plane derivation.
- **Transplant across node/scope/epoch/structure/version.** The per-node key is unique
  per `(scopeSeed, id)` and `scopeSeed` is fresh per epoch, so a sealed body cannot open
  under another node/scope/epoch even before the AAD check; the AAD additionally binds
  `v, id, scope, epoch, structure tag`; `v` is AAD-bound (downgrade fails); per-version
  content keys are random and sealed inside the read-body (no single-version transplant).
- **History-link ratchet direction.** Previous seed sealed under current seed: current
  holders walk back, revoked holders (holding an older seed) cannot walk forward, across
  multiple rotations and lagged sweeps. A revokee who was also a past write-grantee holds
  only old write seeds → old (retired) names; the write-plane history link is likewise
  backward-only. Knowing an old seed as plaintext does not reveal the current seed that
  sealed it.
- **Eager-set cut for live-grant descendant scopes.** Descendant scope roots *with live
  grants* are re-sealed eagerly, correctly cutting an ancestor-scope revokee at those
  points (the gap is only the grant-*less* case, F-4).
- **Ascent-link confidentiality.** The plaintext public half leaks nothing about the
  parent seed (one-way derivation); a revoked child-scope grantee cannot unseal it (needs
  the parent seed) and cannot walk upward; a malicious writer cannot make an ancestor
  reader unseal a *false* seed (the reader derives the true ascent key), only DoS descent
  (covered under F-3).
- **Blinded-tag unlinkability.** `tag = derive(ECDH(ownerEnc, recipientEnc) ‖ ipnsName)`
  is uncomputable to observer, relay, or co-recipient without a private key; tags are
  stable across read rotation (name-stable) and change under write rotation (name change,
  owner re-signs the commitment); cross-scope and cross-name tags for the same recipient
  do not link to an observer. A recipient can self-probe only their own tags.
- **secp256k1 ↔ X25519 cross-protocol.** The X25519 subkey is a KDF output, not the raw
  scalar; ECDSA is RFC 6979 deterministic; HPKE and X25519 manage their own randomness —
  no nonce/randomness reuse across schemes; the shared secret input is KDF-separated.
- **XChaCha20-Poly1305 nonce hygiene.** 24-byte random nonces with per-seal freshness sit
  far inside the birthday bound; HPKE manages its own nonces. No reuse surface.
- **Cross-scope move leakage.** Independent per-scope random seeds mean a source-scope
  holder derives nothing about destination-scope material; the destination re-seal at the
  destination epoch changes both key and AAD scope/epoch, blocking replay of the moved
  body into the destination.
- **Permission replay via the epoch-free commitment.** Replaying an old commitment with a
  higher, since-downgraded permission does not restore capability: write capability is key
  possession (write seed → signing key), not a commitment claim; after write rotation the
  old keys sign only retired names.
- **Content confidentiality after revocation.** New epochs use fresh random seeds and the
  next content version is fresh-keyed; the revokee's retained seeds open only pre-cut
  ciphertext ("presumed leaked"), consistent with the ADR-0002 successor stance.

## Accepted residuals — bounds verification

- **Forgery window "bounded by wave duration":** bound is **wrong** for read-only
  survivors — see F-2 (mailbox-withholding + keyless re-PUT makes it unbounded). For
  write-grantee survivors (derive names locally) the wave-duration bound holds.
- **Within-epoch cold-first-contact staleness:** confirmed present, but scope is
  **understated** — it is across-epoch on cold devices (F-7).
- **Revokee reads stale interior metadata for a sweep-length window:** bound holds for
  interior nodes and live-grant sub-scopes, but is **exceeded** for grant-less descendant
  scope roots, where the current seed (not just stale metadata) leaks (F-4).
- **Relay sees transient sender/recipient mailbox edges; scope roots reveal grant-blob
  count:** confirmed as stated, no amplification found.
