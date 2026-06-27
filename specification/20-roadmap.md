# 20 — Roadmap (Desktop)

Phase order mirrors the server ([server 15](https://github.com/Nyxite/server)) so the client and backend land each capability together. v1.0.0 is the complete E2EE desktop client (Phases 0–6); phases are build order, not separate products. **E2EE is foundational from Phase 0** and never retrofitted.

Before Phase 1 logic is committed, run the **early spikes** ([18 §18.8](18-build-ci-testing.md)): ydotnet interop, `Nyxite.Crypto`/`Nyxite.Crdt` packaging across RIDs, SQLCipher-on-desktop, OS-keystore + app-lock, ink input, SignalR relay.

## Phase 0 — Foundations
**Deliver**: app shell + navigation + single-instance + tray ([15](15-ui-and-navigation.md)); **account-scoped DI + per-account SQLCipher DB/keys/cache** as the multi-account foundation ([01 §1.8](01-architecture.md), [04 §4.1](04-local-data-model.md), [14 §14.7](14-authentication.md)); Keycloak OIDC + PKCE login (system browser + loopback/`nyxite://`) with TOTP ([14](14-authentication.md)); identity keypair generation + device enrollment + recovery-key flow ([07](07-key-and-device-management.md)); key directory publish/lookup; OS-keystore + SQLCipher DB ([04](04-local-data-model.md), [17](17-security.md)); structure CRUD with **encrypted names** decrypted on-device ([05](05-api-client.md)); crypto engine via shared `Nyxite.Crypto` with conformance vectors ([06](06-cryptography.md)).
**Done when**: a user can log in, enroll the device, set a recovery key, recover on a second device, and browse/create the encrypted project/folder/file structure offline-first; crypto KATs + cross-client vectors pass.

## Phase 1 — Notes that sync (single user)
**Deliver**: markdown + plaintext editors with view/edit modes (AvaloniaEdit + Markdig, [10](10-editors.md)); encrypted CRDT backbone with offline catch-up (relay optional here) ([08 §8.4](08-sync-engine.md), [09](09-realtime-collaboration.md)); ciphertext blob sync + **full-corpus prefetch** with opt-out subtrees and the server sync policies (`server-default`/`excluded`) ([16](16-offline-and-storage-policies.md)); **full-corpus** on-device FTS search ([11](11-search.md)).
**Done when**: a single user edits notes on the desktop, they sync to another device through the encrypted relay/REST, offline edits reconcile, and full-corpus search works locally.

## Phase 2 — Collaboration & sharing
**Deliver**: live multi-client relay collaboration with presence/awareness via the SignalR .NET client ([09](09-realtime-collaboration.md)); account shares (HPKE-wrapped FKs) + link/guest shares (URL-fragment keys) ([13](13-sharing.md)); guest mode (in-memory, no account DB); rotation-based revocation ([07 §7.6](07-key-and-device-management.md)); version history with **client-side diffs** + restore ([12](12-version-history.md)); **reference snapshotting** ([09 §9.6](09-realtime-collaboration.md)).
**Done when**: two users (and an anonymous guest via link) co-edit a document live; revoking a share cuts off access and rotates the key; history diffs and restore work entirely on-device. CRDT conformance with web/Android passes.

## Phase 3 — Handwriting
**Deliver**: pen/stylus ink editor with pressure where available + low-latency Skia rendering ([10 §10.4](10-editors.md)); the shared encrypted ink vector format, co-designed with Android ([10 §10.5](10-editors.md), [19 §19.4](19-open-questions.md)); LWW/version-vector ink sync as encrypted blobs ([08 §8.5](08-sync-engine.md)).
**Done when**: handwritten notes capture faithfully, store encrypted, sync via LWW (losers retained in history), and round-trip with the Android ink format.

## Phase 4 — Polish & config
**Deliver**: rich per-user/per-file client-encrypted settings ([05](05-api-client.md)); device/key management UI (revoke device, re-issue recovery, identity rotation) ([07](07-key-and-device-management.md)); **multi-account switcher UI** (add/switch/remove accounts, per-account instance host) over the Phase-0 account scoping ([14 §14.7](14-authentication.md), [15 §15.1](15-ui-and-navigation.md)); **plaintext export / external-editor interop** ([16 §16.7](16-offline-and-storage-policies.md)); storage/budget controls ([16 §16.3](16-offline-and-storage-policies.md)); security settings (app lock, secure window); accessibility pass ([15](15-ui-and-navigation.md), [17](17-security.md)); packaging + auto-update for all targets ([18 §18.4](18-build-ci-testing.md)).
**Done when**: settings/config are complete and encrypted; export round-trips safely; security and accessibility meet the bar; signed installers ship for Windows + Linux.

## Phase 5 — Format expansion (optional in v1.0.0 scope)
**Deliver**: office docs, source-code text types (CRDT + AvaloniaEdit syntax highlighting), and images as encrypted blobs; chunked/resumable upload for large binaries ([05 §5.6](05-api-client.md), [16 §16.5](16-offline-and-storage-policies.md)). Any processing (thumbnails/extraction) is client-side.

## Phase 6 — Advanced hardening (optional)
**Deliver**: key transparency / safety-number verification UI for the key directory ([13 §13.6](13-sharing.md)); optional metadata-graph hiding support if the server adds it.

> **Multi-account / instance switching is in v1.0.0** (foundation in Phase 0, switcher UI in Phase 4), not deferred — see [14 §14.7](14-authentication.md). **Full-corpus search** is a Phase-1 deliverable and a defining desktop role.

## Cross-cutting / later
- Samsung Notes `.sdoc` import (separate, non-trivial migration item — proprietary ink format).
- macOS support ([19 §19.8](19-open-questions.md)).
- Resilience to a Redis-backplane multi-node relay (transparent to the client; hub contract unchanged).

## Versioning
SemVer for the app; pre-1.0 milestone tags track phases. The API is consumed at `/api/v1`; the crypto frame and CRDT wire protocol are pinned by conformance vectors so primitives/keys can rotate without a format break ([06 §6.3](06-cryptography.md), [18 §18.6](18-build-ci-testing.md)).
