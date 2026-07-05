# NyxiteDesktop — Implementation Readiness Report

**Question:** Can NyxiteDesktop be **fully implemented across all phases (0–6)** using only (a) the `NyxiteDesktop/` repo and (b) the shared `Nyxite/` info repo?

**Verdict: NO.** The desktop `specification/` is unusually complete and internally consistent — for the large majority of surfaces (architecture, data model, crypto framing, sync engine, collaboration, editors, search, version history, sharing, auth, UI, security, packaging) there is more than enough to build against. But there are **6 hard blockers** where required information is either **missing, contradictory, or explicitly still open**, plus several minor ambiguities. The blockers are concentrated at the seams with the *server* repo (whose source/packages are a third repo not provided here) and in one feature that the per-surface spec actively contradicts.

Sources read: all of `NyxiteDesktop/specification/00–20` + `FEATURES.md`; `Nyxite/docs/SPECIFICATION.md`, `OPEN-DECISIONS.md`; `Nyxite/features/desktop.md`, `groups.md`; every `Nyxite/implementation/phase-*.md` (all `-DSK-` and cross-cutting `-CORE-` steps + acceptance test cases) and `implementation/README.md`.

---

## Hard blockers

### HB-1 — Plaintext-at-rest (desktop-only, tri-state) is unspecified in, and contradicted by, the desktop spec
**Phase/step blocked:** Phase 1.2 — `P1.2-DSK-1` ("tri-state plaintext-at-rest (`inherit`/`plaintext`/`encrypted`)…") and acceptance test `P1.2-TC-4`. Also touches Phases 2.4 (history stays encrypted) and 4.5 (settings UI).

**What is missing / wrong:**
- This is a **first-class, ratified desktop feature** — `FEATURES.md`, `Nyxite/features/desktop.md` (Sync section), `OPEN-DECISIONS.md` **#10**, `SPECIFICATION.md` §6/§15, and the build plan (`phase-1.2-storage-control-and-search.md`, deliverable and `P1.2-TC-4`) all require it.
- Yet the desktop `specification/` **never introduces it and asserts the opposite invariant**: `16 §16.2` says "keep on device controls *availability*, not whether it's encrypted"; `16 §16.1`/`17 §17.2` say kept content "stays **encrypted at rest**." There is no `specification/` section for the tri-state toggle at all.
- Concrete desktop mechanics needed to build it, absent from **both** repos:
  - **Data model:** `specification/04` defines `KeepOnDevice`/`KeepOnDeviceDefault` (and mentions the reader-group cascade) but **no `PlaintextAtRest` tri-state column** on `FileEntity`/`FolderEntity`/`ProjectEntity`, nor where it lives (OPEN-DECISIONS only says "desktop-local settings"). The cascade-resolution interaction with the *two* existing cascades (keep-on-device, reader-group attachment) is unreconciled.
  - **CRDT interaction (the hard part):** SPECIFICATION §6 says a plaintext scope keeps "only the **minimal CRDT/sync state needed to merge**," but the entire text-sync model (`08`, `09`) assumes the local source of truth is the SQLCipher DB + encrypted blobs + in-memory Yrs doc. How a plaintext-on-disk working copy (which *replaces* the local copy) round-trips through the Yrs/relay path, and what "minimal CRDT/sync state" is persisted, is specified nowhere.
  - **FTS interaction:** is a plaintext-on-disk file still indexed in the encrypted FTS5 DB? Unspecified (`11`, `04 §4.3` assume DB-resident plaintext only).
  - **Integrity backstop:** "integrity-checked on read against the content hash in metadata" (SPECIFICATION §15) — for a CRDT text file the content hash is BLAKE3 of the *plaintext head*; the read/verify/restore-from-server-ciphertext flow is not detailed on the desktop side.
  - **UI:** `specification/15` Settings/Storage rows list keep-on-device, cache, security, export — but **not** a plaintext-at-rest toggle.

**What I'd need:** a desktop `specification/` section (analogous to `16 §16.2`) defining the tri-state field + storage location, its cascade and precedence vs keep-on-device, the plaintext working-copy ↔ Yrs/CRDT sync-state mechanics, FTS handling, the integrity-check-and-restore flow, and the settings UI — and a correction of the "always encrypted at rest" statements in `16`/`17`.

### HB-2 — Exact public APIs of the shared `Nyxite.*` packages are not available
**Phase/step blocked:** effectively all phases; first bites at `P0.1-DSK-2` (crypto engine), `P0.1-CORE-4`, `P1.1-DSK-1/2` (ydotnet binding), collaboration (`09`).

**What is missing:** The desktop consumes `Nyxite.Crypto`, `Nyxite.Crdt`, `Nyxite.Contracts`, `Nyxite.Domain` **as NuGet packages published from the `server` repo** (`specification/03`, `02 §2.7`, `19 §19.0`). That server repo is neither of the two provided here. The specs give the *conceptual* contract (frame layout, HPKE suite IDs, `ICryptoEngine`/`ICrdtEngine` adapter shapes the desktop defines itself) but **not the concrete signatures the desktop must compile against**: e.g. `FileKeyHandle`, `EncryptedUpdate { seq, ciphertext, keyId }`, `PresenceDto`, the ydotnet/`Nyxite.Crdt` engine surface (update/state-vector/snapshot APIs), and the `Nyxite.Contracts` DTO field names. Without these the network/crypto/CRDT projects cannot be written to compile.

**What I'd need:** the published package API surface (or the `server` repo source) for `Nyxite.Crypto`/`Crdt`/`Contracts`/`Domain`.

### HB-3 — Concrete server REST + SignalR contracts (OpenAPI schemas, sync payload shapes, hub protocol) are server-owned and not pinned
**Phase/step blocked:** `P0.2-DSK-1`, `P1.1-DSK-2`, `P1.2-DSK-1/2`, `P2.x-DSK-*` (sharing, rotation), `P2.4-DSK-1`.

**What is missing:**
- `specification/05` enumerates endpoints and which fields are ciphertext vs JSON, but **not full request/response JSON schemas**. The canonical contract is the server's `/openapi/v1.json` (`05 §5.2`), which is not in either repo.
- The **`GET /sync/manifest` and `POST /sync/changes` payload shapes and per-kind `ref` formats are explicitly still open**: `OPEN-DECISIONS.md` ("Tracked for implementation") states they are "**server-owned**; clients code defensively and lock them via shared conformance vectors when the server pins them. **Resolves during Phase 1**." So by the plan's own admission they are not yet defined.
- The SignalR `RelayHub` **hub protocol is unpinned on the desktop side**: `09 §9.2` says "Configure the **MessagePack or JSON** hub protocol **to match the server's hub**" — the actual choice and message encoding live in the server repo.

**What I'd need:** the server OpenAPI document, the frozen sync manifest/delta payload + `ref` formats, and the pinned relay hub protocol/encoding.

### HB-4 — Shared encrypted ink vector stroke format: exact field schema is still `[P]` / pending co-design
**Phase/step blocked:** Phase 3.1 — `P3.1-CORE-1`, `P3.1-DSK-1/2`, tests `P3.1-TC-2/4/5`.

**What is missing:** `specification/10 §10.5` and `19 §19.4` both mark the format `[P]` and "co-designed with and shared with Android," "the exact field schema is still `[P]`." `OPEN-DECISIONS.md` ("Tracked for implementation → Shared ink stroke format") confirms the "remaining work is desktop + Android co-design of the exact field schema plus cross-client round-trip vectors." The high-level shape (deterministic CBOR, pages→strokes→samples `(x,y,pressure,tilt,orientation,tRelMs)` + brush metadata, byte-stable BLAKE3) is given, but the **exact field/CBOR schema and the round-trip vectors are not defined in either repo** — and byte-stability across clients requires them exactly. Ink serialization cannot be implemented to conform without this.

**What I'd need:** the finalized shared ink-format specification document + cross-client round-trip vectors.

### HB-5 — Native passkey/WebAuthn ceremony on desktop has no specified library or platform-API path
**Phase/step blocked:** Phase 0.1 — `P0.1-DSK-4` ("Login (password+TOTP **or passkey**)"), test `P0.1-TC-3`.

**What is missing:** For OIDC the spec names a concrete library (`IdentityModel.OidcClient`, `02 §2.6`), but for **native WebAuthn/passkey** it only says "a WebAuthn/passkey integration for register/assert ceremonies" (`02 §2.6`, `14 §14.1`) with **no library, no platform-API approach** (Windows Hello / Windows WebAuthn API vs `libfido2` on Linux) and no handling of the register/assert message flow against the server's relying-party challenge (that RP protocol lives in the server repo). Password+TOTP is fully buildable; the passkey half of the "co-equal primary method" is under-specified for a native Avalonia app.

**What I'd need:** a chosen desktop WebAuthn approach/library per OS and the server RP challenge/assertion contract.

### HB-6 — Key-transparency inclusion-proof format & safety-number derivation are "to be defined," yet the desktop must verify them
**Phase/step blocked:** Phase 4.3 — `P4.3-DSK-1`; and as a hard dependency, Phase 4.4 group enrollment — `P4.4-DSK-1` (transparency-verify a member's key before wrapping), tests `P4.3-TC-1/2`, `P4.4-TC-3`.

**What is missing:** `phase-4.3-key-transparency.md` `P4.3-CORE-1` says the job is to **"Define** safety-number derivation … **and the transparency-log inclusion-proof format**" and ship cross-client vectors — i.e. these artifacts do **not yet exist** in either repo. The desktop spec (`06 §6.7`, `07 §7.10`, `13 §13.6`) references verifying inclusion proofs and computing safety numbers but gives **no algorithm/wire format**. Group enrollment (a v1.0.0 feature, decision G-3) refuses to wrap to an unverified key, so this blocks 4.4 too. Cannot implement proof verification against an undefined format.

**What I'd need:** the finalized safety-number derivation and transparency-log inclusion/consistency-proof format + shared vectors.

---

## Minor ambiguities (not blockers — resolvable with the recommended resolutions given, or via a scoped spike)

- **Group key scope granularity (project vs time-period) + scope-id resolution rule.** `groups.md`, OPEN-DECISIONS G-4, and `phase-4.4` leave "project-vs-time granularity" a "deferred design sub-choice." `scope_kind` exists, but **how a given file resolves to its scope's group key** on wrap/unwrap (`P4.4-DSK-1/2`) is not pinned. Buildable behind an abstraction, but the concrete rule is undecided. (Phase 4.4)
- **ydotnet maturity.** `19 §19.1`/`09 §9.10` — highest-risk dependency, but a recommended resolution (adopt via `Nyxite.Crdt`) **and** a pre-planned fallback (own `yffi` binding) are given, gated on the Phase-1 spike. Implementable either way.
- **SQLCipher `bundle_e_sqlcipher` FTS5 + arch coverage** (`02 §2.5`, `18 §18.8`), **DPAPI roaming / Secret Service headless fallback UX** (`07 §7.2`, `19 §19.2`), **FTS5 at full-corpus scale** (`19 §19.6`), **NativeAOT vs single-file** (`19 §19.7`) — each has a recommended resolution and is an early-validation spike, not an information gap.
- **Snapshot thresholds** (`09 §9.6`: ≥200 updates / 5 min / last-leaver) are pinned; only tuning is deferred.
- **Group/reader-attachment UX comprehension** (`19 §19.10`) and **lost-everything re-enrollment flow** (`19 §19.11`) — recommended resolutions given; usability-validation, not missing spec.
- **Auth-chapter reconciliation** flagged in `implementation/README.md` does **not** affect desktop: `specification/14` already reflects native-auth-default with enterprise Keycloak as the pluggable option (decision #9).

---

## Phase-by-phase summary

| Phase | Desktop buildable from these two repos? | Blocking gaps |
|---|---|---|
| 0.1 Account & identity | Mostly — shell, DI, SQLCipher, keystore, password+TOTP, enrollment, recovery all specified | HB-2 (shared pkg APIs), HB-5 (passkey), HB-3 (server contracts) |
| 0.2 Encrypted structure | Yes, conceptually | HB-2, HB-3 (structure DTOs/OpenAPI) |
| 1.1 Notes that sync | Yes, conceptually | HB-2 (ydotnet/CRDT surface), HB-3 (crdt/blob contracts) |
| 1.2 Storage & search | **No** | **HB-1 (plaintext-at-rest)**, HB-3 (manifest/delta shapes) |
| 2.1 Live collaboration | Yes, conceptually | HB-2 (`EncryptedUpdate`/`PresenceDto`), HB-3 (hub protocol) |
| 2.2 Sharing / 2.3 Rotation / 2.4 History | Yes, conceptually | HB-2, HB-3 (share/keys/rotate/version DTOs) |
| 3.1 Handwriting | **No** | **HB-4 (ink format schema + vectors)** |
| 4.1 Device/key & multi-account | Yes | HB-2, HB-3 |
| 4.3 Key transparency | **No** | **HB-6 (proof/safety-number format)** |
| 4.4 Groups | **No** | **HB-6**, plus scope-granularity ambiguity |
| 4.5 Polish & distribution | Yes (export/external-editor & packaging well-specified) | — |
| 5.1–5.3 (optional) office/source/images | Yes, conceptually | HB-2, HB-3 (content-type enum, chunked-upload contract) |
| 6.2/6.3 (optional) graph-hiding / searchable index | Design-stage by definition | Format/scheme to be designed (`P6.2-CORE-1`, `P6.3-CORE-1`) |

---

## Bottom line

The desktop spec is a strong, buildable blueprint for ~80% of the client. Full implementation of **all** phases is **not** possible from these two repos alone because of six hard blockers: one is a genuine **spec defect** (plaintext-at-rest is required by the global spec/plan but absent from and contradicted by the desktop spec — the single most important gap to fix, and it is desktop-owned), and five are **cross-repo seams** whose canonical artifacts live in the `server` repo or in still-open co-design items (shared package APIs, server REST/hub contracts, the ink format, native passkey mechanics, key-transparency proof formats). The remaining open items are spikes or ambiguities with recommended resolutions and can be built against.
