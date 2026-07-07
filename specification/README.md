# Nyxite Desktop — Specification (v1.0.0)

This folder is the detailed, implementation-level specification for the **Nyxite Desktop client** — the native **Avalonia (C# / .NET)** application for **Windows and Linux** that lets a user read, write, hand-draw, organize, sync, collaborate on, share, and search end-to-end-encrypted notes and documents from a desktop computer.

It expands the architectural planning documents in the central [`Nyxite`](https://github.com/Nyxite/Nyxite) repository and the [`server` specification](https://github.com/Nyxite/NyxiteServer) into a concrete client build specification covering the full v1.0.0 product across all roadmap phases.

It is written to be **self-contained enough to build the app from**: it names the libraries, the project/solution graph, the local schema, the API/relay calls, the crypto primitives and how they map to .NET APIs, the editor and sync internals, the UI surface, and the build/test/CI/packaging setup.

## Guiding principle: privacy first (full E2EE)

**Nyxite is end-to-end encrypted everywhere.** Encryption, decryption, content addressing, CRDT merge, **snapshotting**, search indexing, and diffing all happen **on the client**. The server only ever sees ciphertext, opaque IDs, the structure graph, ACL grants, wrapped-key blobs, sizes and timestamps — it never holds a content key and cannot read notes, ink, names, or collaboration traffic. The desktop client is therefore not a thin viewer: it is a **full cryptographic peer** that holds the user's keys, performs all crypto locally, and treats the server as a blind relay/store.

The desktop is the **most capable** of the three clients. With room for the full local corpus it is the **primary full-text search surface** and the reference for "search everything," and — because the server cannot compact an encrypted log — it is a primary producer of the **encrypted CRDT snapshots** that feed version history. Where the browser is the most constrained client and Android is bounded by mobile storage/battery, the desktop is the one that can hold and index everything.

## What "shares C# with the server" means here

Unlike the Android (Kotlin) and Web (TypeScript) clients, which **reimplement** the crypto framing and CRDT glue in their own language and prove interoperability only through conformance vectors, the desktop client is C# and **consumes the very same shared libraries the server is built from**, published as NuGet packages from the `server` repo ([master `docs/LICENSING-AND-REPOS.md`](https://github.com/Nyxite/Nyxite)):

- **`Nyxite.Domain`** — domain entities, enums, value types.
- **`Nyxite.Contracts`** — REST/relay DTOs and the `problem+json` error contract.
- **`Nyxite.Crypto`** — the canonical AES-256-GCM framing, HPKE wrap/unwrap, Ed25519/X25519, BLAKE3 addressing, Argon2id recovery KDF.
- **`Nyxite.Crdt`** — the ydotnet/Yrs glue, snapshot/state-vector helpers, and the wire-protocol surface that also backs the server-side `Nyxite.CrdtConformanceTests` harness (a test harness, **not** live server-side merging).

The desktop therefore gets byte-for-byte crypto and CRDT compatibility **by construction**, not by re-derivation. It still runs the same cross-client conformance vectors in CI ([18](18-build-ci-testing.md)) so that a protocol change can never silently diverge from web/Android.

## Source of truth & convention

- The central `Nyxite` repo is authoritative for product decisions; the `server/specification` set is authoritative for the wire protocol, schema, crypto model, and API. This spec **consumes** those and must not contradict them. Where it picks a concrete desktop-side mechanism the server spec left open, it is marked **[P]** (Proposed), exactly as the server and Android specs use the tag.
- **[OD-n]** references a numbered item in the master `docs/OPEN-DECISIONS.md`.
- Open questions specific to desktop are tracked in [19-open-questions.md](19-open-questions.md) and link back to the master open-decisions list.

## Documents

| # | Document | Covers |
|---|----------|--------|
| 00 | [overview.md](00-overview.md) | Purpose, scope, target platforms, actors, glossary, phase map |
| 01 | [architecture.md](01-architecture.md) | Layered/Clean architecture, MVVM, data flow, threading, hosting |
| 02 | [tech-stack-and-libraries.md](02-tech-stack-and-libraries.md) | Concrete library choices with versions and rationale |
| 03 | [project-structure.md](03-project-structure.md) | Solution/project graph, namespaces, naming |
| 04 | [local-data-model.md](04-local-data-model.md) | EF Core + SQLite/SQLCipher schema, encrypted-at-rest DB, sync state, FTS |
| 05 | [api-client.md](05-api-client.md) | REST client, DTO mapping, error/retry, idempotency, TLS |
| 06 | [cryptography.md](06-cryptography.md) | AEAD, HPKE, Ed25519/X25519, BLAKE3, Argon2id, framing, shared `Nyxite.Crypto` |
| 07 | [key-and-device-management.md](07-key-and-device-management.md) | Identity keypair, device enrollment, recovery key, OS keystore (DPAPI/Secret Service) |
| 08 | [sync-engine.md](08-sync-engine.md) | Manifest/delta sync, policies, CRDT/LWW split, hosted background services |
| 09 | [realtime-collaboration.md](09-realtime-collaboration.md) | SignalR relay, ydotnet/Yrs merge, awareness, guests, snapshots |
| 10 | [editors.md](10-editors.md) | Markdown, plaintext, and ink editors; view/edit modes |
| 11 | [search.md](11-search.md) | **Full-corpus** on-device FTS; the platform's complete search surface |
| 12 | [version-history.md](12-version-history.md) | Snapshot fetch, client-side diff, restore |
| 13 | [sharing.md](13-sharing.md) | Account shares (HPKE), link shares (URL fragment), revocation |
| 14 | [authentication.md](14-authentication.md) | Native auth (password + TOTP, passkeys) by default; enterprise Keycloak OIDC + PKCE pluggable; token storage, refresh, multi-account |
| 15 | [ui-and-navigation.md](15-ui-and-navigation.md) | Windows, navigation, Avalonia/Fluent, accessibility, tray |
| 16 | [offline-and-storage-policies.md](16-offline-and-storage-policies.md) | Full-corpus keep, convenience cache, excluded, **plaintext export / external-editor interop** |
| 17 | [security.md](17-security.md) | Desktop threat model, at-rest protection, app lock, secure handling |
| 18 | [build-ci-testing.md](18-build-ci-testing.md) | Solution build, packaging (MSIX/AppImage/Flatpak/deb), CI, test strategy, conformance |
| 19 | [open-questions.md](19-open-questions.md) | Desktop-specific open items with recommended resolutions + validation spikes |
| 20 | [roadmap.md](20-roadmap.md) | Desktop phase mapping aligned to the server roadmap |

## Status

Specification for a greenfield build. No desktop code exists yet; the `desktop` repo currently holds only `FEATURES.md` and `LICENSE.md`. This document set defines what to build. (`FEATURES.md` is a graduated copy of the canonical `Nyxite/features/desktop.md` and is kept in sync with it; where they ever disagree, the master file and this spec win. Note that `pinned-local` is **not** a sync policy — it is the client-local keep-on-device control, [16 §16.2](16-offline-and-storage-policies.md).)

## License

PolyForm Noncommercial License 1.0.0 — see the repo `LICENSE.md`.
