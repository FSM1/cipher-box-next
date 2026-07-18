# Crypto primitives (`@cipherbox/crypto` + `cipherbox-crypto`)

| | |
| --- | --- |
| **Kind** | part |
| **Sources** | `packages/crypto/src/` (aes/, ecies/, ed25519/, ipns/, keys/, vault/, device/, utils/, constants.ts, types.ts, index.ts, tsup.config.ts, package.json), `crates/crypto/src/` (aes.rs, aes_ctr.rs, ecies.rs, ed25519.rs, hkdf.rs, ipns_name.rs, utils.rs, error.rs, lib.rs, Cargo.toml), `crates/crypto/tests/cross_language.rs`, `tests/vectors/crypto/` (node-aad.json, uuid-acceptance.json, aes-gcm.json, ecies.json, ed25519.json, hkdf.json, ipns-name.json), `scripts/check-vector-parity.sh`, `scripts/generate-test-vectors.ts`, `docs/adr/0003-aad-bound-node-seal-encoding.md`, `docs/VAULT_EXPORT_FORMAT.md` §3–4, `.planning/adr/001-external-wallet-key-derivation.md`, `.planning/phases/61-*/61-CONTEXT.md`, `.planning/phases/75-*/75-RESEARCH.md`, `.planning/phases/77-*/77-RESEARCH.md`, `.planning/security/LOW-SEVERITY-BACKLOG.md` |
| **Verified against** | cipher-box `27c4abec5` |
| **Status** | draft |

## Purpose and scope

The crypto layer is a **twin pair**: `@cipherbox/crypto` (TypeScript, consumed by web,
SDK, API, TEE worker) and `cipherbox-crypto` (Rust, consumed by the desktop/FUSE stack).
Both provide pure cryptographic primitives with no CipherBox domain knowledge: AES-256-GCM
content/metadata encryption (with and without AAD), AES-256-CTR streaming encryption,
ECIES key wrapping over secp256k1 (eciesjs-compatible), Ed25519 keygen/sign/verify,
HKDF-SHA256 deterministic IPNS-keypair derivation, IPNS name derivation, byte codecs
(hex, base64, UUID→16 bytes), secure randomness, and best-effort zeroization helpers.
The Rust crate's header states the contract plainly: "Mirrors @cipherbox/crypto
TypeScript package" (`crates/crypto/src/lib.rs:4`).

This spec owns: every primitive's parameters and byte encodings, the frozen AAD-bound
node-seal encoding (ADR 0003), the ECIES envelope, the key-material
lifecycle/zeroization discipline (D-09 terminal-owner rule), and the cross-language
parity strategy (shared JSON KATs). It does **not** own the `Node`/`SealedChildRef`
schemas or the codec layer that *calls* the seal primitive
([../parts/core-codecs.md](../parts/core-codecs.md)), the DB/REST surfaces that carry
ciphertexts ([../parts/api.md](../parts/api.md)), IPNS *record* verification internals on
the Rust side (they live in `crates/core`/`crates/api-client`, covered by
[../parts/desktop.md](desktop.md) and
[../flows/metadata-sync.md](../flows/metadata-sync.md)), or how the TEE uses ECIES for
republishing ([../flows/republish-liveness.md](../flows/republish-liveness.md)).

## Vocabulary

- **Seal / sealed blob** — the high-level AEAD API output: `[IV (12 bytes)][ciphertext ‖
  16-byte GCM tag]`. "Seal" mints a fresh IV; "encrypt" is the lower-level fixed-IV form.
- **AAD-bound node seal** — `sealAesGcmAad`/`seal_aes_gcm_aad` with an AAD produced by
  `buildNodeAad`/`build_node_aad`, binding a blob to a node identity (ADR 0003).
- **`nodeId`** — canonical hyphenated RFC-4122 UUID string; encoded into the AAD as 16
  raw bytes in field order (never UTF-8 of the string).
- **`kind` byte** — `0x01` folder, `0x02` file, `0x03` root.
- **`role` byte** — `0x01` body, `0x02` child-readkey, `0x03` content, `0x04`
  child-writekey.
- **`generation`** — key-rotation counter, encoded as big-endian u32 in the AAD; the
  generation *clock* is owned by [../flows/rotation.md](../flows/rotation.md).
- **Wrap / unwrap** — ECIES encryption of key material to a secp256k1 `publicKey`
  (`wrapKey`/`unwrapKey`; the implementation borrows the terms from key wrapping even
  though ECIES can carry arbitrary bytes).
- **KAT** — Known-Answer Test: a committed JSON vector under `tests/vectors/` asserted by
  a test in one or both languages; the parity contract between the twins.
- **D-09 / terminal owner** — the zeroization convention (phase 77): exactly one owner
  zeroes a sensitive buffer; a callee that borrows a caller's buffer must never zero it.
- **`ipnsName`** — CIDv1 libp2p-key base36 (`k51…`) encoding of an Ed25519 public key.

## Actors and trust boundaries

The crypto layer performs no I/O and holds no state; the trust story is about which
*host runtime* invokes which primitives with plaintext key material in memory.

| Actor | Uses from this layer | Plaintext key material it sees here | Must never see |
| --- | --- | --- | --- |
| Web client (`apps/web`, `packages/sdk*`, `packages/core`) | everything in the TS package | user `privateKey` (secp256k1, from Web3Auth or ADR-001 wallet derivation), read/write keys, fileKeys, `ipnsPrivateKey`s | TEE epoch private keys |
| Desktop/FUSE (`crates/core`, `crates/sdk`, `crates/fuse`) | everything in the Rust crate | same as web client | same |
| CipherBox API (`apps/api`) | verification-only surface: `publicKeyFromIpnsName`, `verifyIpnsRecordSignature*`, `parseIpnsRecord`, codecs | none — signature verification uses public keys recovered from names | any private key, any content key |
| TEE worker (`apps/tee-worker`) | `unwrapKey`/`wrapKey`, `deriveEd25519PublicKey`, sign/verify, codecs | `ipnsPrivateKey` transiently in-enclave ([../flows/republish-liveness.md](../flows/republish-liveness.md)) | user secp256k1 keys |
| Recovery tool (`recovery.html`) | reimplements the ECIES envelope from `docs/VAULT_EXPORT_FORMAT.md` §3–4 | user `privateKey` locally | — |

Boundary rules enforced *inside* this layer: error messages are generic to avoid
decryption/padding oracles (every failure collapses to `'Encryption failed'` /
`'Decryption failed'` / `'Key unwrapping failed'` with an internal `code`,
`packages/crypto/src/aes/decrypt.ts:60-64`); keys are `Uint8Array`/`[u8; N]`, never
strings; nothing here logs or persists key material.

## Data structures

All structures here are byte layouts and in-memory types; nothing this part owns is
persisted directly (its outputs are embedded into structures owned by
[../parts/core-codecs.md](../parts/core-codecs.md) and rows owned by
[../parts/api.md](../parts/api.md)). There is consequently no write discipline —
everything is immutable bytes or memory-only.

### 45-byte node-seal AAD (frozen, ADR 0003)

Produced by `buildNodeAad(nodeId, kind, generation, role)`
(`packages/crypto/src/aes/seal.ts:80`) / `build_node_aad`
(`crates/crypto/src/aes.rs:185`). Byte-identical across languages, pinned by
`tests/vectors/crypto/node-aad.json`.

| Offset | Length | Field | Encoding |
| --- | --- | --- | --- |
| 0 | 22 | domain | UTF-8 `"cipherbox/node-seal/v1"` |
| 22 | 1 | separator | `0x00` (prevents length-extension collisions between domain versions) |
| 23 | 16 | `nodeId` | raw RFC-4122 UUID bytes, field order — a hex-field parse, never `TextEncoder` |
| 39 | 1 | `kind` | `0x01`/`0x02`/`0x03` |
| 40 | 4 | `generation` | big-endian u32 |
| 44 | 1 | `role` | `0x01`–`0x04` |

Fail-closed validation (D-03): out-of-range `kind`/`role`, non-integer or
out-of-`[0, 2^32-1]` `generation`, or a non-canonical UUID throw
`CryptoError('…', 'INVALID_AAD_INPUT')` / `Err(CryptoError::InvalidAadInput)` before any
bytes are produced. UUID acceptance is **canonical-only** (Option A, phase 75): exactly
the 8-4-4-4-12 hyphenated shape, hex case-insensitive. TS checks a regex against the
*raw* input plus a load-bearing `length !== 36` guard — JS `$` without the `m` flag also
matches before a trailing `\n`, so without the guard
`"…440000\n"` would be accepted by TS and rejected by Rust
(`packages/crypto/src/utils/encoding.ts:70-79`). Rust runs a dependency-free
byte-position check `is_canonical_uuid_form` *before* `Uuid::parse_str`, because the
`uuid` crate alone also accepts simple-32-hex, braced, and `urn:uuid:` forms
(`crates/crypto/src/aes.rs:151-173`). The accept/reject domain is locked by the shared
oracle `tests/vectors/crypto/uuid-acceptance.json` (12 cases, including the
trailing-newline case), asserted on both sides
(`packages/crypto/src/__tests__/build-node-aad.test.ts:99`,
`crates/crypto/tests/cross_language.rs:389`).

### Sealed blob (GCM, with or without AAD)

`[IV (12 bytes)][ciphertext][GCM tag (16 bytes)]` — the tag is appended to the
ciphertext by both runtimes (Web Crypto behavior, mirrored by the `aes-gcm` crate).
Minimum size 28 bytes; anything shorter is rejected before decryption
(`packages/crypto/src/aes/seal.ts:27,176`, `crates/crypto/src/aes.rs:28,141`).

| Parameter | Value |
| --- | --- |
| Algorithm | AES-256-GCM (Web Crypto `AES-GCM` / `aes_gcm::Aes256Gcm`) |
| Key | 32 bytes exactly |
| IV | 12 bytes, fresh random per seal; the seal API accepts **no caller-supplied IV** |
| Tag | 16 bytes, appended |
| AAD | absent (`sealAesGcm`) or the 45-byte node AAD (`sealAesGcmAad`), bound via `additionalData` / `Payload { msg, aad }` |

The lower-level `encryptAesGcm[Aad]`/`decryptAesGcm[Aad]` take an explicit IV (needed
for KATs and for callers that store IV separately); only the seal wrappers mint IVs.

### ECIES envelope (eciesjs v0.4.x default scheme)

Produced by `wrapKey` (TS: `eciesjs` `encrypt`, `packages/crypto/src/ecies/encrypt.ts`)
and `wrap_key` (Rust: `ecies` crate, same author, cross-compatible —
`crates/crypto/src/ecies.rs:1-4`). Layout, per `docs/VAULT_EXPORT_FORMAT.md` §4 (this
section of that doc is still accurate; its surrounding key *model* is not — see Known
gaps):

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 65 | ephemeral secp256k1 public key, uncompressed (`0x04 ‖ x ‖ y`) |
| 65 | 16 | AES-256-GCM nonce — **16 bytes, NOT the standard 12** (eciesjs default) |
| 81 | 16 | GCM auth tag |
| 97 | N | ciphertext (same length as plaintext) |

Internals (fixed by the library pair, documented for reimplementers): ECDH over
secp256k1 → `ikm = ephemeralPublicKey(65) ‖ sharedPoint(65)` → HKDF-SHA256 with **no
salt, no info** → 32-byte AES key → AES-256-GCM with the 16-byte nonce. Total overhead
97 bytes; a wrapped 32-byte key is 129 bytes (258 hex chars). ECIES is
non-deterministic (fresh ephemeral key per call), so the cross-language KAT
(`tests/vectors/crypto/ecies.json`) pins *unwrap of a TS-produced ciphertext* plus a
round-trip, not exact ciphertext bytes (`crates/crypto/tests/cross_language.rs:145`).

Validation before wrapping: length == 65 and `0x04` prefix on both sides; TS
additionally verifies the key is a valid curve point via
`ProjectivePoint.fromHex` (`packages/crypto/src/ecies/encrypt.ts:42-45`) while Rust
delegates point validation to the `ecies` crate — both fail closed, with different
error granularity. Unwrap validates private key length == 32 and ciphertext length ≥ 81
(`ECIES_MIN_CIPHERTEXT_SIZE = 65 + 16` — note this is the *pre-nonce* minimum, smaller
than any real envelope).

### AES-256-CTR counter block

16-byte IV = 8-byte random nonce ‖ 8-byte big-endian counter starting at 0
(`packages/crypto/src/utils/random.ts:68`, `generateCtrIv`). The counter width is the
parity-critical choice: Web Crypto is called with `length: 64`
(`AES_CTR_LENGTH`, `packages/crypto/src/constants.ts:44`) and Rust uses
`ctr::Ctr64BE<Aes256>` (`crates/crypto/src/aes_ctr.rs:26`) — a 64-bit big-endian
counter on both sides. CTR output is plaintext-sized; **no authentication** — integrity
is delegated to IPFS content addressing (both files' header comments say so
explicitly). Random-access range decryption computes
`counter = baseCounter + floor(startByte/16)` and decrypts only the block-aligned span
(`packages/crypto/src/aes/decrypt-ctr.ts:82`, `crates/crypto/src/aes_ctr.rs:71`).

### HKDF-derived IPNS keypair derivations

Common path: secp256k1 `privateKey` (32 bytes) → HKDF-SHA256(salt = `"CipherBox-v1"`,
info = per-domain string) → 32-byte Ed25519 seed → keypair → `ipnsName`. The frozen
info-string catalog, and where each derivation lives today:

| Info string | TS location | Rust location (`crates/crypto/src/hkdf.rs`) | KAT case in `hkdf.json` |
| --- | --- | --- | --- |
| `cipherbox-vault-ipns-v1` | `packages/crypto/src/vault/derive-ipns.ts:44` | `derive_vault_ipns_keypair` | yes |
| `cipherbox-vault-key-ipns-v1` | `derive-ipns.ts:89` | `derive_vault_key_ipns_keypair` | yes |
| `cipherbox-byo-ipfs-config-v1` | `derive-ipns.ts:128` | **absent** | no |
| `cipherbox-vault-settings-v1` | `derive-ipns.ts:167` | `derive_vault_settings_ipns_keypair` | yes |
| `cipherbox-device-registry-ipns-v1` | `packages/core/src/registry/derive-ipns.ts` (moved out of crypto) | `derive_registry_ipns_keypair` | yes |
| `cipherbox-recycle-bin-ipns-v1` | `packages/core/src/bin/derive-ipns.ts` (moved out of crypto) | `derive_bin_ipns_keypair` | yes |
| `cipherbox-file-ipns-v1:<fileId>` | **no production TS implementation remains** (v1/v2-era per-file derivation; v3 mints random file keypairs) | `derive_file_ipns_keypair` (min `fileId` length 10) | yes |

The general HKDF primitive is `deriveKey({ inputKey, salt, info, outputLength=32 })`
(`packages/crypto/src/keys/derive.ts:42`, Web Crypto `deriveBits`); Rust inlines
`Hkdf::<Sha256>` in `derive_ipns_keypair` (`hkdf.rs:50`). A separate, unrelated HKDF
use — the ADR-001 external-wallet keypair (salt `'CipherBox-ECIES-v1'`, info =
lowercased wallet address, input = low-S-normalized EIP-712 signature) — lives in
`apps/web/src/lib/crypto/signatureKeyDerivation.ts`, *not* in this package; this layer
only defines the `VaultKey` type it produces
(`.planning/adr/001-external-wallet-key-derivation.md`,
[../flows/auth.md](../flows/auth.md)).

### Ed25519 keys and signatures

32-byte private key **seed**, 32-byte public key, 64-byte deterministic RFC-8032
signature. TS: `@noble/ed25519` with `sha512Sync` wired at module load
(`packages/crypto/src/ed25519/keygen.ts:15`). Rust: `ed25519-dalek`
(`crates/crypto/src/ed25519.rs`). Determinism is what makes the cross-language
signature KAT possible (`tests/vectors/crypto/ed25519.json`). Note the vault-export doc
describes a 64-byte `seed ‖ publicKey` "libp2p format" — this layer deals exclusively in
32-byte seeds; the 64-byte form is a legacy export-format concern
([../flows/vault-export-recovery.md](../flows/vault-export-recovery.md)).

### `ipnsName` encoding

CIDv1(version 1, codec `0x72` libp2p-key, identity multihash over the libp2p
`PublicKey` protobuf `{Type: Ed25519(1), Data: 32 bytes}`), rendered base36 with `k`
prefix. TS is library-backed (`@libp2p/peer-id` + `multiformats`,
`packages/crypto/src/ipns/derive-name.ts:27`); Rust **hand-rolls** the protobuf, varint,
and base36 encoding (`crates/crypto/src/ipns_name.rs:13-117`) — parity pinned by
`tests/vectors/crypto/ipns-name.json`. The inverse, `publicKeyFromIpnsName`
(`derive-name.ts:67`), recovers the raw 32-byte key from the name and is the
authoritative key source everywhere (never a DB column); it exists in TS only — the
Rust inverse lives in `crates/core`.

### In-memory types and error surface

- `VaultKey { publicKey: 65B, privateKey: 32B }` — source-agnostic (Web3Auth MPC or
  ADR-001 derived), `packages/crypto/src/types.ts:11`.
- `Ed25519Keypair`, `DeviceKeypair { publicKey, privateKey, deviceId = hex(SHA-256(publicKey)) }`
  (`packages/crypto/src/device/`) — TS only; desktop device identity does not use the crate.
- `EncryptedData { ciphertext, iv }` — the split form for callers that store IV
  separately.
- `CryptoError` with a string-literal `CryptoErrorCode` union of 16 codes
  (`types.ts:33-49`); messages stay generic, codes are for internal branching. Rust:
  `enum CryptoError` via `thiserror` (`crates/crypto/src/error.rs`) — the two error
  vocabularies are *not* aligned name-for-name and are not part of any parity contract.

## Interface

Capability-level surface (all TS exports flow through `packages/crypto/src/index.ts`;
Rust through `crates/crypto/src/lib.rs`):

| Capability | TS | Rust |
| --- | --- | --- |
| AAD builder for node seals | `buildNodeAad` | `build_node_aad` |
| AAD-bound seal/unseal | `sealAesGcmAad` / `unsealAesGcmAad` (+ fixed-IV `encryptAesGcmAad`/`decryptAesGcmAad`) | `seal_aes_gcm_aad` / `unseal_aes_gcm_aad` (+ fixed-IV pair) |
| Plain GCM seal/unseal (legacy, non-node uses) | `sealAesGcm` / `unsealAesGcm` / `encryptAesGcm` / `decryptAesGcm` | `seal_aes_gcm` / `unseal_aes_gcm` / … |
| CTR streaming + range decrypt | `encryptAesCtr` / `decryptAesCtr` / `decryptAesCtrRange` | `encrypt_aes_ctr` / `decrypt_aes_ctr` / `decrypt_aes_ctr_range` (range fn not re-exported at crate root) |
| ECIES wrap/unwrap | `wrapKey` / `unwrapKey` / `reWrapKey` (unwrap-then-rewrap, zeroes the intermediate) | `wrap_key` / `unwrap_key` (returns `Zeroizing<Vec<u8>>`); no rewrap |
| Ed25519 | `generateEd25519Keypair`, `deriveEd25519PublicKey`, `signEd25519`, `verifyEd25519` | `generate_ed25519_keypair`, `get_public_key`, `sign_ed25519`, `verify_ed25519` |
| Deterministic IPNS keypairs | vault / vault-key / byo-config / vault-settings (`deriveVault…IpnsKeypair`) | vault / vault-key / registry / bin / file / vault-settings |
| IPNS name codec | `deriveIpnsName`, `publicKeyFromIpnsName` | `derive_ipns_name` (forward only) |
| IPNS record verify/parse | `verifyIpnsRecordSignature[Detailed]` (verdicts `valid`/`expired`/`invalid`), `parseIpnsRecord` — backed by the `ipns` npm package | **not in this crate** (lives in `crates/core::ipns` / `crates/api-client`) |
| Device identity | `generateDeviceKeypair`, `deriveDeviceId` | — |
| Codecs | `hexToBytes`/`bytesToHex`, `bytesToBase64`/`base64ToBytes` (canonical repo-wide base64 since phase 77, chunked at 32768), `uuidToBytes`, `concatBytes` | `hex_to_bytes`/`bytes_to_hex` (via `hex` crate); **no base64** (Rust consumers use the `base64` crate directly) |
| Randomness | `generateRandomBytes` (throws `SECURE_CONTEXT_REQUIRED` outside secure contexts), `generateFileKey`, `generateIv`, `generateCtrIv` | `generate_random_bytes`, `generate_file_key`, `generate_iv` (all `OsRng`); no CTR-IV helper |
| Zeroization helpers | `clearBytes`, `clearAll` (null-safe `.fill(0)`) | `clear_bytes` (`zeroize`) |
| Misc (Rust-only, non-crypto) | — | `generate_uuid_v4` (canonical lowercase hyphenated string), `mime_from_extension` |
| Legacy key derivation (dead) | `deriveKey`, `deriveContextKey`, `generateFolderKey` — zero production consumers outside the package | — |

Consumer map for the load-bearing primitive: `sealAesGcmAad`/`buildNodeAad` are called
in production only from `packages/core/src/node/{seal,decode,types}.ts` and
`packages/sdk-core/src/folder/registration.ts`; Rust from `crates/core/src/node/seal.rs`
and `crates/sdk/src/rotation/engine.rs`. The non-AAD `sealAesGcm` pair has **no**
production consumer in either language (only a desktop test-vector generator script) —
kept deliberately per D-00b for non-node uses.

## Dependencies

- **TS:** `eciesjs ^0.4.16` (ECIES scheme), `@noble/ed25519` + `@noble/hashes`
  (Ed25519, SHA-256/512), `@libp2p/crypto` + `@libp2p/peer-id` + `multiformats` (IPNS
  name codec), `ipns ^10.1.3` (record validate/unmarshal); Web Crypto for
  AES/HKDF/randomness. `@noble/secp256k1` is imported at runtime by
  `ecies/encrypt.ts` but declared only in `devDependencies` — it works because tsup
  bundles it (`noExternal: [/.*/]` for CJS; devDeps are bundled in the ESM build too),
  but it is a latent packaging trap (`packages/crypto/package.json:33-39`,
  `tsup.config.ts`).
- **Rust:** `aes-gcm`, `aes` + `ctr`, `ecies` (eciesjs-compatible, same author),
  `ed25519-dalek`, `hkdf` + `sha2`, `rand` (`OsRng`), `zeroize`, `uuid` (1.20.0),
  `hex`, `thiserror` (`crates/crypto/Cargo.toml`).
- This part depends on **no other CipherBox part**; everything above it depends on it.

## Behaviors

### Build node AAD and seal a blob

- **Trigger** — a codec seals a node body, child key, or content
  ([../parts/core-codecs.md](../parts/core-codecs.md) owns *what* is sealed under which
  role; role assignment lives there, not here).
- **Preconditions** — caller holds a 32-byte key and the node's identity tuple
  (`nodeId`, `kind`, `generation`, `role`).
- **Steps**
  1. `buildNodeAad` validates fail-closed (kind/role/generation/canonical UUID) and
     emits the 45-byte AAD.
  2. `sealAesGcmAad` validates key length, mints a fresh random 12-byte IV
     (caller-supplied IVs are impossible at this level — D-00a), and encrypts with the
     AAD bound into the tag via Web Crypto `additionalData` / `aes-gcm`
     `Payload { msg, aad }`.
  3. Output is `[IV][ct‖tag]`, IV-first (pinned by the envelope-order assertion in
     `cross_language.rs:340-353`).
- **Postconditions** — the blob can be opened only with the same key **and** a
  byte-identical AAD; any change to `nodeId`, `kind`, `generation`, `role`, or the
  domain string is an authentication failure (transplant resistance, the D-02 negative
  matrix in both test suites).
- **Failure modes** — invalid AAD input → `INVALID_AAD_INPUT` before any crypto; wrong
  key length → `INVALID_KEY_SIZE`; any AEAD failure on unseal (wrong key, wrong AAD,
  tampered/truncated blob) → one generic `DECRYPTION_FAILED` with no cause
  discrimination (oracle prevention).

### Encrypt content — GCM vs CTR

- **Trigger** — file upload/download paths choose the mode
  ([../flows/content-storage.md](../flows/content-storage.md)); GCM for ordinary files,
  CTR for streamable media needing byte-range seeks.
- **Steps (CTR range decrypt, the non-obvious one)**
  1. Validate key (32B) and IV (16B); TS also rejects negative offsets, Rust's `usize`
     makes them unrepresentable; `start > end` is an error on both sides (TS
     `INVALID_INPUT`, Rust `InvalidRange`).
  2. Clamp `endByte` to the data; a fully out-of-range request returns an **empty
     buffer, not an error**, in both languages.
  3. Compute the block-aligned window, build a counter block =
     `nonce(8) ‖ BE64(baseCounter + startBlock)` (Rust uses `wrapping_add`), decrypt the
     window, slice out the exact requested bytes.
- **Postconditions** — CTR output is unauthenticated; the caller's integrity check is
  the IPFS CID of the ciphertext.
- **Failure modes** — none are silent; but note GCM decryption of CTR data (or vice
  versa) is only caught by the GCM tag, never by CTR.

### Wrap, unwrap, and re-wrap keys (ECIES)

- **Trigger** — grant issuance and vault blobs
  ([../flows/sharing-grants.md](../flows/sharing-grants.md),
  [../flows/vault-export-recovery.md](../flows/vault-export-recovery.md)), TEE
  enrollment (`wrapIpnsKeyForTee` in sdk-core is a thin bytes-in/bytes-out wrapper over
  `wrapKey` since phase 77;
  [../flows/republish-liveness.md](../flows/republish-liveness.md)).
- **Steps** — validate recipient key (65B, `0x04`, on-curve in TS), delegate to the
  library; each call produces a distinct ciphertext (fresh ephemeral key). `reWrapKey`
  (TS only) = unwrap with the owner's private key → wrap to the recipient's public key
  → `.fill(0)` the intermediate plaintext on both success and failure paths
  (`packages/crypto/src/ecies/rewrap.ts:33-54`).
- **Postconditions** — TS `unwrapKey` returns a plain `Uint8Array` the **caller** owns
  (and must eventually `clearBytes`); Rust `unwrap_key` returns
  `Zeroizing<Vec<u8>>` that self-zeroes on drop — same ownership rule, enforced by
  convention in TS and by type in Rust.
- **Failure modes** — all collapse to `KEY_WRAPPING_FAILED` / `KEY_UNWRAPPING_FAILED`
  (`EciesWrappingFailed`/`EciesUnwrappingFailed`), deliberately not distinguishing
  wrong-key from tampered-ciphertext.

### Derive a deterministic IPNS keypair

- **Trigger** — vault bootstrap, settings/BYO-config storage, registry/bin (see the
  catalog table above for which side implements which).
- **Steps** — validate the input secp256k1 key is 32 bytes → HKDF-SHA256 with the
  frozen salt/info → Ed25519 seed → public key → `ipnsName`. Same inputs always yield
  the same name (self-sovereign discovery from the user key alone). Rust zeroes the HKDF
  output (`okm.zeroize()`) after constructing the `SigningKey` and returns the private
  key as `Zeroizing`; TS returns plain buffers the caller owns.
- **Postconditions** — domain separation guarantees different info strings never
  collide for the same user key (asserted in both test suites).
- **Failure modes** — wrong input-key length → `INVALID_KEY_SIZE` /
  `InvalidPrivateKey`; Rust `derive_file_ipns_keypair` additionally rejects
  `fileId.len() < 10` (`InvalidFileId`) — a validation mirroring a TS function that no
  longer exists.

### Verify an IPNS record signature (TS only)

- **Trigger** — API resolve path and SDK strict resolve
  ([../flows/republish-liveness.md](../flows/republish-liveness.md),
  [../flows/metadata-sync.md](../flows/metadata-sync.md)).
- **Steps** — recover the Ed25519 public key from the `ipnsName` itself (CID parse →
  `peerIdFromCID` → inline key), then run the `ipns` package validator over the
  marshaled record. The detailed variant maps outcomes to a three-way verdict:
  `valid` (signature + within validity window), `expired` (authentic but past EOL —
  distinguished because `RecordExpiredError` is thrown only *after* the signature
  check passes), `invalid` (everything else)
  (`packages/crypto/src/ipns/verify-record.ts:47-82`). The boolean variant is strict:
  only `valid` → `true`. `parseIpnsRecord` extracts
  `{value, sequence, signatureV2, data, pubKey?, validity: Date}` via the same library
  so the wire format stays in lockstep with record creation.
- **Failure modes** — verify never throws (returns a verdict/false); an RSA-style
  non-inline key in the name is `invalid` by policy (CipherBox is Ed25519-only). The
  Rust twin of this behavior — including the strict RFC3339 Validity parser and
  ValidityType binding, phase 75 — lives in `crates/core`/`crates/api-client`, outside
  this crate; the crypto crate itself only signs/verifies raw Ed25519.

### Key-material lifecycle and zeroization (D-09)

The single rule: **the terminal owner zeroes; a callee must never zero a caller-owned
buffer.** Getting this backwards once broke 48/89 E2E tests (project memory), which is
why the convention is now documented at every relevant function. Concretely, in this
layer:

1. Functions that *borrow* a key (`encryptAesGcm*`, `decryptAesGcm*`, `wrapKey`,
   `signEd25519`, `deriveKey`, …) never touch the caller's buffer. Where an internal
   copy is required (Web Crypto needs a plain `ArrayBuffer`), the copy is made and
   zeroed by `importAesKey` — copy → `crypto.subtle.importKey` → `finally
   { keyView.fill(0) }` — introduced in phase 77 precisely so the seven AES helpers
   share one testable owned-copy-zeroing seam
   (`packages/crypto/src/aes/import-key.ts`).
2. Functions that *mint and return* key material (`generateEd25519Keypair`,
   `deriveVault*IpnsKeypair`, `unwrapKey`) hand ownership to the caller un-zeroed; the
   caller is the terminal owner. Rust encodes this in types
   (`Zeroizing<Vec<u8>>` returns in `ecies.rs`, `ed25519.rs`, `hkdf.rs`); TS relies on
   callers using `clearBytes`/`clearAll`.
3. Functions that mint, use, and *discard* (`reWrapKey`'s intermediate plaintext, Rust's
   `okm` and stack `key_bytes` copies in `ed25519.rs:46,86`) zero on every exit path,
   including error paths.
4. `clearBytes` is explicitly **best-effort** in JS — GC/JIT copies may survive; the
   doc comment says not to rely on it as a hard guarantee
   (`packages/crypto/src/utils/memory.ts:1-11`). Rust zeroization via `zeroize` is
   compiler-fence-backed.

Error-path zeroization of *minted-but-not-yet-returned* keys (e.g. `createSubfolder`)
is the owning caller's job in sdk-core, not this layer's —
[../parts/sdk-core.md](../parts/sdk-core.md).

### Cross-language parity verification (KAT discipline)

- **The contract** — a change to any frozen encoding must fail a committed test in both
  languages, not surface as silent decryption failure in the field ("a byte mismatch is
  silent total decryption failure", phase 61 TEST-02).
- **Mechanics** — shared JSON fixtures in `tests/vectors/crypto/`; Rust asserts all of
  them in `crates/crypto/tests/cross_language.rs` (aes-gcm, ed25519, ecies, hkdf,
  ipns-name, node-aad, uuid-acceptance). The TS package asserts **only**
  `node-aad.json` and `uuid-acceptance.json` (`build-node-aad.test.ts`); the other five
  fixtures were *generated from* the TS implementation
  (`scripts/generate-test-vectors.ts`) and are asserted TS-side only indirectly via
  hardcoded expected values in the package's own unit tests (e.g.
  `vault-ipns.test.ts`'s pre-computed names). So the parity proof is
  "TS generated, Rust must reproduce" for the older vectors and genuinely two-sided
  for the phase-61/75 ones.
- **Anti-vacuity guards** — both sides hard-assert vector counts and role coverage
  (exactly 4 `aad_vectors` covering roles 1–4, exactly 1 `seal_vector`, ≥2 accept / ≥6
  reject UUID cases) so coverage cannot silently erode
  (`cross_language.rs:287-290,310,396-403`).
- **Standing rules (ADR 0003)** — every new `role` byte must extend the KAT; any
  byte-layout change bumps the domain separator to `node-seal/v2` (old blobs then
  fail-closed rather than mis-decrypt).
- **CI wiring** — the `vector-parity` job runs `cargo test -p cipherbox-crypto --test
  cross_language`, the `@cipherbox/crypto` vitest suite, and
  `scripts/check-vector-parity.sh` (a meta-check of file existence/JSON validity, not a
  re-run). Parity gotchas this discipline exists to catch: the JS `$`-anchor
  trailing-newline UUID trap (above), and — in the sibling IPNS/codec parity surface,
  phase 75 — TS `new Date()` leniency vs Rust's strict RFC3339 parser, cborg's
  all-or-nothing `rejectDuplicateMapKeys` vs Rust's targeted checks, and Rust
  `parse::<T>()` accepting leading signs.

## Runtime variants

- **TS runtime host** — the package requires a Web Crypto secure context;
  `generateRandomBytes` throws `SECURE_CONTEXT_REQUIRED` when `crypto.getRandomValues`
  is missing (plain-HTTP browser contexts). Node ≥19 and browsers both provide the
  `crypto` global; there is no polyfill path.
- **TS build shape** — the CJS bundle inlines *all* dependencies (`noExternal: [/.*/]`)
  so CommonJS consumers (NestJS API, Jest) survive ESM-only deps; the ESM build leaves
  real dependencies external for Vite (`packages/crypto/tsup.config.ts`). Same source,
  two dependency-resolution behaviors.
- **Rust** — no feature gates; builds identically on all platforms (phase 61 C-04).

## Invariants

1. **INV-1** — The node-seal AAD MUST be exactly the 45-byte layout of ADR 0003
   (domain ‖ `0x00` ‖ 16-byte raw UUID ‖ kind ‖ BE-u32 generation ‖ role), byte-identical
   across TS and Rust.
2. **INV-2** — Any change to that layout MUST bump the domain string
   (`node-seal/v1` → `v2`); v1 blobs MUST then fail to unseal under v2 and vice versa.
3. **INV-3** — `buildNodeAad`/`build_node_aad` MUST be fail-closed: invalid kind, role,
   generation, or a non-canonical UUID throws/errs; a wrong-length AAD MUST never be
   produced.
4. **INV-4** — The UUID acceptance domain MUST be canonical-only (8-4-4-4-12 hyphenated,
   case-insensitive hex) and identical in both languages, including rejection of a
   trailing newline; the domain is locked by `uuid-acceptance.json` asserted on both
   sides.
5. **INV-5** — Every seal-level GCM operation MUST mint a fresh random 12-byte IV; the
   seal API MUST NOT accept a caller-supplied IV. IV or CTR-nonce reuse under one key is
   forbidden.
6. **INV-6** — The sealed-blob envelope MUST be `[IV(12)][ciphertext ‖ tag(16)]`,
   IV-first, identical for AAD and non-AAD variants.
7. **INV-7** — Every new role byte MUST land together with a KAT vector covering it, in
   the same change (ADR 0003 merge gate).
8. **INV-8** — A callee MUST NOT zero a caller-owned buffer; only the terminal owner
   zeroes. Internal owned copies of key material MUST be zeroed on every exit path.
9. **INV-9** — Decryption/unwrap failures MUST NOT reveal their cause (wrong key vs
   wrong AAD vs tampering) in message or type distinguishable to an attacker.
10. **INV-10** — Key material MUST live only in `Uint8Array`/byte arrays, never
    converted through strings, never logged, never persisted by this layer.
11. **INV-11** — The ECIES envelope MUST remain eciesjs-v0.4-default-compatible:
    uncompressed 65-byte ephemeral key, **16-byte** GCM nonce, 16-byte tag, HKDF-SHA256
    with empty salt/info over `ephemeralPK ‖ sharedPoint`.
12. **INV-12** — Recipient public keys MUST be validated (65 bytes, `0x04` prefix, valid
    curve point) before any wrap; private keys MUST be exactly 32 bytes.
13. **INV-13** — The HKDF salt `"CipherBox-v1"` and every info string in the derivation
    catalog are frozen; new derivations get new info strings, existing ones are never
    reinterpreted.
14. **INV-14** — CTR mode MUST use a 64-bit big-endian counter (Web Crypto `length: 64`
    ↔ `Ctr64BE`) with the 8-nonce/8-counter IV split, counter starting at 0; CTR output
    MUST never be treated as authenticated.
15. **INV-15** — All randomness MUST come from a CSPRNG (`crypto.getRandomValues` /
    `OsRng`); generation MUST hard-fail rather than fall back when unavailable.
16. **INV-16** — The Ed25519 public key used for IPNS verification MUST be recovered
    from the `ipnsName` itself, never trusted from a record field or DB column.

## Known gaps and quirks

- **Twin surfaces have drifted in both directions.** TS-only: `reWrapKey`, device
  identity, IPNS record verify/parse, `deriveByoConfigIpnsKeypair`, the canonical
  base64 codec, `generateCtrIv`, `publicKeyFromIpnsName`. Rust-only:
  registry/bin/file IPNS derivations (TS moved registry/bin to `packages/core`, and the
  per-file derivation has **no TS production implementation at all** — v3 mints random
  file keypairs), `generate_uuid_v4`, `mime_from_extension`. The `hkdf.rs` header
  comment still claims it "Matches the TypeScript implementations in @cipherbox/crypto
  (vault/derive-ipns.ts, file/derive-ipns.ts, registry/derive-ipns.ts)" — two of those
  three files no longer exist there (`crates/crypto/src/hkdf.rs:4-6`).
- **The KAT symmetry claim is only true for the phase-61/75 vectors.** `aes-gcm.json`,
  `ed25519.json`, `ecies.json`, `hkdf.json`, `ipns-name.json` are asserted by Rust only;
  no TS test loads them (verified by grep at the pinned commit). The TS side pins
  equivalent behavior via hardcoded constants in its own unit tests, but a TS
  regression would be caught only if someone regenerates the vectors.
- **`check-vector-parity.sh` has drifted.** Its `EXPECTED_VECTORS` list omits
  `uuid-acceptance.json` (phase 75) and the node-codec/ipns-verify vectors, while still
  listing the v1-era `tests/vectors/core/{folder-metadata,vault-blob,ipns-record,bin-metadata}.json`
  (`scripts/check-vector-parity.sh:14-25`). It is a meta-check only, so nothing is
  functionally broken, but it no longer describes the real parity surface.
- **Non-crypto utilities live in the Rust crypto crate** (`generate_uuid_v4`,
  `mime_from_extension` with a 40-entry MIME table) — historical accretion from the
  desktop extraction (`crates/crypto/src/utils.rs:44-118`).
- **Dead v1-era exports in TS**: `deriveKey` (public), `deriveContextKey`,
  `generateFolderKey` have zero production consumers outside the package;
  `generateFolderKey` is just `generateRandomKey` under a legacy name
  (`packages/crypto/src/keys/hierarchy.ts`). The non-AAD `sealAesGcm` pair is likewise
  dormant in both languages (kept intentionally, D-00b).
- **`CRYPTO_VERSION = '0.2.0'`** (`packages/crypto/src/index.ts:39`) while the package
  is at `0.33.1` — the constant was never wired to release-please and is stale.
- **Error-code misuse**: `deriveIpnsName` failure throws `SIGNING_FAILED` (no signing
  involved — LOW-backlog item 4, still live at `derive-name.ts:48`); `deriveKey`
  failure throws `ENCRYPTION_FAILED`. Rust's `CryptoError` carries several
  never-constructed variants (`AesCtrEncryptionFailed`, `InvalidKeySize{…}`,
  `Ed25519KeyGenFailed`, …).
- **`.planning/security/LOW-SEVERITY-BACKLOG.md` is itself stale** (last audited
  2026-03-29): items 1–3, 6, 11, 12 cite files that no longer exist
  (`ipns/create-record.ts`, `ipns/marshal.ts`, `folder/metadata.ts` — retired in the
  v3 cutover), and item 5 (un-zeroed AES key copies) was actually resolved by phase
  77's `importAesKey` but is still listed open.
- **`docs/VAULT_EXPORT_FORMAT.md` is PARTIAL**: §3.1/§4 (ECIES envelope, 16-byte nonce,
  binary layout, HKDF internals) remain accurate and are the best reimplementation
  reference; but its key model (`encryptedRootFolderKey` + 64-byte libp2p Ed25519
  private keys, `folderKeyEncrypted` children, v1/v2 folder metadata) predates
  VaultKeyBlob v3 and the node model — see
  [../flows/vault-export-recovery.md](../flows/vault-export-recovery.md).
- **Minor codec divergences outside any oracle**: TS `hexToBytes` accepts an optional
  `0x` prefix; Rust `hex::decode` does not. TS `base64ToBytes` (`atob`) is
  browser-forgiving about padding; Rust consumers use strict `base64::STANDARD`. Neither
  is KAT-locked; both sit on wire boundaries owned by other parts.
- **`@noble/secp256k1` is a runtime import declared as a devDependency** (curve-point
  validation in `wrapKey`); safe today only because tsup bundles it into both outputs.
- **`ECIES_MIN_CIPHERTEXT_SIZE` (81) is below the real envelope minimum (97)** — a
  ciphertext of length 81–96 passes the pre-check and fails inside the library; harmless
  (fail-closed) but the constant's comment ("ephemeral pubkey + auth tag") forgets the
  16-byte nonce.
- **Rust `decrypt_aes_ctr_range` is not re-exported from the crate root** — consumers
  must path through `cipherbox_crypto::aes_ctr::` (`lib.rs:20` re-exports only
  encrypt/decrypt).
- **TS validates ECIES recipient keys as curve points; Rust does not pre-validate** —
  both reject invalid keys, but at different layers with different error specificity.

## Rewrite notes

- **The parity surface should be one auditable manifest, not folklore.** Today "what is
  KAT-locked" is discoverable only by reading two test files, and the meta-check script
  has already drifted. Every frozen encoding (AAD layout, seal envelope, ECIES
  envelope, HKDF catalog, name codec, base64/hex acceptance) should be enumerated in
  one machine-checked manifest that both test suites *and* the parity script consume —
  the phase 61/75 discipline (shared vector + count guards + both-sides assertion) is
  the proven pattern; it just was never retrofitted to the five older vector files.
- **Twin drift is unmanaged outside the KAT'd surface.** The two packages agree
  byte-for-byte where a vector pins them and have quietly diverged everywhere else
  (derivation catalogs, helper inventories, error vocabularies, doc comments naming
  deleted files). A rewrite should either generate one surface from a single source of
  truth or explicitly declare per-primitive ownership ("TS-only", "Rust-only",
  "paired+KAT") so divergence is a decision, not an accident.
- **The ECIES envelope is library-defined, not spec-defined.** CipherBox's most
  durable ciphertexts (vault blobs, grants, TEE enrollment) depend on eciesjs v0.4
  *defaults* — including the non-standard 16-byte GCM nonce — with compatibility
  resting on "same author wrote the Rust crate". `VAULT_EXPORT_FORMAT.md` §3–4 already
  reverse-engineers the layout; a rewrite should promote that to a frozen first-class
  spec with a full-envelope KAT (fixed ephemeral key) so a library major bump cannot
  silently orphan every stored ciphertext.
- **Zeroization should be type-enforced, not comment-enforced.** Rust already does this
  (`Zeroizing`); TS relies on D-09 doc comments at every call site, and the one
  historical violation broke half the E2E suite. A TS rewrite could wrap key material
  in a minimal owning type (or at least lint borrowed-buffer `.fill(0)` calls) instead
  of re-litigating ownership per function.
- **Shed the accretions**: dead v1 exports (`deriveContextKey`, `generateFolderKey`,
  non-AAD seal pair), non-crypto utilities in the Rust crate, the stale
  `CRYPTO_VERSION` constant, and error codes reused across unrelated failure classes.
  The layer's value is that it is small, frozen, and boring; everything that isn't a
  primitive or a frozen encoding belongs a layer up.
