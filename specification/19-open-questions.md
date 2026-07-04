# 19 — Open Questions (Desktop-Specific)

Desktop-specific open items, each with a **recommended resolution** and, where needed, a **validation spike**. These link back to the master tracker — the canonical [`docs/OPEN-DECISIONS.md`](https://github.com/Nyxite/Nyxite) — and must not restate or contradict its ratified decisions. Items already settled for this spec are listed in [§19.0](#190-ratified-for-this-spec) so they are not re-litigated.

## 19.0 Ratified for this spec

These were decided during specification and are treated as fixed unless a spike invalidates them:

- **UI framework**: Avalonia 11 on .NET 9, Windows + Linux (macOS deferred, reachable later — [00 §0.5](00-overview.md)).
- **MVVM stack**: **CommunityToolkit.Mvvm by default; ReactiveUI in the editor/collaboration/search screens** ([01 §1.3](01-architecture.md), [02 §2.2](02-tech-stack-and-libraries.md)).
- **Multi-account / instance switching in v1.0.0** — parity with Android; account-scoped DI + per-account encrypted DB/keys/cache/index ([14 §14.7](14-authentication.md)).
- **Packaging**: MSIX (+winget) on Windows; AppImage + Flatpak + `.deb` on Linux ([18 §18.4](18-build-ci-testing.md)).
- **Shared code consumption**: `Nyxite.Domain`/`Contracts`/`Crypto`/`Crdt` as **NuGet packages** from the `server` repo, per the master README; crypto/CRDT are not re-implemented ([README](README.md), [06](06-cryptography.md)).
- **Keep-on-device default**: **full corpus** (opt heavy subtrees out), the inverse of mobile ([16 §16.2](16-offline-and-storage-policies.md)).
- **Plaintext export / external-editor interop**: in scope, desktop-only, opt-in, warned ([16 §16.7](16-offline-and-storage-policies.md)).

## 19.1 ydotnet binding maturity *(maps to the master "CRDT bindings" validation item)*

- **Question**: Is ydotnet mature/complete enough to run the full client-side merge and snapshotting, and does it interop byte-for-byte with Yjs (web) and yrs/UniFFI (Android)?
- **Why it matters**: it is the highest-risk client dependency; the desktop is also the reference snapshotter ([09 §9.6](09-realtime-collaboration.md)).
- **Recommended resolution**: adopt ydotnet via the shared `Nyxite.Crdt` glue; **gate adoption on the Phase-1 spike** passing the shared conformance vectors. **Fallback**: a thin own binding over `yffi` (P/Invoke or a small Rust shim) if ydotnet is inadequate.
- **Spike** ([18 §18.8](18-build-ci-testing.md)): replay shared Yrs wire vectors; soak-test a live three-client (desktop/web/android) session; benchmark snapshot/merge on large docs.

## 19.2 On-device key storage & app-lock factor *(maps to master desktop open question)*

- **Question**: DPAPI (Windows) and Secret Service (Linux) unseal on OS login and have no hardware-isolated, attestable equivalent to mobile StrongBox/Secure Enclave. What protects keys beyond mere OS login, and what is the fallback where no Secret Service exists?
- **Recommended resolution**: wrap the DB master key + identity store with the OS keystore, **plus** an **app-lock factor** (Windows Hello / OS credential / Argon2id-stretched app passphrase) with a **default 10-minute idle window** (1–60 min / every-launch) as the real at-rest defense ([07 §7.2](07-key-and-device-management.md), [17 §17.3](17-security.md)). On Linux desktops with no Secret Service, fall back to a passphrase-derived wrap, surfaced clearly.
- **Spike**: Secret Service availability across target Linux desktops + headless fallback UX; DPAPI under roaming profiles; whether Windows Hello / `libfido2` can strengthen the at-rest wrap.

## 19.3 Offline LWW conflict handling for ink/binary on reconnect *(master desktop open question)*

- **Question**: How does the desktop reconcile concurrent ink/binary edits made offline when it reconnects?
- **Recommended resolution**: the version-vector + `If-Match` + LWW-with-loser-retained model in [08 §8.5](08-sync-engine.md); concurrent edits surface non-destructively in history ([12 §12.5](12-version-history.md)) with keep-both/restore. No silent loss. (Same model as Android; resolved by the master per-file-type CRDT/LWW split.)
- **Spike**: simulate two devices editing the same ink page offline, reconnecting in both orders; assert both versions are retained and surfaced.

## 19.4 Ink capture & the shared encrypted vector stroke format *(master desktop + android open question)*

- **Question**: The on-disk/on-wire ink format must be **defined once and shared with Android**; the exact field schema is still `[P]`.
- **Recommended resolution**: a versioned, **deterministic CBOR** format ([10 §10.5](10-editors.md)) co-designed by the desktop and Android teams and specified in a shared document; both clients conform exactly and share round-trip test vectors. Desktop captures pressure where the OS/driver exposes it and degrades gracefully ([10 §10.4](10-editors.md)).
- **Spike**: pen pressure/tilt across Windows Ink / libinput / Wacom; CBOR round-trip + stable BLAKE3 address; cross-client open of a note authored on the other client.

## 19.5 Snapshot/compaction policy (thresholds, who snapshots) *(master desktop open question)*

- **Question**: When and by whom are encrypted CRDT snapshots produced?
- **Recommended resolution**: triggers **≥200 updates / 5-minute / last-participant-leaving**; any participant may snapshot, but a **desktop in the room preferentially snapshots** to spare lighter clients (soft preference, not required for correctness) ([09 §9.6](09-realtime-collaboration.md)). Validate the thresholds against real editing traffic and tune.
- **Spike**: measure update volume vs snapshot size/frequency in a realistic session; confirm the server prunes pre-snapshot updates safely with in-flight clients.

## 19.6 Full-text index technology & incremental update at full-corpus scale *(master desktop open question)*

- **Question**: Does SQLite FTS5 inside SQLCipher hold up indexing the **whole corpus** with incremental updates as content syncs in?
- **Recommended resolution**: FTS5 (`unicode61` + `porter`) inside the per-account SQLCipher DB, updated incrementally on each decrypt ([11](11-search.md)); rebuildable via `ReindexService`. Validate index size and query latency at large corpus sizes; if FTS5 is insufficient, evaluate an encrypted external index — but only if it leaks nothing (a Phase-6-style concern).
- **Spike**: index a synthetic large corpus (tens of thousands of notes); measure DB size, incremental upsert cost, and query latency.

## 19.7 NativeAOT vs. self-contained single-file

- **Question**: Can the app ship as NativeAOT (faster start, smaller, harder to tamper) given native interop (SQLCipher, ydotnet, BLAKE3, libsecret) and Avalonia/reflection needs?
- **Recommended resolution**: **default to self-contained single-file**; treat NativeAOT as an optimization validated per RID. Don't block v1.0.0 on it.
- **Spike**: AOT-publish a vertical slice exercising crypto + CRDT + SQLCipher + Avalonia; verify trimming keep-directives.

## 19.8 macOS (not planned)

- **Question**: When/whether to add macOS.
- **Resolution (DECIDED)**: **Not planned.** The canonical surface list is Windows + Linux, and macOS is not a target. It would only be reconsidered **if the team gains access to Apple hardware** to develop and test on. The Avalonia + shared-library architecture does not preclude it — a macOS Keychain `IKeyStoreVault` and `.app`/notarization would be the work — but nothing macOS-specific is designed, built, tested, or roadmapped. This is not a deferred-but-planned item; it is off the roadmap.

## 19.9 Background presence & multi-account sync scope

- **Question**: Should full-corpus prefetch and relay presence run for **non-active** accounts in the background (tray)?
- **Resolution (DECIDED — tiered)**: the **active account** is fully synced (full-corpus prefetch + relay); **recently-used accounts run structure/delta sync only**, with heavy full-corpus prefetch **deferred until the account becomes active**; all of it tied to the app-lock window ([17 §17.3](17-security.md)). This bounds background resource use while keeping the structure tree fresh on switch. The exact periodicity/“recently-used” horizon is tuned from real usage, but the tiered model is fixed.

## 19.10 Group-management & reader-group-attachment UX *(maps to master "reader-group attachment UX and cascade semantics")*

- **Question**: How does the desktop present group creation/enrollment/removal and the **reader-group attachment** cascade so that a non-expert conveys the intended visibility — especially the enterprise "manager reads all" auto-wrap ([13 §13.7](13-sharing.md), [16 §16.8](16-offline-and-storage-policies.md))? How is the transparency-verified-enrollment result and the forward-secrecy vs. "already-decrypted can't be recalled" caveat surfaced without overwhelming the user?
- **Why it matters**: an attachment set at the wrong scope silently over- or under-shares every *future* file in a subtree; a misread enrollment fingerprint could admit a substituted key (the whole-corpus exposure that motivates the Phase-4.3 dependency).
- **Recommended resolution**: reuse the existing cascade UI patterns of keep-on-device / plaintext-at-rest (tri-state `inherit`/group/none per project/folder/file, [16 §16.8](16-offline-and-storage-policies.md)) with an explicit "new files here are also readable by <group>" affordance; show the enrollee's key fingerprint **and** the key-transparency proof status inline at enroll; carry the §13.5 honest-revocation caveat into the removal dialog. Validate comprehension with usability testing.
- **Spike**: usability-test the attachment cascade and enrollment flow; confirm users predict who can read a file created in an attached subtree, and correctly interpret the rotation caveat.

## 19.11 Re-enrollment flow for a member who lost all devices + recovery phrase *(maps to master "re-enrollment flow")*

- **Question**: When a member loses **every device and the recovery phrase**, group access cannot self-recover (no-escrow model, [07 §7.4](07-key-and-device-management.md)). What is the desktop UX for a **group admin** to re-enroll them (issue one new grant to the member's new identity public key), and how is it distinguished from ordinary enrollment?
- **Why it matters**: it is the only group-specific recovery seam ([07 §7.10](07-key-and-device-management.md)); done wrong it either strands the member or re-wraps to an unverified key.
- **Recommended resolution**: treat re-enrollment as ordinary enrollment against the member's **new, transparency-verified** identity key — one new grant, no special server path — but flag in the admin UI that this is a *re-enrollment* (prior grants for the old key remain in the append-only history, auditable) and require the same fingerprint + transparency-proof confirmation as a first enroll. Personal *content* the lost member solely authored under their own key remains unrecoverable; the group grant only restores group-readable content.
- **Spike**: walk an admin through re-enrolling a member on a fresh identity key; confirm group access returns with one grant and that the flow refuses an unverified key.
