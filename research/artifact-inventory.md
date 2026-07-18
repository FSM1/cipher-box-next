# CipherBox Reusable Design-Artifact Inventory

Resolution research for wayfinder ticket #4. Sweep of the local checkout at
`cipher-box/` (`/Users/myankelev/Code/random/cipher-box`), covering `docs/` (incl.
`docs/adr/`), `.planning/research/`, `.planning/codebase/`, `.planning/design/`,
`.planning/adr/`, `.planning/security/`, top-level `.planning/` state docs, and a
sample of architecture-relevant phase dirs (61, 62, 63, 64, 66, 67, 71, 75, 77, 80).
All paths below are relative to the cipher-box repo root.

Sweep date: 2026-07-18. Freshness was spot-checked against live code: the repo is in
the **node/v3 + rotation era** (phases 61–80 shipped through PR #615). Verified
present: `packages/core/src/node/`, `packages/sdk-core/src/rotation/`,
`ipns_records` (renamed from `folder_ipns`), `share_keys` deleted, `recipientPins`
in `packages/core`, KAT vectors `tests/vectors/node-codec.json` and
`tests/vectors/crypto/node-aad.json`. The legacy `packages/core/src/folder/` model
(`FolderMetadata`/`FileMetadata`/`FilePointer`/`folderKey`) no longer exists — any
doc presenting it as current has drifted.

## Freshness legend

- **CURRENT** — rotation-era (phases 61–80), matches live code.
- **STABLE** — older but refactor-invariant (auth, infra, conventions); still accurate.
- **PARTIAL** — mixed: some sections current, some superseded.
- **STALE** — describes the retired v1 metadata/sharing model as current.
- **PARKED** — research for a milestone that never landed (m4 productivity suite).

## High-value / read-first

1. `.planning/design/2026-06-26-sharing-read-keychaining-design.md` — **CURRENT**.
   The single source of truth for the v2.0 refactor (named as such by PROJECT.md and
   REQUIREMENTS.md): unified Node model, two sealed bodies, AAD seal, read
   key-chaining, resumable rotation engine, TEE contract. Feeds nearly every ticket.
2. `.planning/research/grant-delivery-rotation-research-goals.md` — **CURRENT**
   (2026-07-16, newest artifact; zero drift). The open-design frontier: where grant
   key material lives (relay vs metadata vs inbox), rotation re-delivery, residual
   relay trust. RQ1–9, H1–5, prototype tracks P1–5, decision matrix, canonical test
   scenario. Directly authors tickets #25 and #26.
3. `docs/METADATA_SCHEMAS.md` — **CURRENT** (edited through PR #615). Canonical
   node/v3 schema reference: `Node`, `SealedChildRef`, `PublishedNode`,
   `NodeWriteBody`, `VaultKeyBlob v3`, generation clock, invariants, TS↔Rust parity.
4. `docs/adr/0001` + `0002` + `0003` — **CURRENT**. Authoritative freezes:
   write-revocation = full Ed25519 rotation; read-revocation protects future content
   only; AAD-bound node-seal byte encoding. Small but load-bearing; these supersede
   the older prose in SHARING.md/ARCHITECTURE.md.
5. `.planning/REQUIREMENTS.md` — **CURRENT**. REQ-ID catalog (CRYPTO/NODE/READ/ROT/
   WRITE/TEE/DATA/WEB/SDK-READ) with a full REQ→phase traceability table — the
   cleanest anchor from each part/flow spec to concrete requirements.
6. `.planning/research/ARCHITECTURE.md` — **CURRENT** (blueprint that has since
   shipped, verified). Deletion/rename/addition ledger for the v3 cutover + 8-phase
   build order — definitive for #27/#28 and grounding for all part specs.
7. `.planning/research/PITFALLS.md` — **CURRENT**. 20 line-cited silent-failure
   traps of the refactor; doubles as a regression/review checklist for v2 specs.
8. Phase CONTEXT docs 62/64/66/67/80 (see per-ticket sections) — **CURRENT**.
   Durable per-subsystem decision records (schema keystone, revocation soundness,
   API/DB cutover + publish CAS, TEE lease-renewer, write-plane + recipient pinning).

## Known-drifted — use with care

- `docs/ARCHITECTURE.md` — **PARTIAL/STALE**. Component map, module inventories, DB
  table map, TEE epoch mechanics still useful; but the encryption hierarchy and data
  flows describe the retired `rootFolderKey`/`FilePointer` model.
- `docs/SHARING.md` — **STALE** on model. Most complete end-to-end sharing flow
  reference, but its `share_keys`/lazy-folderKey-rotation revocation model is
  superseded by ADR 0001/0002 and the phase 63–66 work (`share_keys` is deleted).
- `docs/VAULT_EXPORT_FORMAT.md` — **PARTIAL**. ECIES envelope specifics (16-byte GCM
  nonce, binary layout) still accurate; the exported two-key model
  (`encryptedRootFolderKey` + `encryptedRootIpnsPrivateKey`) predates VaultKeyBlob v3.
- `.planning/codebase/*` (7 files) — **PARTIAL**. Analysis 2026-03, drift-reviewed
  2026-06-19. Layer topology, conventions, integrations, test infra still accurate;
  everything below the data-model boundary (folder/file modules, share flow, entity
  names) is pre-v3. STRUCTURE.md lacks `core/src/node/` and `sdk-core/src/rotation/`.
- `.planning/research/m4/*` — **PARKED** (2026-02-11). Productivity-suite milestone
  (billing/teams/editors/signing) never landed; doubly stale (built on the pre-v3
  model). Valuable only as the v-next feature menu (#5). STACK.md pins will bit-rot.
- `docs/METADATA_EVOLUTION_PROTOCOL.md` — **PARTIAL**. §5/§6.4 node/v3 version-lever
  rules current; §6.1 examples and §7 recovery matrix still cite v1 schemas.
- `docs/TESTING.md` / `.planning/codebase/TESTING.md` — **PARTIAL**. Infra and CI
  gate map good; vector/suite inventories predate the node-codec/node-aad KATs.
- `docs/GETTING-STARTED.md` — minor drift (Kubo v0.40.0 vs actual v0.42.0).
- `.planning/security/REVIEW-*` older than 2026-07 (~23 files) — historical; they
  review the OLD architecture the v2.0 rework structurally replaced.
- `.planning/adr/002-web3auth-mfa.md` — status "Proposed" but MFA shipped in M2;
  rationale only.
- Note an ADR numbering trap: the v2.0-critical ADRs are `docs/adr/0001–0003`;
  `.planning/adr/001–002` are an unrelated older pair (external-wallet derivation,
  MFA proposal).

## Cross-cutting anchors

These feed almost every ticket and are cited once here rather than repeated:

- `.planning/design/2026-06-26-sharing-read-keychaining-design.md` (**CURRENT**) —
  primary design source for #7–#11, #14, #16–#21, #25–#27.
- `.planning/design/2026-06-26-sharing-flows-walkthrough.md` (**CURRENT**) —
  step-by-step CLIENT/API/DB/IPFS/IPNS/TEE key/data traces for six flows, each step
  marked (existing) vs (new). Best raw material for flow specs #15–#21, #25.
- `.planning/REQUIREMENTS.md` (**CURRENT**) — REQ→phase traceability for all part/flow specs.
- `.planning/ROADMAP.md` (**CURRENT**, 2026-07-18) — phase decomposition 18–83 with
  goals/success criteria; the only enumeration of open closeout phases 74/80–83.
- `.planning/PROJECT.md` (**CURRENT**, updated Phase 77) — canonical overview, Key
  Decisions table (incl. the stated greenfield "node/v3 sole codec, no dual-codec
  migration bridge" stance), scope boundaries.
- `.planning/codebase/STACK.md` + `CONVENTIONS.md` (**STABLE**) — stack/version
  reference and coding conventions every part spec should encode.

## Per-ticket input artifacts

### #2 — Spec template / TOC

- `.planning/PROJECT.md` — **CURRENT** — canonical summary + Key Decisions table as
  a template model.
- `.planning/codebase/CONVENTIONS.md` — **STABLE** — conventions a spec template
  must encode (naming, string-literal unions, camelCase API vs snake_case DB,
  api-client regeneration).
- `.planning/codebase/TESTING.md` + `docs/TESTING.md` — **PARTIAL** — the
  test-strategy/coverage-gate section every spec needs (per-package gates, CI map).
- `docs/METADATA_EVOLUTION_PROTOCOL.md` — **PARTIAL** — evolution-rules section
  template (additive vs breaking, version levers, KAT lockstep).
- `.planning/research/SUMMARY.md` + `ARCHITECTURE.md` — **CURRENT** — model for how
  a milestone gets sliced into dependency-ordered phases; de-facto spec structure.
- Phase 61 CONTEXT (`.planning/phases/61-.../61-CONTEXT.md`) — **CURRENT** — the
  ADR-freeze + KAT-gate discipline as a reusable spec pattern.

### #5 — V-next feature set

- `.planning/research/m4/FEATURES.md` — **PARKED** — the feature menu: encrypted
  docs/sheets/slides, teams, billing rails, signing; competitor matrix
  (CryptPad/Tresorit/Proton), complexity estimates, MVP scoping. Primary feeder.
- `.planning/research/m4/SUMMARY.md`, `ARCHITECTURE.md`, `PITFALLS.md`, `STACK.md` —
  **PARKED** — pillar ordering (Billing→Teams→Editors→Signing), team-key hierarchy,
  ZK pitfalls (durable reasoning), stack picks (bit-rotting).
- `.planning/research/questions.md` — 2026-06-22 — speculative "agent-native ZK
  storage" milestone gating questions (ZK moat, x402, custody risk); v-next ideation.
- `.planning/STATE.md` Deferred Items table — **CURRENT** — ERC-1271, CRDT IPNS
  inbox, async search, alt MFA → v-next backlog seeds.
- `.planning/milestones/FEATURES.md` — 2026-03-30 — the shipped v1.0/v1.1 feature
  matrix (Web/Desktop/API/SDK/E2E): the existing surface v-next must preserve.
- `.planning/PROJECT.md` Out-of-Scope section — **CURRENT** — deferred items list.

### #6 — V1 migration stance

- `.planning/PROJECT.md` — **CURRENT** — the greenfield decision stated verbatim:
  node/v3 sole codec, no dual-codec/migration bridge.
- `.planning/research/ARCHITECTURE.md` — **CURRENT** — forward-only cutover +
  deletion/rename ledger; FK-rename migration mechanics.
- Phase 62 + 66 CONTEXT — **CURRENT** — explicit hard-cut rationale (62) and
  destructive drop-recreate DB cutover rationale (66).
- `docs/DATABASE_EVOLUTION_PROTOCOL.md` — **STABLE** — migration discipline,
  additive/destructive classification, historical incidents.
- `.planning/MILESTONES.md` + `.planning/milestones/v1.1-SCOPING_RATIONALE.md` —
  v1.1-era — migration precedent (vault blob v1→v2) and the DB-authority
  rationale v2.0 amends.

### #7 — Part spec: crypto package

- `docs/adr/0003-aad-bound-node-seal-encoding.md` — **CURRENT** — primary: the
  frozen 45-byte AAD layout, role/kind bytes, AEAD params, TS+Rust impl paths, KAT
  merge-gate discipline.
- Phase 61 CONTEXT + RESEARCH — **CURRENT** — the seal-primitive freeze in depth
  (fail-closed AAD validation, UUID→16-byte parity, KAT wave order).
- Phase 75 RESEARCH — **CURRENT** — cross-language parity end-state: strict
  RFC3339, ValidityType binding, canonical-only UUID; "shared JSON vector is the
  parity oracle" discipline.
- Phase 77 RESEARCH — **CURRENT** — standing conventions: single base64 codec in
  `@cipherbox/crypto`, error-path key zeroization, terminal-owner zeroization (D-09).
- `docs/VAULT_EXPORT_FORMAT.md` — **PARTIAL** — ECIES envelope details (eciesjs,
  non-standard 16-byte GCM nonce, ciphertext layout) still accurate.
- `.planning/adr/001-external-wallet-key-derivation.md` — **STABLE** — HKDF
  signature-derived keypair, low-S normalization.
- `.planning/security/LOW-SEVERITY-BACKLOG.md` — **PARTIAL** — open validation/
  zeroization items in crypto helpers (some cite retired paths).

### #8 — Part spec: core codecs

- Phase 62 CONTEXT — **CURRENT** — primary: the keystone `Node`/`SealedChildRef`/
  `PublishedNode` codecs, two sealed bodies, content self-seal, vault blob v3 byte
  layout, golden-vector freeze, the two SC#6 invariants.
- `docs/METADATA_SCHEMAS.md` — **CURRENT** — canonical schema reference + TS↔Rust
  parity pointers.
- `docs/adr/0003` — **CURRENT** — the AAD consumed by every codec seal.
- Phase 75 RESEARCH — **CURRENT** — node-codec KAT hardening (`fileIv`
  base64-decode-and-compare), codec parity gaps closed.
- `docs/METADATA_EVOLUTION_PROTOCOL.md` — **PARTIAL** — `schema:'node/v3'`
  discriminator, cross-language vector lockstep rules.

### #9 — Part spec: sdk-core

- Phase 63 CONTEXT — **CURRENT** — primary: read-chain navigation walk, O(1)
  read-grant issuance, scope-exit predicate, rotation engine skeleton + named seams,
  typed navigation results.
- Phase 64 CONTEXT — **CURRENT** — the four seams filled: content-key rotation,
  inner-grant re-mint, concurrent-add CAS merge, crash-resume convergence.
- Phase 80 CONTEXT — **CURRENT** — write-plane reconstruction on scope-exit,
  TS re-mint recipient-pin consumer, zeroization parity.
- `.planning/REQUIREMENTS.md` READ/ROT/SDK-READ rows — **CURRENT** — REQ anchors.
- `.planning/research/PITFALLS.md` — **CURRENT** — rotation-engine coverage traps
  (barrel exclusion), CRIT-1/M1/HIGH-3/HIGH-4.
- `.planning/security/REVIEW-80.md` — **CURRENT** — re-mint durability verdicts.

### #10 — Part spec: sdk

- Phase 63 CONTEXT (add-item/move/link-rewrite flows) + design doc §2.2/§5 —
  **CURRENT** — write-chain behavior.
- `.planning/REQUIREMENTS.md` WRITE rows — **CURRENT**.
- Phase 77 RESEARCH — **CURRENT** — retired dead share scaffolding
  (`ShareCallbacks`/`addShareKeysFn`), canonical field names.
- `.planning/codebase/ARCHITECTURE.md` — **PARTIAL** — layered-SDK topology and
  upload/download flows (pre-v3 detail).

### #11 — Part spec: api

- Phase 66 CONTEXT — **CURRENT** — primary: `share_keys` deletion, slimmed `shares`
  (encrypted read/write key refs, hard-DELETE revoke), `folder_ipns`→`ipns_records`,
  atomic publish CAS (seq-CAS + forward-only generation + tombstone, 409/410),
  resolve anti-rollback.
- Phase 71 CONTEXT — **CURRENT** — server-side authz/integrity: root-ownership gate,
  widen-only re-claim merge, CID-equivocation guard, D-10 terminology rename map
  (`readDescriptorRef`→`encryptedReadKey` etc.).
- `docs/DATABASE_EVOLUTION_PROTOCOL.md` — **STABLE** — migration rules + schema map.
- `.planning/REQUIREMENTS.md` DATA/TEE rows — **CURRENT**.
- `docs/ARCHITECTURE.md` NestJS module map + `docs/CONFIGURATION.md` env reference —
  **STABLE** at that granularity.
- `.planning/codebase/INTEGRATIONS.md` — **STABLE** (re-touched 2026-07-06) — DB/
  Redis/BullMQ/JWKS/throttle surface.
- `docs/CAPACITY.md` — **CURRENT** (2026-06-23) — measured perf and bottlenecks.

### #12 — Part spec: web

- `.planning/REQUIREMENTS.md` WEB-01..04 + SDK-READ rows — **CURRENT** — the anchor;
  web logic is deliberately thin (logic lives in sdk, D-06).
- `docs/DEVELOPMENT.md` — **CURRENT** — the D-06 test split (logic→sdk Vitest,
  UI→Playwright, web Vitest non-blocking).
- Phase 80 CONTEXT (web reconcile consumer) — **CURRENT**.
- `docs/ARCHITECTURE.md` Zustand store inventory + `.planning/codebase/`
  ARCHITECTURE/STRUCTURE hook inventories — **PARTIAL** — pre-v3 but structurally
  useful.
- `.planning/research/PITFALLS.md` — **CURRENT** — folderTree/Zustand desync,
  `.spec.ts` skip trap, generation high-water in client state.

### #13 — Part spec: desktop + fuse

- `docs/FILESYSTEM_SPECIFICATION.md` — **CURRENT** — mount backends (FUSE-T SMB /
  libfuse3 / WinFsp), naming/size/depth constraints, GCM vs CTR.
- Phase 80 CONTEXT — **CURRENT** — Rust rotation engine write-plane reconstruction,
  InodeTable interplay, replay.rs durability.
- `.planning/security/REVIEW-2026-07-11-phase74-rotation.md` +
  `REVIEW-2026-07-12-phase76.md` — **CURRENT** — Rust/FUSE rotation-revocation
  soundness; EOL rollback guard, zeroization, decrypt-and-resume recovery.
- `.planning/codebase/CONCERNS.md` — **PARTIAL** — still-live fragility list:
  FUSE-T SMB cache, WinFSP, vendored fuser, mutex unwraps.
- `docs/DEPLOYMENT.md` desktop packaging FUSE matrix — **STABLE**.
- Phase 75 RESEARCH — **CURRENT** — Rust as source-of-truth verifier for parity.

### #14 — Part spec: tee-worker

- Phase 67 CONTEXT — **CURRENT** — primary: the lease-renewer rewrite
  (verify-in-enclave: parse signedRecord, verify Ed25519 vs
  `publicKeyFromIpnsName`, re-sign same CID + same sequence, later EOL only;
  `ipns_records` as sole signing input; schedule collapse; refuse-stale +
  re-enroll).
- `.planning/REQUIREMENTS.md` TEE-01..07 rows — **CURRENT**.
- `.planning/security/REVIEW-2026-07-12-phase76.md` — **CURRENT** — TEE write-path
  hardening verdicts.
- `docs/DEPLOYMENT.md` + `docs/CONFIGURATION.md` — **STABLE** — Phala CVM vs
  simulator modes, "never recreate app_id", env surface.
- `docs/ARCHITECTURE.md` keyEpoch grace mechanics — **STABLE** for epoch handling.

### #15 — Flow spec: auth

- `docs/AUTHENTICATION_ARCHITECTURE.md` — **STABLE** (2026-03-23; auth is
  orthogonal to the v3 refactor) — primary: MPC Core Kit TSS factor model, pre/post
  MFA, threat analysis, cross-device approval.
- `.planning/adr/001-external-wallet-key-derivation.md` — **STABLE** — EIP-712
  signature-derived keypair path + known limitation.
- `.planning/adr/002-web3auth-mfa.md` — rationale only (MFA since shipped).
- `.planning/codebase/INTEGRATIONS.md` — **STABLE** — two-phase auth, token
  lifecycle, JWKS.
- Flows walkthrough Flow 1 (vault init) — **CURRENT** — where auth meets VaultKeyBlob v3.

### #16 — Flow spec: filesystem metadata + sync

- `docs/FILESYSTEM_SPECIFICATION.md` — **CURRENT** — primary (node/v3 storage-model
  section + platform constraints).
- `docs/METADATA_SCHEMAS.md` — **CURRENT** — the schemas being synced.
- Phase 62 + 63 CONTEXT — **CURRENT** — node create/add/move/link-rewrite semantics.
- Flows walkthrough (folder create / file upload traces) — **CURRENT**.
- `.planning/codebase/CONCERNS.md` — **PARTIAL** — 30s IPNS polling, pagination gaps.
- `docs/ARCHITECTURE.md` sync flows — **STALE** detail, structural reference only.

### #17 — Flow spec: content storage + versioning

- `docs/METADATA_SCHEMAS.md` (`NodeContent`, `VersionEntry`, encryptionMode
  GCM/CTR) — **CURRENT** — primary.
- `docs/FILESYSTEM_SPECIFICATION.md` — **CURRENT** — version count/cooldown rules,
  size/quota limits.
- Phase 62 CONTEXT (content self-seal) + Phase 64 CONTEXT (content re-key,
  `contentRekeyPending`) — **CURRENT**.
- Flows walkthrough file-upload/version trace — **CURRENT**.
- `docs/CAPACITY.md` — **CURRENT** — throughput/pin bottlenecks.

### #18 — Flow spec: sharing + grants

- Phase 63 CONTEXT — **CURRENT** — primary: O(1) read-grant issuance, scope-exit
  predicate, invite claim, "share of subtree ≡ share of file" insight.
- Phase 71 CONTEXT — **CURRENT** — invite/claim authz hardening, widen-only merge,
  grant-row schema + naming.
- `docs/adr/0001` + `0002` — **CURRENT** — revocation semantics.
- Grant-delivery research goals doc — **CURRENT** — the open questions layer.
- `docs/SHARING.md` — **STALE** model — richest end-to-end flow detail (UI entry
  points, API surface); read only with the ADR/phase-63 corrections in hand.
- `.planning/research/m4/PITFALLS.md` team-key reasoning — transferable background.

### #19 — Flow spec: key rotation

- Phase 64 CONTEXT — **CURRENT** — primary: revocation soundness, the CRITICAL
  re-seal-under-parent's-new-key fix, convergence/forward-only-generation invariant,
  crash-resume.
- Phase 80 CONTEXT — **CURRENT** — write-plane rotation + re-mint durability.
- `docs/adr/0001` + `0002` — **CURRENT** — what rotation must and must not protect.
- `.planning/REQUIREMENTS.md` ROT-01..07 — **CURRENT**.
- `.planning/security/REVIEW-80.md` + phase74 review — **CURRENT** — adversarial
  verdicts on the shipped engine.
- `.planning/research/PITFALLS.md` CRIT-1/M1/HIGH-3/HIGH-4 — **CURRENT**.

### #20 — Flow spec: republishing + liveness

- Phase 67 CONTEXT — **CURRENT** — primary: lease-renewer contract, EOL-renewal via
  atomic CAS, tombstone gate.
- `.planning/security/REVIEW-2026-07-12-phase76.md` — **CURRENT** — strictly-later-
  EOL rollback guard.
- `docs/CAPACITY.md` — **CURRENT** — republish batch sizing, IPNS PUT costs.
- `.planning/codebase/CONCERNS.md` + `INTEGRATIONS.md` — orphaned-record
  accumulation, delegated-routing fragility, TEE schedule integration.
- `docs/ARCHITECTURE.md` 6h-republish/keyEpoch description — **PARTIAL** — mechanics
  superseded by phase 67 but epoch grace still relevant.

### #21 — Flow spec: vault export / recovery

- `docs/VAULT_EXPORT_FORMAT.md` — **PARTIAL** — primary for the envelope mechanics;
  key model needs the VaultKeyBlob v3 update (two ECIES root keys).
- Phase 62 CONTEXT — **CURRENT** — vault recovery blob v3 byte layout
  (`0x03|u16|ECIES(readKey)|u16|ECIES(writeKey)`).
- `docs/METADATA_SCHEMAS.md` VaultKeyBlob v3 — **CURRENT**.
- `docs/METADATA_EVOLUTION_PROTOCOL.md` recovery-tool compatibility matrix —
  **PARTIAL** (recovery.html shipped v3 support in Phase 78, matrix may lag).
- `.planning/security/REVIEW-2026-07-12-phase76.md` decrypt-and-resume recovery —
  **CURRENT**.

### #22 — Flow spec: client-side search

- Thinnest corpus of all tickets. `.planning/milestones/FEATURES.md` — search rows
  in the shipped feature matrix; `docs/FILESYSTEM_SPECIFICATION.md` naming/duplicate
  rules; `.planning/codebase/TESTING.md` — web-E2E search suite exists. STATE.md
  defers async search to v-next. Expect to read code, not docs, for this spec.

### #23 — Design: decentralized IPNS resolution / publish

- Phase 66 CONTEXT — **CURRENT** — primary: the integrity-authoritative publish/
  resolve plane (atomic CAS, forward-only generation, anti-rollback case split).
- `.planning/milestones/v1.1-SCOPING_RATIONALE.md` — v1.1-era — the DB-first-vs-DHT
  resolve rationale and `folder_ipns`'s six roles: the prior decision a
  decentralized design must consciously amend.
- Phase 75 RESEARCH — **CURRENT** — IPNS record verification parity (Validity/EOL,
  RFC3339 strictness) any independent resolver must match.
- `.planning/codebase/INTEGRATIONS.md` — **STABLE** — Someguy + delegated-routing
  endpoints and fallback behavior.
- Grant-delivery research goals RQ6/P4 (decentralized inbox spike) — **CURRENT**.
- `docs/CAPACITY.md` — **CURRENT** — resolve/publish latency envelope.

### #24 — Design: record liveness / republisher

- Phase 67 CONTEXT — **CURRENT** — primary: verify-in-enclave lease-renewal is the
  trust-boundary baseline for any redesign.
- `.planning/security/REVIEW-2026-07-12-phase76.md` — **CURRENT** — EOL guard, CAS.
- `.planning/codebase/CONCERNS.md` — TEE single-provider (Phala) risk, republish
  window analysis.
- `.planning/milestones/v1.1-SCOPING_RATIONALE.md` — scheduling-role history.
- Grant-delivery research goals §12 — **CURRENT** — explicitly scopes TEE
  republishing out of the grant sprint, bounding this ticket's interface.

### #25 — Design: sharing + grant delivery

- `.planning/research/grant-delivery-rotation-research-goals.md` — **CURRENT** —
  THE input: grant-locus options (relay/metadata/inbox/hybrid), re-delivery on
  rotation, residual relay trust, decision matrix, prototype tracks, canonical
  Alice/folderA/b.txt scenario, exact code refs.
- Phase 80 CONTEXT (D-03 recipient-pubkey pinning + file-share carve-out D-03g) +
  `.planning/security/REVIEW-80.md` — **CURRENT** — the compromised-relay threat
  and its current mitigation + deferred pin-lifecycle items.
- Phase 71 CONTEXT — **CURRENT** — invite/claim surface and its hardening.
- `docs/SHARING.md` — **STALE** — legacy delivery flow as contrast material.
- Phase 74 security review — **CURRENT** — the re-mint trust finding pinning closed.

### #26 — Design: rotation model

- Phase 64 CONTEXT — **CURRENT** — primary: the soundness core and convergence
  invariant.
- Phase 80 CONTEXT — **CURRENT** — write-plane reconstruction rule + pin trust model.
- Grant-delivery research goals (hygiene-vs-revoking rotation split, rotation-scope
  experiment P5) — **CURRENT** — the open questions this ticket must answer.
- `docs/adr/0001` + `0002` — **CURRENT** — the ratified revocation semantics.
- Phase 63 CONTEXT (engine skeleton, web-as-rotation-host answer) — **CURRENT**.
- `.planning/security/REVIEW-80.md` + phase74 review — **CURRENT** — adversarial
  pressure-test record; deferred MEDIUMs are open design inputs.

### #27 — Design: v2 schemas + crypto envelope

- `docs/METADATA_SCHEMAS.md` — **CURRENT** — primary: the as-built v3 schema set.
- `docs/adr/0003` + Phase 61 CONTEXT — **CURRENT** — the frozen seal/AAD envelope.
- Phase 62 CONTEXT — **CURRENT** — codec keystone + invariants.
- `.planning/research/ARCHITECTURE.md` — **CURRENT** — `PublishedNode` envelope,
  key-unwrap walk, symbol ledger.
- `docs/METADATA_EVOLUTION_PROTOCOL.md` — **PARTIAL** — the version levers
  (`schema` discriminator, blob version byte, AAD domain separator) any v2-next
  schema must respect.
- Phase 80 CONTEXT D-03b — **CURRENT** — `NodeWriteBody` schema evolution +
  CBOR parity precedent.
- Phase 71 D-10 + Phase 77 — **CURRENT** — canonical field naming for envelope/grant
  fields.

### #28 — Design: component decomposition

- `.planning/research/ARCHITECTURE.md` 8-phase build order + seam recommendations —
  **CURRENT** — primary.
- `.planning/codebase/ARCHITECTURE.md` + `STRUCTURE.md` — **PARTIAL** — the layer
  topology, dependency rules, and annotated file map (add `node/` + `rotation/`
  mentally).
- `docs/ARCHITECTURE.md` component map — **PARTIAL** — apps/packages/crates
  inventory.
- `.planning/MILESTONES.md` — v1.1 — how the TS/Rust package split came to be.
- `docs/DEVELOPMENT.md` + `DEPLOYMENT.md` + `CONFIGURATION.md` — **STABLE** — the
  operational seams (per-app config, deploy units, test gates).
- `.planning/ROADMAP.md` — **CURRENT** — phase-by-phase package-boundary evidence.

## Gaps worth noting

- **No dedicated search-design artifact** (#22): client-side search has feature-matrix
  rows and E2E coverage but no design doc; that spec starts from code.
- **Flow-level auth walkthrough** (#15) is strong on Web3Auth internals but the
  token/vault bootstrap sequence lives mostly in the flows walkthrough + code.
- **The m4 set is the only #5 feeder** and is 5 months stale and parked — v-next
  scoping should treat it as a menu, not a plan.
- **Phase PATTERNS.md docs are point-in-time** implementation aids (line-numbered
  analog maps); low reference value now. CONTEXT.md docs are the durable layer.
- Phases 61–67 CONTEXT docs carry stale forward phase-number references (the
  "phase 69" era renumbering); Phase 80 CONTEXT is internally consistent.
