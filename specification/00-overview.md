# 00 — Overview

## 0.1 Purpose

The Nyxite Desktop client is a **native, end-to-end-encrypted notes and documents app** for Windows and Linux. It lets a user, from a desktop or laptop:

- Browse the project → folder → file hierarchy (names decrypted on-device).
- Read and edit **markdown**, **plain text**, and **handwritten ink** files, with distinct **view** and **edit** modes.
- Sync content across devices with per-file policies, **encrypting before upload and decrypting after download** — the server only moves ciphertext.
- Collaborate live with other users and anonymous guests via a **client-side CRDT merge** over an encrypted relay.
- Search its **full local corpus** of decrypted content — the desktop is the platform's complete search surface.
- Browse **version history** with client-computed diffs, and restore.
- Create and consume **share links** (key in URL fragment) and **account shares** (file key wrapped to a recipient's public key via HPKE).
- **Export decrypted working copies** and round-trip through external editors when the user explicitly chooses to ([16 §16.7](16-offline-and-storage-policies.md)) — a desktop-only capability.

It is built on **Avalonia (C# / .NET)** so a single codebase serves Windows and Linux while **sharing the server's C# crypto, CRDT, contract, and domain libraries** ([README](README.md)), giving it cross-client byte-compatibility by construction.

## 0.2 What makes this client different from a normal sync app

Because Nyxite is zero-knowledge, the desktop app carries responsibilities a typical client offloads to a backend:

- It **generates and holds the user's keys**; it performs **all** AES-256-GCM encryption/decryption and HPKE wrap/unwrap locally.
- It **computes content addresses** (BLAKE3 of plaintext) itself; the server stores the ciphertext under the client-supplied address and cannot verify it.
- It **runs the CRDT engine** (ydotnet/Yrs) and merges locally; the server never merges.
- It **builds and queries its own full-text search index** over the **entire** corpus; there is no server search.
- It **computes diffs** between decrypted snapshots; there is no server diff.
- It **produces the encrypted CRDT snapshots** that compact the update log and feed version history; the server cannot, since it can't read the log.
- It **drives key rotation** on share revocation for forward secrecy.

The server gives it: authenticated transport, durable ordered storage of ciphertext, an encrypted relay, an ACL gate, a key directory (public keys), and structure metadata. Everything cryptographic is the client's job. See [06-cryptography.md](06-cryptography.md) and [07-key-and-device-management.md](07-key-and-device-management.md).

## 0.3 Desktop's role across the platform

| Capability | Web (browser) | Android (mobile) | **Desktop** |
|---|---|---|---|
| Local corpus | Cached/session subset only | Local subset (keep-on-device + cache) | **Full corpus** (room to hold everything) |
| Search | Weakest (session subset) | Best-effort (local subset) | **Complete** — indexes everything available |
| Snapshotting | Can snapshot | Can snapshot | **Primary/reference snapshotter** |
| Ink | Pointer/canvas parity | S-Pen pressure/tilt (capture leader) | Pen/Wacom + mouse; shares the encrypted ink format |
| Plaintext export / external editor | No | No (explicitly deferred to desktop) | **Yes** — controlled working-copy export |

The desktop is the workhorse: the surface where a power user keeps the whole library offline, searches all of it, runs long-lived collaboration sessions, and snapshots aggressively.

## 0.4 Scope of v1.0.0

v1.0.0 is the **complete E2EE desktop client** spanning Phases 0–6 of the master roadmap, built in the same phase order as the server (see [20-roadmap.md](20-roadmap.md)). The key/device/recovery subsystem is foundational (Phase 0) and is never retrofitted later.

Explicitly **in scope**: markdown + plaintext + ink editing, the server sync policies (`server-default`/`excluded`) plus client-local keep-on-device, encrypted relay collaboration with guests, account + link sharing, rotation-based revocation, version history with client diffs/restore, **full-corpus** on-device search, Keycloak login with TOTP, device enrollment and recovery-key flows, on-device key storage via the **OS keystore** (Windows DPAPI / Linux Secret Service), **multiple accounts / instance switching** (per-account isolated storage and keys, [14 §14.7](14-authentication.md)), and **controlled plaintext export / external-editor interop** ([16 §16.7](16-offline-and-storage-policies.md)).

Explicitly **out of scope for v1.0.0** (deferred, matching server Phase 5–6 and the separate migration item): office-document and source-code content types, image attachments, chunked upload for very large binaries, key-transparency/safety-number verification, metadata-graph hiding, and Samsung Notes `.sdoc` import. These are noted where they touch the architecture so seams exist for them.

## 0.5 Target platforms & baseline

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Runtime | **.NET 9** (LTS-track; pin in CI) | Modern `System.Security.Cryptography` (AES-GCM), `System.Text.Json`, first-class SignalR client, single-file/self-contained publish. |
| UI | **Avalonia 11** | One C# UI codebase for Windows + Linux; custom-drawable surfaces for the ink canvas. |
| Windows | **Windows 10 1809+ / Windows 11** (x64, arm64) | DPAPI, modern TLS, MSIX packaging; Windows Hello optional for app lock. |
| Linux | **glibc desktops** (Ubuntu 22.04+ / Fedora / Arch class), x64 + arm64 | Secret Service (libsecret) via D-Bus; AppImage/Flatpak/deb packaging. |
| Input | Mouse, keyboard, **pen/stylus** (Wacom, Windows Ink, Linux libinput) | Ink capture with pressure where the OS/driver exposes it ([10 §10.4](10-editors.md)). |
| Form factor | Resizable desktop windows; multi-window | List/detail + multi-document; power-user layout. |
| Offline | First-class | E2EE + full local corpus means the app is fully usable with no network. |

macOS is **not planned** (the canonical surface list is Windows + Linux). It would only be reconsidered if the team gains access to Apple hardware to develop and test on; until then it is not a target. The Avalonia + shared-library architecture does not *preclude* it (a macOS Keychain `IKeyStoreVault` and `.app`/notarization would be the work), but no macOS support is designed, built, or tested for, and it is not on the roadmap.

## 0.6 Actors (as seen by the Desktop client)

| Actor | On Desktop |
|-------|-----------|
| **User** | Signs in via Keycloak (OIDC + TOTP), enrolls the device, holds the identity keypair, owns/edits/shares files. |
| **Guest** | This app acting on a link share: no Keycloak account, file key taken from the URL fragment, relay access via a short-lived share token. The app can both *open* an incoming link and *create* link shares. |
| **Peer** | Another user/guest editing the same document; seen through presence/awareness over the relay. |
| **Server** | Blind relay/store. The client treats every byte it sends as ciphertext the server cannot read. |
| **Keycloak** | External IdP for account auth and TOTP. |
| **External editor** | An OS app the user explicitly opens an exported plaintext working copy in ([16 §16.7](16-offline-and-storage-policies.md)); outside the encryption boundary by user choice. |

## 0.7 Glossary (client-facing)

- **FK (file key)** — per-file AES-256-GCM 256-bit key, generated on-device, stored on the server only *wrapped*.
- **Identity keypair** — per-user X25519 (HPKE/key-agreement) + Ed25519 (signing); private parts never leave the device.
- **Device key** — per-device keypair used for device-to-device enrollment approval.
- **Recovery key** — high-entropy user-held secret (phrase) that, via Argon2id, wraps an escrow of the identity private key; the only recovery path.
- **Wrapped key** — an FK encrypted to a member's X25519 public key via HPKE (account share).
- **Fragment key** — an FK carried in a share link's URL fragment (`#k=…`), never sent to the server (link/guest share).
- **Content address** — BLAKE3-256 hash of the *plaintext*, used as the blob's storage key; computed on-device.
- **Encrypted frame** — the on-the-wire/at-rest container: `magic(4)|version(1)|key_id(16)|nonce(12)|ciphertext|tag(16)` with AAD binding `file_id` + object kind ([06](06-cryptography.md)).
- **Key generation** — integer that bumps on FK rotation; clients must use the current generation.
- **Sync policy** — per-file server policy: `server-default` | `excluded`. (Offline pinning is the separate **client-local** `keepOnDevice` field, never sent to the server — [16 §16.2](16-offline-and-storage-policies.md).)
- **CRDT / Yrs / ydotnet** — the Yjs-family CRDT; `ydotnet` is the .NET binding the client uses to merge text documents and produce snapshots.
- **OS keystore** — the platform secret store: **DPAPI** (`ProtectedData`) on Windows, **Secret Service** (libsecret/D-Bus) on Linux ([07 §7.2](07-key-and-device-management.md)).

## 0.8 Phase map (Desktop)

| Phase | Desktop deliverable |
|-------|---------------------|
| 0 | App shell + navigation; account-scoped DI + per-account encrypted SQLite DB/keys/cache; Keycloak login + TOTP; device enrollment, identity keys in the OS keystore, recovery-key UX; structure browsing with encrypted names; local encrypted DB. |
| 1 | Markdown + plaintext editing on the encrypted CRDT (single user offline-first), blob sync, server sync policies (server-default/excluded) + client-local keep-on-device, on-demand download, view/edit modes, **full-corpus** on-device search. |
| 2 | Live relay collaboration (client-side merge), account + link sharing, guest mode, rotation-based revocation, version history with client diffs + restore; reference snapshotting. |
| 3 | Pen/stylus ink capture + the shared encrypted vector stroke format, LWW/version-vector ink sync (encrypted blobs). |
| 4 | Rich per-user/per-file config, settings, multi-account switcher UI, plaintext export / external-editor interop, audit surfacing where applicable, polish. |
| 5 | (Optional) office/source/image content types as encrypted blobs; chunked upload. |
| 6 | (Optional) key-transparency/safety-number verification UI. |

See [20-roadmap.md](20-roadmap.md) for detail and acceptance criteria.
