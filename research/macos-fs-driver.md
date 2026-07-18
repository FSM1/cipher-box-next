# macOS Filesystem Driver for the v2 Desktop

Resolves [cipher-box-next#32](https://github.com/FSM1/cipher-box-next/issues/32). Research date: 2026-07-18. All version numbers and dates were verified against upstream releases, Apple documentation, and issue trackers; each claim links its primary source. Opinion is confined to the **Recommendation** section.

## Question

Which macOS filesystem driver should the v2 desktop mount target — FSKit (Apple's native framework, possibly via macFUSE's FSKit backend), classic macFUSE (kext), or FUSE-T (NFS/SMB localhost shim)?

v1 uses FUSE-T (default NFS backend) and suffers its cache behavior: FUSE `inval_inode` notifications are ignored, cross-client sync flakes ~15%, and overwrite-rename semantics diverge. Windows (WinFsp) and Linux (`fuser`) are fine — this is macOS-only.

## v1 baseline: what the common core actually needs

The v1 FUSE layer (`crates/fuse` in cipher-box) implements a standard passthrough surface of 22 operations via `fuser` 0.16 (`libfuse` feature, so it links whatever libfuse the host provides — macFUSE's or FUSE-T's shim) and `winfsp` 0.12 behind feature flags:

`init`, `destroy`, `lookup`, `getattr`, `setattr`, `open`, `read`, `write`, `flush`, `release`, `create`, `unlink`, `rename`, `mkdir`, `rmdir`, `opendir`, `readdir`, `releasedir`, `statfs`, `access`, `getxattr`, `listxattr`

No byte-range locking, no `copy_file_range`, no fallocate. Cache invalidation is handled app-side (internal metadata cache with gated re-resolve); the unsolved layer in v1 is the OS-side page/dentry/attribute cache, which FUSE-T's NFS backend never invalidates. Any v2 macOS backend must provide:

- The 22 ops above (or their vfs-level equivalents)
- A working push-invalidation channel (the `inval_inode`/`inval_entry` class), so a remote IPNS change becomes visible without waiting for a TTL
- Faithful atomic overwrite-rename semantics (the macOS FUSE-T overwrite-rename divergence broke v1's cross-client tests)
- Stable file IDs and monotonic mtimes (several macOS client caches key on these)
- Unicode normalization tolerance (v1 pairs `fuser` with `unicode-normalization` for NFC/NFD handling)

## Option 1: FSKit

### Maturity and version floors

- FSKit shipped functionally in macOS 15.4 (all core types are marked `introducedAt: 15.4` in [Apple's FSKit docs](https://developer.apple.com/documentation/FSKit)); the API surfaced in the macOS 15 betas after WWDC 2024 ([Eclectic Light, 2024-06-26](https://eclecticlight.co/2024/06/26/how-file-systems-can-change-in-sequoia-with-fskit/)).
- macOS 15.4–15.x supports **block-device-backed** filesystems only. Path/URL-backed resources — what a synthetic E2EE drive needs — arrived in macOS 26 via [`FSPathURLResource`](https://developer.apple.com/documentation/fskit/fspathurlresource) (`introducedAt: 26.0`), and Apple's first real sample, [Building a passthrough file system](https://developer.apple.com/documentation/fskit/building-a-passthrough-file-system), requires macOS 26.
- Only `FSUnaryFileSystem` (one resource → one volume) is supported; the fuller `FSFileSystem` exists in the API but is unusable, still true in the macOS 27 beta docs ([FSKit docs](https://developer.apple.com/documentation/FSKit)).
- No dedicated WWDC session exists in 2024, 2025, or 2026; Apple staff confirmed "no sessions about it" ([forums thread 756630](https://developer.apple.com/forums/thread/756630)).

### The two blockers, both fixed only in the macOS 27 beta

- **No cache invalidation before macOS 27.** In 15.4–26 there is no module-initiated invalidation/notification API at all — corroborated by macFUSE's wiki stating the FUSE notification API "is not supported, yet" on its FSKit backend ([macFUSE FUSE-Backends wiki](https://github.com/macfuse/macfuse/wiki/FUSE-Backends)). The macOS 27 beta adds exactly this: [`FSVolume.DataCacheHandler`](https://developer.apple.com/documentation/fskit/fsvolume/datacachehandler) with `setCacheState` actions `push` / `pushInvalidate` / `invalidate` — a genuine `inval_inode` equivalent, but beta-only today. For CipherBox this is decisive: shipping FSKit before macOS 27 means *worse* coherence than the FUSE-T situation v1 already suffers.
- **No sanctioned programmatic mount before macOS 27.** Through macOS 26 the blessed workaround is spawning `/sbin/mount` yourself, per Apple DTS on the forums, which is also incompatible with App Sandbox ([forums thread 799283](https://developer.apple.com/forums/thread/799283)). The macOS 27 beta adds [`FSClient.mountSingleVolume`](https://developer.apple.com/documentation/fskit/fsclient), gated on a new `com.apple.developer.fskit.mount` entitlement whose App Store availability is not yet documented.

### Other characteristics

- Install friction is essentially zero: FSModules are ordinary app extensions with the [`com.apple.developer.fskit.fsmodule` entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.fskit.fsmodule) — no kext, no system-extension approval, no Reduced Security; the user toggles the module in System Settings → General → Login Items & Extensions → File System Extensions. Apple states FSKit modules are "app extensions compatible with Mac App Store distribution" ([FSKit docs](https://developer.apple.com/documentation/FSKit)).
- Performance of the userspace read/write path is acknowledged by Apple DTS as unoptimized; one developer measured 100–150% CPU vs 40% under macFUSE for the same workload and shelved their port ([forums thread 799283](https://developer.apple.com/forums/thread/799283)).
- Early-adopter reports: hung volume calls can wedge processes with no timeout, stale-appex resolution after app updates needs `lsregister`/`pluginkit` workarounds, kext-vs-FSKit probing conflicts ([forums thread 788609](https://developer.apple.com/forums/thread/788609)).
- Rust story: [`objc2-fs-kit`](https://docs.rs/objc2-fs-kit/latest/objc2_fs_kit/) (v0.3.2, part of the actively maintained objc2 framework-crates project) provides full bindings, so FS logic can live in Rust — but the FSModule must be packaged as an ExtensionKit appex that `fskitd` launches, so realistically a thin Swift shell delegates into the shared Rust core (prior art for shim-to-foreign-core: [debox-network/FSKitBridge](https://github.com/debox-network/FSKitBridge)). There is no `fuser` path: [cberner/fuser#306](https://github.com/cberner/fuser/issues/306) (FSKit support) has been an open request since 2024.

## Option 2: macFUSE

### Release line and kext status

- Latest stable: macFUSE 5.3.3, 2026-07-04, supporting macOS 12–27, Apple Silicon + Intel ([releases](https://github.com/macfuse/macfuse/releases), [5.3.3 announcement](https://macfuse.github.io/2026/07/04/macfuse-5.3.3.html)). Actively maintained — 5.3.1 (2026-06-13) already added initial macOS 27 support ([announcement](https://macfuse.github.io/2026/06/13/macfuse-5.3.1.html)).
- First-time kext install on Apple Silicon, per the official [Getting Started wiki](https://github.com/macfuse/macfuse/wiki/Getting-Started): full shutdown → boot into Recovery → Startup Security Utility → select **Reduced Security** + "Allow user management of kernel extensions" → restart → approve in System Settings → restart again. Every macFUSE update requires another approve-and-restart. This is exactly the friction v1 adopted FUSE-T to avoid.
- Apple's official stance: kexts "have been deprecated," with FSKit positioned as the filesystem successor; no hard cutoff version has been announced ([Apple kernel extension support page](https://developer.apple.com/support/kernel-extensions/)). The kext still loads on macOS 26 and the 27 beta, with per-release churn actively patched — but the surface is inherently panic-capable (reproducible kernel panic on macOS 26.2/M4 with 5.1.3: [macfuse#1143](https://github.com/macfuse/macfuse/issues/1143)).

### macFUSE's FSKit backend

- Real and headline of the 5.x line: introduced experimental in [5.0.0](https://github.com/macfuse/macfuse/releases/tag/macfuse-5.0.0) (2025-05-05), opt-in per mount with `-o backend=fskit` on macOS 15.4+; libfuse3 support in [5.1.0](https://macfuse.github.io/2025/10/30/macfuse-5.1.0.html); sandbox-capable XPC mount API in [5.2.0](https://macfuse.github.io/2026/04/09/macfuse-5.2.0.html); a channel API making FSKit-backend reads up to 15x faster in [5.3.0](https://macfuse.github.io/2026/06/08/macfuse-5.3.0.html). The kext backend remains the default.
- Documented FSKit-backend limitations ([FUSE-Backends wiki](https://github.com/macfuse/macfuse/wiki/FUSE-Backends)): mount points restricted to `/Volumes`, all files opened read/write, **FUSE notification API unsupported** (so still no `inval_inode`), no `fuse_context`, many kernel-handled mount options unimplemented. A field report found rclone over `backend=fskit` "unusable" as of Nov 2025 ([rclone forum](https://forum.rclone.org/t/macfuse-fskit-backend/53037)).
- The FSKit backend is exposed through libfuse/libfuse3's session machinery. `fuser` bypasses libfuse's event loop and speaks the FUSE wire protocol against the kernel device directly, so `fuser`-over-macFUSE-FSKit is unsupported/unverified — no issue or PR for it exists upstream.

### `fuser` crate and licensing

- `fuser` 0.17.0 (2026-02-14) still marks its macOS support "(untested)", and the project no longer accepts pull requests ([cberner/fuser](https://github.com/cberner/fuser)). The macFUSE 4.x/5.x rename-protocol divergence fix was proposed in [PR #453](https://github.com/cberner/fuser/pull/453) and closed unmerged on 2026-07-17; [issue #341](https://github.com/cberner/fuser/issues/341) (rename delivers wrong source name on macOS) remains open. Rename correctness is not a corner case for a mounted drive.
- Hard commercial blocker: the macFUSE license forbids binary redistribution "bundled with commercial software... includ[ing] the automated download or installation" without written permission, and the wiki states commercial bundling requires a paid license ([LICENSE.txt](https://github.com/macfuse/macfuse), [Open-Source-Status wiki](https://github.com/macfuse/macfuse/wiki/Open-Source-Status)). The free path is user-driven separate install (the Cryptomator/VeraCrypt model).

## Option 3: FUSE-T

### Upstream status

- Latest release 1.2.7, 2026-06-03; actively maintained after a quiet Aug 2025 → Mar 2026 stretch, with a dense burst of releases Mar–Jun 2026 ([releases](https://github.com/macos-fuse-t/fuse-t/releases)). The core NFS server remains closed-source; the public pieces are the libfuse shim fork and — new — a Go SMB2/3 server (`go-smb2`, pushed 2026-05-28) that appears to be the SMB backend.
- License: free for non-commercial use; "for commercial use or/and bundling with commercial software the software vendor has to obtain a commercial license" ([License.txt](https://raw.githubusercontent.com/macos-fuse-t/fuse-t/main/License.txt)).
- FUSE-T grew its own FSKit backend in 1.1.0 (macOS 26+) and experimental upstream-libfuse3 support in 1.2.0 ([releases](https://github.com/macos-fuse-t/fuse-t/releases)).

### Cache invalidation: the v1 pain, confirmed structural on NFS — and newly fixed on SMB

- The wiki documents that FUSE notifications "work for SMB backend" and are "unsupported for NFS and FSKit" ([fuse-t wiki](https://github.com/macos-fuse-t/fuse-t/wiki)). The v1 flake is therefore structural on the default NFS backend: `inval_inode` can never work there, and coherence is purely macOS NFS client cache expiry. There is no documented `actimeo` passthrough on a fuse-t mount; the only documented knobs are `-o noattrcache` and `-o nomtime`.
- **1.2.1 (2026-04-05) added "invalidation support for smb backend"** ([release notes](https://github.com/macos-fuse-t/fuse-t/releases/tag/1.2.1)), followed by SMB message-reordering and multi-connection fixes in 1.2.5/1.2.7 (May–Jun 2026). This is the first shipping configuration in the entire option space where the common core's `inval_inode` calls can actually reach the OS cache.
- Caveats on the SMB backend: "Is SMB ready for production?" is open since 2024 ([#78](https://github.com/macos-fuse-t/fuse-t/issues/78)); AppleDouble `._` files appear despite `noappledouble` ([#81](https://github.com/macos-fuse-t/fuse-t/issues/81)); the invalidation support is ~3 months old.
- Known coherence/stability issues on NFS: stale mounted drive ([#76](https://github.com/macos-fuse-t/fuse-t/issues/76), closed without visible resolution), attribute-cache degradation after large scans that `-o noattrcache` eliminates ([#105](https://github.com/macos-fuse-t/fuse-t/issues/105)), data corruption/hangs under concurrent copies ([#45](https://github.com/macos-fuse-t/fuse-t/issues/45), open since 2023).
- **[#109](https://github.com/macos-fuse-t/fuse-t/issues/109) (open, 2026-07-02): macOS kernel panic (`nfs_vinvalbuf2: ubc_msync failed!`) reachable through any fuse-t NFS mount**, root-caused to a race in Apple's NFS kext (filed as FB23527406), with listed triggers of attribute churn, `namedattr` xattr traffic, and atomic-rename rewrites of small files — i.e., precisely an E2EE drive's workload. This indicts Apple's NFS client itself, which also taints any hand-rolled NFS-loopback alternative.
- No upstream issue matches v1's observed overwrite-rename divergence — it appears unreported; worth filing.

## Option 4: fourth options

- **Hand-rolled localhost NFS server in Rust** ([`nfsserve`](https://crates.io/crates/nfsserve), v0.11.0 2026-04-01, now maintained by Hugging Face; NFSv3 only): you gain control of mount options FUSE-T hides (`actimeo=0`/`noac` per [mount_nfs man page](https://ss64.com/mac/mount_nfs.html); `lookupcache` is not supported on macOS) — but it is architecturally the same TTL-coherence model with no push invalidation (NFSv3 has no server→client callbacks), the same Apple NFS kext panic surface as fuse-t [#109](https://github.com/macos-fuse-t/fuse-t/issues/109), plus open correctness bugs of its own (duplicate `ls` entries: [nfsserve#35](https://github.com/xetdata/nfsserve/issues/35)). Not a coherence fix.
- **WebDAV loopback**: macOS `webdavfs` has no invalidation protocol, caches aggressively with no tuning knobs, and Finder floods it with PROPFIND ([QNAP FAQ](https://www.qnap.com/en/how-to/faq/article/why-is-mac-finder-slow-when-accessing-nas-shared-folders-via-webdav), [Nextcloud reports](https://help.nextcloud.com/t/poor-webdav-performance-with-mac-os-finder/108435)). Strictly dominated by the NFS option; not viable.
- **Apple File Provider** ([`NSFileProviderReplicatedExtension`](https://developer.apple.com/documentation/fileprovider/nsfileproviderreplicatedextension)): the sanctioned cloud-storage path (Dropbox/Drive/OneDrive were moved onto it) and the only option here with genuinely push-based coherence and full App Store compatibility. But it is a sync/materialize model, not passthrough: files are dataless until materialized into an on-disk replica under `~/Library/CloudStorage`, so decrypted plaintext persists on disk (at-rest story degrades to FileVault), and non-plain-POSIX reads fetch whole files despite [partial-content fetching](https://developer.apple.com/documentation/fileprovider/nsfileproviderpartialcontentfetching). A Rust core can drive a thin Swift appex over XPC — Nextcloud's [`FileProviderExt`](https://github.com/nextcloud/NextcloudFileProviderKit) with a C++/Qt core is the working precedent — but it is a second FS frontend that does not map onto the `fuser` inode model.

## Comparison

| | FUSE-T ≥1.2.7 SMB | FUSE-T NFS (v1 config) | macFUSE kext | macFUSE FSKit backend | Native FSKit | File Provider |
|---|---|---|---|---|---|---|
| `inval_inode` works | Yes (since 1.2.1) | No (structural) | Yes | No | macOS 27 beta only | N/A (push model) |
| Keeps `fuser` common core | Yes | Yes | Yes (with rename-ABI risk) | No (libfuse only) | No (Swift appex + bindings) | No (new frontend) |
| Install friction | None | None | Recovery mode + Reduced Security + reboots | macFUSE install (no kext approval) | Settings toggle only | Settings toggle only |
| macOS floor | 13 | 13 | 12 | 15.4 | 26 (path-backed); 27 for mount+invalidation | 12.3+ |
| Commercial bundling | Paid license | Paid license | Paid license | Paid license | Free | Free |
| Passthrough (no plaintext replica) | Yes | Yes | Yes | Yes | Yes | No |
| Maturity risk | Medium (SMB invalidation ~3 months old) | Known-bad + panic surface (#109) | Mature, panic-capable, Apple-deprecated | Field-reported unusable (Nov 2025) | APIs still landing in beta | Mature |

## Recommendation

*This section is opinion, derived from the facts above.*

**Primary target: FUSE-T ≥ 1.2.7 with the SMB backend, keeping the `fuser`-based common core.** It is the only shipping-today configuration where the common core's `inval_inode` calls actually reach the OS cache — which is the exact v1 failure — with zero install friction and no change to the cross-platform Rust FS layer. The NFS backend (v1's config) should be abandoned regardless of any other decision: its invalidation gap is documented-structural, and fuse-t #109 shows the E2EE workload can panic Apple's NFS kext.

**Strategic successor: a native FSKit module, planned but not built yet.** Once macOS 27 ships stable with `FSVolume.DataCacheHandler` (real invalidation) and `FSClient.mountSingleVolume` (real programmatic mount), FSKit becomes the Apple-sanctioned, friction-free, license-free endgame — via a thin Swift appex shell delegating into the shared Rust core (`objc2-fs-kit` or FFI/XPC), not via `fuser`. Do not adopt it before then: pre-27 FSKit has *no* invalidation at all, which is strictly worse than what v1 suffers today.

**Rejected:** macFUSE kext (Recovery-mode install friction, paid bundling license, Apple-deprecated, kernel-panic surface, and `fuser`'s unresolved macFUSE-5 rename ABI risk); macFUSE FSKit backend (unreachable from `fuser`, no notification API, poor field reports); NFS-based options including `nfsserve` loopback (structurally TTL-coherent, shared Apple NFS kext panic surface); WebDAV (dominated). **File Provider** is the fallback only if strict coherence is later deemed to trump passthrough semantics — its materialized plaintext replica is a real E2EE regression, so it should not be the default choice for a mounted-drive product.

**What the common core should demand from the macOS backend (design consequence):** keep the core's FS surface at the vfs-operation level (the 22 ops above) behind a host-adapter trait with an explicit push-invalidation callback — not at the FUSE wire-protocol level — so the FUSE-T/`fuser` backend today and an FSKit `*Handler` implementation on macOS 27 can both satisfy it. Thread stable file IDs and monotonic mtimes through the core: every macOS client cache in this option space keys on them.

## Verify on real hardware before committing

1. FUSE-T ≥1.2.7 `-backend=smb`: an `inval_inode` round-trip (remote change → visible without remount, bounded latency), the v1 cross-client Test-5 scenario, overwrite-rename atomicity, concurrent-write stress ([#45](https://github.com/macos-fuse-t/fuse-t/issues/45)), AppleDouble noise ([#81](https://github.com/macos-fuse-t/fuse-t/issues/81)), Finder credential-prompt behavior, xattr behavior with xattrs disabled-by-default since 1.0.46.
2. `fuser` 0.17 against FUSE-T's libfuse shim on macOS 13/14/15/26 (v1 runs 0.16; confirm the rename path — the macFUSE rename-ABI issue should not apply to FUSE-T, verify).
3. FUSE-T commercial license terms and cost for v2 bundling (required by [License.txt](https://raw.githubusercontent.com/macos-fuse-t/fuse-t/main/License.txt); presumably already in place for v1 — confirm).
4. macOS 27 beta spike: `FSPathURLResource` + `DataCacheHandler` invalidation semantics, `mountSingleVolume` + whether the `com.apple.developer.fskit.mount` entitlement is grantable, and mmap/UBC behavior for path-resource volumes (undocumented).
5. File a fuse-t upstream issue for the v1 overwrite-rename divergence — no matching report exists.

## Unverified items

- Whether `fuser` 0.17.0 works end-to-end against macFUSE 5.x (macOS marked "untested" upstream) — moot under the recommendation, load-bearing only if macFUSE is reconsidered.
- App Store availability of the `com.apple.developer.fskit.mount` entitlement (macOS 27 beta, undocumented).
- mmap/UBC semantics for FSKit path-resource volumes pre-macOS 27.
- fuse-t [#76](https://github.com/macos-fuse-t/fuse-t/issues/76) closure rationale (thread shows no visible resolution).
- No explicit maintainer wontfix exists for fuse-t NFS invalidation — the wiki statement is the authoritative record.
