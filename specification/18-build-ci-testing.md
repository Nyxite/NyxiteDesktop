# 18 — Build, CI, Testing & Packaging

## 18.1 Build setup

- **.NET 9 SDK**, MSBuild, **Central Package Management** (`Directory.Packages.props`) + `Directory.Build.props` for shared settings (nullable enabled, analyzers, deterministic builds) ([02 §2.1](02-tech-stack-and-libraries.md), [03 §3.4](03-project-structure.md)).
- Build configs: `Debug` (test instance host default), `Release` (trimmed/optimized, signed). Target runtimes: `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64`.
- Publish modes: **self-contained, single-file** per RID; **NativeAOT** evaluated for startup/size (validate native interop — SQLCipher, ydotnet, BLAKE3, libsecret — under AOT; [19](19-open-questions.md)).

## 18.2 Configuration

- Instance host (API base, public-share base, plus an OIDC authority for enterprise-SSO instances) configurable per account via settings ([05](05-api-client.md), [14](14-authentication.md)); default `nyxite.app`. No secrets in the artifact; the only auth config that ships is, for enterprise Keycloak instances, the OIDC **public** client id (PKCE, no client secret on a desktop app) — native auth needs no embedded secret.

## 18.3 Module/build hygiene

- Architecture/layering enforced by a **`NetArchTest`** (or `ArchUnitNET`) test: feature projects must not reference data/network/crypto; network must not reference crypto/CRDT/content models ([01 §1.2](01-architecture.md), [03 §3.1](03-project-structure.md)).
- `dotnet format` + Roslyn analyzers gate the build; treat warnings as errors in CI.

## 18.4 Signing, packaging & release

| OS | Artifact | Signing |
|----|----------|---------|
| Windows | **MSIX** installer (per-user) + optional **winget** manifest | Authenticode / MSIX code-signing cert (CI secret) |
| Linux | **AppImage** + **Flatpak** (Flathub manifest) + **`.deb`** | detached signature / Flatpak GPG; checksums published |

- **Auto-update**: MSIX app-installer update feed on Windows; Flatpak repo updates and an AppImage update mechanism (`zsync`/in-app check) on Linux. The app checks for updates, verifies signatures, and prompts.
- Versioning: **SemVer**; pre-1.0 milestone tags track roadmap phases ([20](20-roadmap.md)). The API is consumed at `/api/v1`; the crypto frame and CRDT wire protocol are pinned by conformance vectors so primitives/keys can rotate without a format break ([06 §6.3](06-cryptography.md), [§18.6](#186-conformance-vector-sharing)).
- `packaging/` holds the MSIX manifest, AppImage recipe, Flatpak manifest, and `.deb` control files.

## 18.5 Test strategy (the critical surfaces first)

| Layer | Tools | Focus |
|-------|-------|-------|
| **Crypto conformance** | xUnit + shared KAT/cross-client vectors | **AES-GCM framing, HPKE wrap/unwrap, Ed25519, X25519, BLAKE3, Argon2id** must interop byte-for-byte with server/android/web. Since desktop *is* `Nyxite.Crypto`, these vectors are the canonical guard that the shared lib stays interoperable. **Highest priority.** ([06 §6.9](06-cryptography.md)) |
| **CRDT conformance** | xUnit + shared Yrs wire vectors | ydotnet (via `Nyxite.Crdt`) must produce identical merged state and encoded updates vs Yjs/android yrs/UniFFI — the same vectors the server `Nyxite.CrdtConformanceTests` uses. **Validate ydotnet before committing.** ([09 §9.10](09-realtime-collaboration.md)) |
| Domain | xUnit, FluentAssertions, NSubstitute | Use cases, policy logic, conflict/sync state machine — pure runtime. |
| Data | EF Core SQLite (file + in-memory), `WireMock.Net` | DbContext/migrations (asserted, against a SQLCipher-keyed connection), API mapping, error mapping, outbox/idempotency, delta/manifest reconcile. |
| ViewModels | xUnit + `Microsoft.Reactive.Testing` (Rx) | MVVM/Reactive state and commands. |
| UI | **Avalonia.Headless** + screenshot tests | Views, editor interactions, navigation, deep links. |
| Ink | Synthetic pointer/pressure streams | Stroke capture, serialization round-trip + stable BLAKE3 address, render; **round-trip with Android's ink format vectors**. |
| E2E | Integration against a test server (Testcontainers / staging) | Login → enroll → create → sync → collaborate → share → revoke/rotate → restore → **export round-trip**. |
| Packaging smoke | CI per RID | Each artifact launches, opens, signs in against the test host. |

## 18.6 Conformance vector sharing

- The crypto KATs, HPKE pairs, and CRDT update sequences are the **same shared vector files** co-owned with the server/android/web repos, checked into `tests/.../Resources`, updated in lockstep when the protocol changes. A protocol change that breaks a vector must break CI on **all** clients. Because desktop and server build from the same `Nyxite.Crypto`/`Nyxite.Crdt`, the desktop suite doubles as a server-side interop guard.

## 18.7 CI pipeline

- On PR: build all RIDs (or a representative subset), `dotnet format`/analyzers, unit tests, architecture test, **crypto + CRDT conformance**, EF Core migration tests, headless UI tests.
- On main/tags: full multi-RID build, packaging smoke tests, signed-artifact build, NuGet lock-file + signature verification, dependency vulnerability scan; publish artifacts.
- **Fail fast on any conformance regression** — interop is non-negotiable in an E2EE multi-client system.

## 18.8 Early validation spikes (do these first)

1. **ydotnet spike** — interop + performance with real collaboration; decide ydotnet vs. an own `yffi` binding ([09 §9.10](09-realtime-collaboration.md), [19](19-open-questions.md)).
2. **`Nyxite.Crypto` packaging spike** — confirm the shared crypto/CRDT/contract packages build and run cleanly in the desktop solution across all RIDs (and under NativeAOT if pursued).
3. **SQLCipher-on-desktop spike** — `bundle_e_sqlcipher` with FTS5 across win/linux × x64/arm64; EF Core migrations against a keyed connection.
4. **OS-keystore spike** — DPAPI vs Secret Service availability/behavior, headless-Linux fallback, app-lock factor integration ([07 §7.2](07-key-and-device-management.md)).
5. **Ink-input spike** — pen pressure/tilt across Windows Ink, Linux libinput, and Wacom on real hardware ([10 §10.4](10-editors.md)).
6. **SignalR relay spike** — the .NET client against the `RelayHub`: reconnect, share-token upgrade, sleep/resume.
