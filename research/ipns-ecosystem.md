# IPNS Decentralized Resolution and Publish Ecosystem

Research findings for ticket #3. All claims sourced from primary sources (specs.ipfs.tech, docs.ipfs.tech, github.com/ipfs and github.com/libp2p repos, blog.ipfs.tech, probelab.io, w3name/Storacha docs). Empirical probes dated 2026-07-18. Unconfirmable items are flagged inline.

## TL;DR

- Client-side IPNS verification is fully specified and small: Ed25519 V2 signature over `"ipns-signature:" + DAG-CBOR data`, EOL check, 10 KiB size cap, sequence-based selection. `js-ipns` does it in the browser today; Rust must use a single-maintainer crate (`rust-ipns`) or hand-roll (~small, well-specified surface).
- The practical trustless transport for both clients is the **delegated routing `/routing/v1` HTTP API** (IPIP-337): `GET`/`PUT /routing/v1/ipns/{name}` carrying the raw signed record, which the client verifies itself. `someguy` (actively maintained, Docker-deployable) bridges HTTP to the Amino DHT for both reads and writes, so a self-hosted someguy is exactly the "optional accelerator" shape CipherBox v2 wants — anyone else's `/routing/v1` endpoint (public delegated-ipfs.dev, any Kubo ≥0.40 gateway) is a drop-in substitute.
- Browser DHT participation is not viable; delegated HTTP is the recommended and only reliable browser path. Rust desktop can embed a libp2p DHT client, but rust-libp2p's kad has no record-validation hooks, so delegated HTTP (or an optional local Kubo) is the pragmatic desktop path too.
- Liveness is the hard constraint: DHT nodes drop records after ~48 h regardless of record EOL, so **someone must re-PUT each record at least every ~48 h (Kubo does every 4 h)**. Re-PUT does not require the private key — any holder of the signed record bytes can republish it until its EOL. That makes a keyless republisher (sourcing records from the network, never from a DB) architecturally possible if records are signed with long EOLs.

## 1. IPNS record format and client-side verification

Source: [IPNS Record and Protocol spec](https://specs.ipfs.tech/ipns/ipns-record/).

The wire format is a protobuf `IpnsEntry` with fields `value`, `signatureV1`, `validityType`, `validity`, `sequence`, `ttl` (all six are legacy V1 duplicates), plus `pubKey`, `signatureV2`, and `data`. The canonical content lives in `data`: a DAG-CBOR document with mandatory keys:

| Key | Meaning |
| --- | --- |
| `Value` | bytes; content path, e.g. `/ipfs/<cid>` |
| `Validity` | ASCII RFC3339 timestamp (nanosecond precision) — the EOL |
| `ValidityType` | uint64; only `0` (EOL) is defined |
| `Sequence` | uint64, starts at 0 |
| `TTL` | uint64 **nanoseconds** — caching hint |

CBOR keys must be canonically sorted (RFC 7049 length-then-bytewise); field order affects signature verification.

**Signatures.** `signatureV2` (current) signs the prefix bytes `"ipns-signature:"` concatenated with the raw CBOR bytes of `data`. `signatureV1` (deprecated since 2021) "MUST never be used for signature verification" though V1 fields may still be populated for backward compat (boxo does this by default).

**What a compliant verifier MUST do**, in order:

1. Reject records over **10 KiB** before protobuf parsing.
2. Confirm `signatureV2` and `data` are present and non-empty.
3. Obtain the public key: from `IpnsEntry.pubKey` if present, else extract it from the IPNS name.
4. Deserialize `data` as DAG-CBOR.
5. Verify `signatureV2` over `"ipns-signature:" + data` bytes.
6. If legacy V1 fields are present, confirm protobuf/CBOR field agreement (`value == data[Value]`, etc.).
7. For `ValidityType = 0`: parse `Validity` and confirm it is in the future.

boxo's `Validate` ([ipfs/boxo `ipns/validation.go`](https://github.com/ipfs/boxo/blob/main/ipns/validation.go)) implements exactly this list and is the Go reference.

**Ed25519 name → key.** An IPNS name is a multihash of a serialized libp2p `PublicKey`; for Ed25519 the multihash uses the **identity** function, so the key is recoverable from the name and `IpnsEntry.pubKey` "MAY be skipped to save space" (it MUST be included only when it cannot be extracted, e.g. legacy RSA). A `k51...` name is a CIDv1, `libp2p-key` codec (`0x72`), base36. Client derivation: decode CID → check codec `0x72` → extract multihash → confirm identity type → digest is a libp2p `PublicKey` protobuf whose `Data` is the raw 32-byte Ed25519 key. (This matches CipherBox v1's `publicKeyFromIpnsName` approach.) The DHT routing key is binary `"/ipns/" + <multihash bytes>` — the raw multihash, not the base36 string.

**Sequence rules.** Sequence MUST increase whenever the value changes. When comparing multiple records: higher sequence wins. The equal-sequence tie-break is **not normative in the spec** (flagged); both reference implementations agree by convention — js-ipns `ipnsSelector` and boxo `Compare` pick the later EOL on equal sequence (boxo additionally prefers V2-signed over V1-only, and finally falls back to `bytes.Compare` of the marshalled records).

### Implementations

- **js-ipns** ([ipfs/js-ipns](https://github.com/ipfs/js-ipns), npm package **`ipns`**): actively maintained (v11.0.0, May 2026). Exposes `createIPNSRecord`, `validate(pubKey, bytes)`, `ipnsValidator` (routing-key form), `ipnsSelector`, `marshalIPNSRecord`/`unmarshalIPNSRecord`, `extractPublicKeyFromIPNSRecord`, `multihashToIPNSRoutingKey`. This is the browser verification story, complete.
- **Rust**: there is **no official IPFS-org Rust IPNS implementation**. The community crate [`rust-ipns`](https://lib.rs/crates/rust-ipns) (Darius Clark, v0.9.0 June 2026, now living in [dariusc93/rust-ipfs](https://github.com/dariusc93/rust-ipfs); the standalone repo 404s — flagged) targets the current spec (deps: `libp2p-identity`, `serde_ipld_dagcbor`, `quick-protobuf`), but it is small and single-maintainer. Alternative: hand-roll against the spec — the MUST-list above is the complete conformance surface and every building block exists as a mature crate (`quick-protobuf`, `serde_ipld_dagcbor`, `ed25519-dalek`/`libp2p-identity`, `cid`).
- **Go** (for comparison): `boxo/ipns` is the reference used by Kubo.

## 2. Record lifetime semantics — three distinct numbers

Sources: [IPNS spec](https://specs.ipfs.tech/ipns/ipns-record/), [Kubo config](https://github.com/ipfs/kubo/blob/master/docs/config.md), [boxo defaults](https://github.com/ipfs/boxo/blob/main/ipns/defaults.go), [docs.ipfs.tech IPNS concepts](https://docs.ipfs.tech/concepts/ipns/), [go-libp2p-kad-dht amino defaults](https://github.com/libp2p/go-libp2p-kad-dht/blob/master/amino/defaults.go).

| Number | What it is | Default |
| --- | --- | --- |
| **Validity/EOL** | Publisher-set expiry inside the signed record; record invalid after this | Kubo `Ipns.RecordLifetime` = **48 h** (boxo `DefaultRecordLifetime`; was 24 h in older Kubo) |
| **TTL** | Caching hint only (nanoseconds in the record) | **5 min** (boxo `DefaultRecordTTL`; docs.ipfs.tech's "one hour" phrasing lags — flagged) |
| **DHT record expiry** | How long DHT nodes keep a PUT record, independent of EOL | **48 h** (Amino `DefaultMaxRecordAge`; "records expire after 48 hours" per docs.ipfs.tech) |
| **Republish interval** | How often the publisher re-PUTs so the record survives node churn | Kubo `Ipns.RepublishPeriod` = **4 h** |

Key consequences:

- A record signed with a 1-month EOL still vanishes from the DHT ~48 h after the last PUT. In practice churn erodes availability well before 48 h, hence Kubo's 4 h republish cadence.
- **Republishing does not require the private key.** A re-PUT of the same signed record bytes is valid until the record's own EOL; only re-signing (extending EOL or bumping sequence) needs the key. Any party holding the record bytes can keep a long-EOL record alive.
- Resolver caches honor the record TTL (Kubo `Ipns.MaxCacheTTL` can cap it server-side).

## 3. Delegated routing: `/routing/v1` (IPIP-337)

Source: [Routing V1 HTTP API spec](https://specs.ipfs.tech/routing/http-routing-v1/).

**Endpoints:**

- `GET /routing/v1/providers/{cid}` and `GET /routing/v1/peers/{peer-id}` — provider/peer records (JSON or ndjson).
- `GET /routing/v1/ipns/{name}` — name as CIDv1; `200` returns the raw record with `Content-Type: application/vnd.ipfs.ipns-record`. "Any other content type MUST be interpreted as 'no record found'."
- `PUT /routing/v1/ipns/{name}` — body is the serialized record "with a valid signature matching the `name` path parameter". `200` = published, `400` = invalid, `406` = wrong content type.
- Optional `GET /routing/v1/dht/closest/peers/{key}` (IPIP-476) for light clients doing their own DHT walks.

**Trust split:** on PUT the server MUST validate the signature against the name — that is the only mandated server-side check, and the spec is silent on what the server does with an accepted record (DHT propagation is an implementation choice). On GET the response is the verifiable record format; the [Trustless Gateway spec](https://specs.ipfs.tech/http-gateways/trustless-gateway/) states the client duties explicitly: "A Client MUST confirm the record signature match `libp2p-key` from the requested IPNS Name" and "MUST perform additional record verification according to the IPNS specification". So the transport is untrusted by design — exactly compatible with CipherBox's network-canonical constraint. Caching: `Cache-Control: max-age` SHOULD mirror the record TTL (default `max-age=60`).

### someguy

[ipfs/someguy](https://github.com/ipfs/someguy): "an HTTP Delegated Routing V1 server that proxies requests to the Amino DHT and other delegated routing servers." Runs an Amino DHT client (default accelerated) in parallel with delegated HTTP routers (endpoints auto-configured from `conf.ipfs-mainnet.org/autoconf.json`; IPNI/cid.contact arrives that way for providers). **Both IPNS GET and PUT are supported**, and `parallelRouter.PutIPNS` fans a PUT out to all configured routers including the DHT — so a PUT to someguy genuinely publishes to the Amino DHT. Deployable via `ghcr.io/ipfs/someguy` Docker image or `go install`. Actively maintained (v0.14.1, 2026-07-15, Shipyard/Kubo team).

### Public endpoints

- **`https://delegated-ipfs.dev/routing/v1`** — public someguy run by Interplanetary Shipyard / Waterworks Community on behalf of the IPFS Foundation ([docs.ipfs.tech public utilities](https://docs.ipfs.tech/concepts/public-utilities/)). **No published rate limits, quota, or SLA** (flagged — treat as best-effort). Empirical probe (2026-07-18): the IPNS PUT handler is wired (wrong content type → `406` per spec, not `405`); whether that deployment's PUT reaches the DHT is unconfirmable from docs, though the code default does.
- **`https://cid.contact/routing/v1`** (IPNI) — providers only; `/ipns/` returns 404. No IPNS.
- **Any Kubo ≥ 0.40** exposes `/routing/v1` on its gateway port by default (`Gateway.ExposeRoutingAPI`, incl. IPIP-476) — so a CipherBox-hosted Kubo is automatically also a delegated-routing accelerator.
- **Cloudflare**: no `/routing/v1` endpoint found (their IPFS gateway was retired; negative evidence only — flagged).
- Relevant twist: **Kubo ≥ 0.37 can itself be a delegated-routing *client*** (`Ipns.DelegatedPublishers`, `Routing.Type=delegated`, `ipfs name publish --allow-delegated`) — publishing IPNS via HTTP without DHT connectivity.

**Trustless gateways** also serve records: `GET /ipns/{name}` with `Accept: application/vnd.ipfs.ipns-record` on `trustless-gateway.link`, `ipfs.io`, `dweb.link` (probed: the code path is wired on all three; no live name was available to demonstrate a 200 record body — flagged). This is a read-only alternative to `/routing/v1` GET.

## 4. Browser reality (Helia / js-libp2p)

Sources: [@helia/ipns source](https://github.com/ipfs/helia/tree/main/packages/ipns), [helia-delegated-routing-v1-http-api](https://github.com/ipfs/helia-delegated-routing-v1-http-api), [js-libp2p configuration docs](https://github.com/libp2p/js-libp2p/blob/main/doc/CONFIGURATION.md), [libp2p browser connectivity guide](https://libp2p.io/docs/browser-connectivity/), [AutoTLS announcement](https://libp2p.io/blog/autotls/), [Shipyard 2024 post](https://blog.ipfs.tech/2024-shipyard-improving-ipfs-on-the-web/), [docs.ipfs.tech nodes](https://docs.ipfs.tech/concepts/nodes/).

**`@helia/ipns` capabilities:**

- Routers: default Helia routing (aggregating libp2p and/or HTTP routers), a local-datastore cache router, delegated HTTP routing (`delegatedHTTPRouting('https://delegated-ipfs.dev')` in `@helia/routers`, now `packages/delegated-routing-client` on main), pubsub router, and custom routers.
- `publish()`: builds the signed record (lifetime default 48 h, TTL default 5 min, V1+V2 sigs by default), auto-increments sequence from the local store, and **PUTs to all routers in parallel** — including HTTP `PUT /routing/v1/ipns/{name}` via the delegated router. A built-in republisher re-PUTs every 1 h by default.
- `resolve()`: local cache first (honoring TTL), then all routers in parallel; **client-side verification on by default** (`ipnsValidator`: V2 signature against the key from the routing key or embedded pubkey, EOL, 10 KiB cap, V1/V2 field agreement), best record chosen by `ipnsSelector` (highest sequence).

So a browser can, today, both resolve and publish IPNS trustlessly over delegated HTTP with real cryptographic verification. That path does not even require libp2p.

**Why the DHT path is not viable in browsers:**

- A browser can only ever be a DHT *client* (`@libp2p/kad-dht` needs a publicly dialable address for server mode; browsers cannot open TCP ports or send UDP).
- Transport mismatch: most Amino DHT servers listen on TCP/QUIC only. Browsers can dial only peers offering Secure WebSockets with browser-trusted certs (AutoTLS `*.libp2p.direct`, Kubo ≥ 0.32.1 opt-in), WebTransport (dial-only in js-libp2p; Shipyard deprioritized it over browser implementation bugs; Safari still in progress), or WebRTC-direct (5–7 RTT setup; not in Service Workers; Chrome caps ~500 connections/window). No primary source publishes the dialable fraction of DHT servers (flagged).
- Official position is the inverse recommendation: delegated routing "is useful in browsers and other constrained environments where it's infeasible to be a DHT server/client" (docs.ipfs.tech). There is no official statement that `@libp2p/kad-dht` is supported in browsers.

## 5. Desktop (Rust) paths

Sources: [rust-libp2p ipfs-kad example](https://github.com/libp2p/rust-libp2p/blob/master/examples/ipfs-kad/README.md), rust-libp2p issues [#2140](https://github.com/libp2p/rust-libp2p/issues/2140)/[#2281](https://github.com/libp2p/rust-libp2p/issues/2281)/[#2448](https://github.com/libp2p/rust-libp2p/issues/2448), [Kubo RPC reference](https://docs.ipfs.tech/reference/kubo/rpc/).

| Path | Verdict |
| --- | --- |
| **Embedded rust-libp2p kad (DHT client)** | Workable but raw. `put_record`/`get_record` interop with Amino (public DHT servers accept only `/pk/` and `/ipns/` value records). **No validator/selector hooks exist** (long-open issues; workaround is a custom `RecordStore`), so the app must marshal, sign, validate, and select (highest-seq) IPNS records itself. Full sovereignty, no server dependency, but you own all record logic and pay DHT-walk latency. |
| **Delegated `/routing/v1` HTTP client** | The pragmatic default. The API surface is two endpoints; any Rust HTTP client implements it (no official Rust client crate — flagged). Combine with self-implemented verification from §1. Works against CipherBox-hosted someguy/Kubo *or* any public endpoint — nothing breaks if CipherBox infra disappears, honoring the constraint. |
| **Local Kubo daemon (RPC)** | The boring fully-supported path: `/api/v0/name/publish` (lifetime default 48 h, TTL 5 m, `--allow-offline`, `--allow-delegated`) and `/api/v0/name/resolve` (quorum `dht-record-count` default 16, `dht-timeout` 1 m). Heavyweight to bundle with a Tauri app; best as an optional power-user accelerator, not the default. Kubo verifies records itself. |
| **iroh** | Not an option — departed from IPFS interop (no IPNS, no Amino DHT). |

## 6. Latency and liveness in practice

Sources: [ProbeLab IPNS-on-Amino measurement](https://probelab.io/blog/ipns-performance-amino-dht) (Aug 2025), [ProbeLab Optimistic Provide](https://probelab.io/blog/optimistic-provide/), [Kubo v0.39 release](https://github.com/ipfs/kubo/releases/tag/v0.39.0), [docs.ipfs.tech IPNS](https://docs.ipfs.tech/concepts/ipns/).

- **DHT resolve (measured):** median **~11 s**, p10 ~5–6 s, tails 37–60 s — dominated by the Kubo-parity quorum of 16 distinct-peer responses to find the newest record. ProbeLab: "non-optimal for interactive use cases." All retrievals in the study succeeded (records were live).
- **DHT publish:** classic DHT PUT walk historically median ~20 s (~80% within 20 s, ~95% within 60 s). Optimistic Provide brought *provider record* publishes to ~0.7 s (default since Kubo v0.39), but that covers provider records, **not IPNS PUTs** — applying it to IPNS is explicitly future work (flagged).
- **Delegated HTTP resolve:** a `GET /routing/v1/ipns/{name}` against a warm someguy/gateway returns in normal HTTP time (sub-second to low seconds); a cold someguy still pays the DHT walk behind the scenes. `Cache-Control: max-age` mirrors the record TTL (default 5 min), so freshness lag is bounded by TTL.
- **Liveness:** a name stays resolvable only while someone re-PUTs the record within every ~48 h DHT-expiry window (Kubo: every 4 h). If the publisher and all republishers go offline longer than that, the name goes dark even if the record's own EOL is months out — and comes back the moment anyone re-PUTs the stored record bytes.

## 7. Hosted IPNS options and pinning-service interactions

- **w3name** ([storacha/w3name](https://github.com/storacha/w3name), API at `name.web3.storage`): HTTP record store speaking real signed IPNS records. `GET /name/:key` → `{value, record}`; `POST /name/:key` accepts a record iff signature valid, key matches, validity in future, sequence higher (or tie-break). WebSocket watch endpoint. Rate limit **30 req/10 s per IP**. **It does not touch the DHT in either direction** ("it does not consult the IPFS DHT"; no republishing — flagged against any claim otherwise). Trust model: cannot forge (client-signed, client library re-validates), but **can withhold or serve a stale lower-seq record** — nothing forces it to serve the latest. Status: alive but low-activity (not archived; last release Sep 2025; survived the web3.storage sunset explicitly). As a closed record store it **violates CipherBox's network-canonical constraint** if used alone; at best a side-channel accelerator.
- **Filebase** and **Fleek**: hosted IPNS publishing APIs, but **key-custodial** — their service holds the IPNS key and publishes on your behalf (Fleek docs: propagation "1 to 30 minutes"). Non-starters for E2EE key sovereignty. (Vendor docs, outside the IPFS-org source set — flagged.)
- **Pinning services generally:** the [Pinning Service API](https://ipfs.github.io/pinning-services-api-spec/) has no IPNS concept — pinning keeps *content* (CIDs) alive; IPNS liveness (record re-PUT) is a separate, orthogonal job nobody's pinning API does for you. Plan for both independently.

## 8. IPNS over PubSub

Sources: [IPNS PubSub router spec](https://specs.ipfs.tech/ipns/ipns-pubsub-router/), [Kubo experimental features](https://github.com/ipfs/kubo/blob/master/docs/experimental-features.md#ipns-pubsub), Kubo v0.40 changelog.

- **Not deprecated.** Spec maturity is `reliable` (2022); Kubo status is "experimental, default-disabled" (`Ipns.UsePubsub` / `--enable-namesys-pubsub`), and Kubo v0.40 (2026) actively *improved* it (persistent per-peer max-seqno validation fixing message cycles). Publishers push to the topic in addition to the DHT; both sides must opt in.
- **Browser viability:** `@helia/ipns` ships `pubSubIPNSRouting`, but with an explicit caveat — only suitable where updates are frequent and multiple peers subscribe, otherwise publishes fail with "Insufficient peers"; and a subscriber only receives records published *while subscribed*, so first resolution always needs another router.
- Verdict for CipherBox: a potential low-latency *push* overlay between online clients sharing a vault, never a resolution primitive. Do not build on it as a primary path.

## 9. Implications for CipherBox v2

The network-canonical constraint maps cleanly onto the ecosystem's own direction: every viable path carries the same signed record format that the client verifies itself, so CipherBox infra (someguy and/or Kubo) can sit in the middle as a pure accelerator that any public `/routing/v1` endpoint or trustless gateway can substitute for. The API database never needs to see a record.

Two architectural consequences beyond transport choice:

1. **Replace the TEE republisher's DB source with the network itself.** Since re-PUT needs no private key, a republisher can resolve a record from the DHT/delegated routing and re-PUT the same bytes — keeping names alive until each record's EOL without storing records in the DB. This requires **signing records with long EOLs** (weeks–months, not Kubo's 48 h default) and accepting the trade-off docs.ipfs.tech names explicitly: long EOL = availability, short EOL = faster invalidation of stale pointers. Sequence-based selection means a newer record always beats a republished old one, so long EOLs are safe for a system where clients bump sequence on every write. The TEE (which can re-sign with extended EOL) remains useful only for names whose owners are offline longer than their EOL.
2. **Publish must fan out.** Helia's model — PUT to all routers in parallel (DHT via someguy + any public endpoint) — is the right durability posture: a publish that lands only on CipherBox infra violates the constraint the moment that infra dies.

### Candidate architectures

| Client | Architecture | Resolve UX | Publish UX | Liveness / notes |
| --- | --- | --- | --- | --- |
| Browser | **A (recommended): `@helia/ipns` + delegated HTTP routing** → self-hosted someguy, with public `delegated-ipfs.dev` + trustless-gateway record GET as fallbacks; client-side `ipnsValidator` always on | Sub-second–few s warm (HTTP + cache, bounded by 5 min TTL); cold resolve pays the ~11 s DHT walk server-side | HTTP PUT accepted in <1 s; DHT propagation behind someguy ~seconds–tens of s | Fully trustless (client verifies); nothing breaks without CipherBox infra (fallback endpoints are drop-in); public endpoint has no SLA/rate-limit docs |
| Browser | B: browser libp2p DHT client (WebTransport/WSS/WebRTC) | Unreliable — transport-starved (most DHT servers not browser-dialable), quorum walk ≥11 s when it works | Same, worse | Not viable today per official guidance; revisit if AutoTLS adoption changes the dialable fraction |
| Browser | C: w3name-style hosted record store | Fast (single HTTP GET) | Fast | Fails the constraint: closed store, no DHT, service can withhold/serve-stale; at most an extra side-channel |
| Desktop (Rust) | **A (recommended): `/routing/v1` HTTP client** (same endpoints/fallbacks as browser) + own spec-compliant verify/sign (rust-ipns crate or hand-rolled ~small surface) | Same as browser A | Same as browser A | One resolution/verification code path shared conceptually with browser; zero daemon footprint |
| Desktop (Rust) | B: embedded rust-libp2p kad DHT client (in addition to A) | ~11 s median direct DHT resolve; useful as a no-infra fallback and for verifying network state independently | ~20 s classic DHT walk PUT | Maximum sovereignty — works with zero HTTP endpoints alive; cost: no kad validation hooks, all record logic app-owned |
| Desktop (Rust) | C: optional local/bundled Kubo (RPC) | 16-peer quorum, `dht-timeout` 1 m; good once warm | Kubo handles republish every 4 h while running | Best as opt-in power-user accelerator (also gives the user their own `/routing/v1` server since v0.40); too heavy as the default Tauri dependency |
| Both | Republisher service (replaces TEE-from-DB): resolve-then-re-PUT of network-sourced records; TEE re-signing only for EOL extension | n/a | n/a | Requires long-EOL signing policy; DB never serves records; any third party could run the same republisher from public data |

### Recommended shape

Browser and desktop both speak `/routing/v1` GET/PUT with mandatory client-side verification (js-ipns / rust-ipns-or-hand-rolled), publishing in parallel to a CipherBox someguy and at least one independent endpoint; desktop optionally adds an embedded DHT client as the infra-free fallback; records are signed with long EOLs and kept alive by a keyless resolve-and-re-PUT republisher, with TEE re-signing reserved for EOL extension of long-offline names.
