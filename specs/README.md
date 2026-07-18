# CipherBox as-built spec corpus

Concrete specs of the CipherBox v1 system as implemented in
[FSM1/cipher-box](https://github.com/FSM1/cipher-box) — flows, data structures, and
invariants, not implementation. Every spec follows [TEMPLATE.md](TEMPLATE.md). The corpus
is the factual input to the v2 blueprint; wayfinder map:
[cipher-box-next#1](https://github.com/FSM1/cipher-box-next/issues/1).

## Parts

| Spec | Covers | Ticket |
| --- | --- | --- |
| [parts/crypto.md](parts/crypto.md) | `packages/crypto` + `crates/crypto` — primitives, ECIES wrapping, AES-256-GCM envelopes, zeroization | #7 |
| [parts/core-codecs.md](parts/core-codecs.md) | `packages/core` + `crates/core` — node codecs, read/write-plane structures, CBOR strictness, shared vectors | #8 |
| [parts/sdk-core.md](parts/sdk-core.md) | `packages/sdk-core` — folder tree, registration, rotation engine, share/recipient-pins, IPNS record construction | #9 |
| [parts/sdk.md](parts/sdk.md) | `packages/sdk` + `crates/sdk` — client orchestration, owner-reconcile, shared-write; TS/Rust divergences | #10 |
| [parts/api.md](parts/api.md) | `apps/api` — wire surface, DB schema, auth tokens, IPNS relay + CAS gating, pinning/quota, TEE endpoints | #11 |
| [parts/web.md](parts/web.md) | `apps/web` — views, state split, upload/download, sharing UI, polling | #12 |
| [parts/desktop.md](parts/desktop.md) | `apps/desktop` + `crates/fuse` — Tauri commands, FUSE inode/journal/replay, scope-exit rotation, platforms | #13 |
| [parts/tee-worker.md](parts/tee-worker.md) | `apps/tee-worker` — republish loop, key epochs + grace, CVM vs simulator | #14 |

## Flows

| Spec | Covers | Ticket |
| --- | --- | --- |
| [flows/auth.md](flows/auth.md) | Web3Auth login → key derivation → token exchange → session; key-material lifecycle | #15 |
| [flows/metadata-sync.md](flows/metadata-sync.md) | Folder metadata end-to-end; IPNS read plane vs write plane; CAS/sequence; polling sync | #16 |
| [flows/content-storage.md](flows/content-storage.md) | Upload/download pipeline, content envelope, CID lifecycle, versioning, quota | #17 |
| [flows/sharing-grants.md](flows/sharing-grants.md) | User-to-user and link sharing, shared-write, grant mint/re-mint, recipient pins, revocation | #18 |
| [flows/rotation.md](flows/rotation.md) | keyEpoch model, rotation cascade, write-plane rotation, scope-exit rotation, fail-closed gates | #19 |
| [flows/republish-liveness.md](flows/republish-liveness.md) | TEE republish end-to-end, record sourcing, epochs/grace, lapse failure modes | #20 |
| [flows/vault-export-recovery.md](flows/vault-export-recovery.md) | Vault export format, import, recovery tooling, server-independent recovery | #21 |
| [flows/search.md](flows/search.md) | Client-side index build/update/query, privacy properties | #22 |

## Conventions

- One spec per file; specs link, never restate, each other's structures.
- `Verified against` pins the cipher-box commit a spec was checked at; bump it when re-verifying.
- Known gaps stay in the spec they belong to — the corpus records what IS, drift included.
- Rewrite opinions live only in each spec's **Rewrite notes** section; everything above it is fact.
