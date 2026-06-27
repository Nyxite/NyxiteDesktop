# 02 — Tech Stack & Libraries

Concrete choices with rationale. Versions are **[P]** floors at time of writing; pin exact versions in `Directory.Packages.props` (Central Package Management, [18](18-build-ci-testing.md)) and keep them current. Prefer the smallest well-maintained library that meets the need; every dependency in a zero-knowledge client is attack surface.

The crypto and CRDT layers are **not** re-chosen here — they are the **shared `Nyxite.Crypto` and `Nyxite.Crdt` packages** the server publishes ([README](README.md), [06](06-cryptography.md), [09](09-realtime-collaboration.md)). This chapter pins the libraries those packages and the app depend on.

## 2.1 Language, runtime, build

| Concern | Choice | Notes |
|---------|--------|-------|
| Language | **C# 13** | Records, pattern matching, `Span<T>`/`Memory<T>` for crypto buffers. |
| Runtime | **.NET 9** | `System.Security.Cryptography` AES-GCM; first-class SignalR client; self-contained + single-file publish; NativeAOT considered for size ([18](18-build-ci-testing.md)). |
| Build | **MSBuild + `dotnet`**, **Central Package Management** (`Directory.Packages.props`), `Directory.Build.props` | One pinned version set across all projects. |
| Solution | Multi-project `.slnx`/`.sln` (see [03](03-project-structure.md)) | Layer enforcement + build speed. |
| Analyzers | Roslyn analyzers + `.editorconfig` + nullable reference types **enabled** | Consistency + null-safety across the solution. |

## 2.2 UI

| Concern | Choice | Rationale |
|---------|--------|-----------|
| UI toolkit | **Avalonia 11** | Single C# UI for Windows + Linux; supports custom-drawn surfaces required by the ink canvas. |
| Theme / design system | **Fluent theme** (`Avalonia.Themes.Fluent`) + Nyxite color resources | Modern, adaptive; dark theme first-class (fits the "Nyx" night branding, [15 §15.4](15-ui-and-navigation.md)). |
| MVVM (default) | **CommunityToolkit.Mvvm** | Source-generated `[ObservableProperty]`/`[RelayCommand]`; lowest ceremony for the majority of screens ([01 §1.3](01-architecture.md)). |
| MVVM (reactive screens) | **ReactiveUI** (`Avalonia.ReactiveUI`) | Stream-shaped state in the editor/collaboration/search surfaces (`WhenAnyValue`, `ReactiveCommand`, `Throttle`). |
| Markdown render | **Markdig** (→ Avalonia render) | CommonMark + extensions parser; render to Avalonia controls for view mode ([10 §10.2](10-editors.md)). No remote fetch from content (privacy). |
| Code/text editor control | **AvaloniaEdit** | Text/markdown edit surface, syntax highlighting hook for Phase-5 source-code types ([10 §10.3](10-editors.md)). |
| Icons/assets | From `Nyxite/icons/desktop` | App icon, tray icon, window icons ([03 §3.5](03-project-structure.md)). |

## 2.3 Ink / stylus

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Ink surface | **Custom Avalonia control** drawing via the **Skia** backend (`SkiaSharp`, Avalonia's renderer) | Low-latency stroke rendering; full control over the stroke model. |
| Pointer/pressure input | Avalonia `PointerPressed`/`PointerMoved` with **`PointerPointProperties.Pressure`**, `XTilt`/`YTilt`, `Twist` where the platform/driver exposes them (Windows Ink, Linux libinput, Wacom) | Captures pressure/tilt for natural ink ([10 §10.4](10-editors.md)). |
| Stroke smoothing | Catmull-Rom / the ink model's own smoothing | Natural curves from sampled points. |

The on-disk/on-wire **ink stroke format** is a Nyxite-defined, versioned, encryptable vector format **co-designed with and shared with Android** ([10 §10.5](10-editors.md), [Android 10 §10.5](https://github.com/Nyxite/android)); Skia/Avalonia is the capture/render engine, not the storage format.

## 2.4 Concurrency, hosting & DI

| Concern | Choice |
|---------|--------|
| Async | **`async`/`await`**, `System.Threading.Channels`, `IObservable`/Rx (editor) |
| Hosting | **.NET Generic Host** (`Microsoft.Extensions.Hosting`) |
| DI | **`Microsoft.Extensions.DependencyInjection`** with an account-scoped child scope ([01 §1.8](01-architecture.md)) |
| Background work | **`BackgroundService`/`IHostedService`** with timers/channels (no mobile WorkManager analog needed) |
| Logging | **`Microsoft.Extensions.Logging`** behind a scrubbing facade ([§2.9](#29-observability--quality)) |

## 2.5 Local persistence

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Relational store | **EF Core 9** (SQLite provider) | Matches the server's EF Core idiom; typed DbContext, LINQ queries, migrations ([04](04-local-data-model.md)). |
| At-rest encryption | **SQLCipher** via **`SQLitePCLRaw.bundle_e_sqlcipher`** (passphrase via `PRAGMA key`) | Whole-DB encryption; passphrase = OS-keystore-protected master key ([07](07-key-and-device-management.md), [17](17-security.md)). |
| Full-text search | **SQLite FTS5** (bundled with the SQLCipher build) | On-device **full-corpus** search index ([11](11-search.md)). |
| Settings | **`Microsoft.Extensions.Configuration`** + a small JSON settings file for non-secret prefs; secrets go to the OS keystore | Non-secret prefs vs. secrets split. |
| Blob/file store | App-private directory under the per-OS data path ([16 §16.1](16-offline-and-storage-policies.md)) | Kept ciphertext + decrypted ink/binary. |

> SQLCipher build note: confirm the `bundle_e_sqlcipher` package ships FTS5 and the target arch (x64/arm64, Windows + Linux); otherwise pin a custom SQLCipher native build. This is an early validation item ([19](19-open-questions.md)).

## 2.6 Networking

| Concern | Choice | Rationale |
|---------|--------|-----------|
| HTTP client | **`HttpClient`** (`SocketsHttpHandler`) via **`IHttpClientFactory`** | Pooling, handler pipeline (auth, retry, idempotency), TLS config/pinning. |
| REST binding | **Refit** | Typed `/api/v1` surface from interfaces ([05](05-api-client.md)); raw stream path for ciphertext. |
| JSON | **`System.Text.Json`** | Source-gen serializers; matches `Nyxite.Contracts` DTOs (metadata is JSON, content bodies are binary). |
| Resilience | **`Microsoft.Extensions.Http.Resilience`** / Polly | Backoff/retry honoring `Retry-After`, circuit-breaking ([05 §5.2](05-api-client.md)). |
| Realtime | **`Microsoft.Aspnetcore.SignalR.Client`** (the official .NET client) | The server's relay is SignalR `RelayHub` ([09](09-realtime-collaboration.md)). First-class — no RxJava bridge needed, unlike Android. |
| OIDC/auth | **`IdentityModel.OidcClient`** (Authorization Code + PKCE) with a loopback/`nyxite://` redirect | Keycloak login in the system browser ([14](14-authentication.md)). |

## 2.7 Cryptography (shared, not re-chosen)

All primitives are the server's canonical implementations, consumed via **`Nyxite.Crypto`** ([06](06-cryptography.md)). The desktop adds only OS-keystore integration. For completeness, the underlying primitive providers the shared package relies on:

| Purpose | Primitive | Provider (inside `Nyxite.Crypto`) |
|---------|-----------|-----------------------------------|
| AEAD (AES-256-GCM) | AES-256-GCM (96-bit nonce, 128-bit tag) | `System.Security.Cryptography.AesGcm` (BCL) |
| HPKE (X25519+HKDF-SHA256+AES-256-GCM) | RFC 9180 `KEM=0x0020`, `KDF=0x0001`, `AEAD=0x0002` | the shared lib's HPKE impl (BouncyCastle or a vetted RFC 9180 impl) — **same code as the server** |
| Signing | Ed25519 | shared lib (BouncyCastle/NSec) |
| Key agreement | X25519 | shared lib (BouncyCastle/NSec) |
| Content address | BLAKE3-256 | **`Blake3`** (official Rust binding) |
| Recovery KDF | Argon2id (m=64 MiB, t=3, p=1) | **`Konscious.Security.Cryptography.Argon2`** or libsodium |
| OS key wrap (desktop-only add) | DPAPI / Secret Service | `System.Security.Cryptography.ProtectedData` (Windows) / libsecret D-Bus (Linux) ([07 §7.2](07-key-and-device-management.md)) |

> Because the desktop and server share `Nyxite.Crypto`, the framing and HPKE suite are **identical by construction** — the cross-client risk that Android/web carry (re-deriving the suite IDs) does not exist here. The desktop still runs the shared conformance vectors in CI so a change to `Nyxite.Crypto` that breaks web/Android is caught ([18 §18.5](18-build-ci-testing.md)).

## 2.8 CRDT (shared)

| Concern | Choice | Risk |
|---------|--------|------|
| Text CRDT | **`Nyxite.Crdt`** over **ydotnet** (Yrs .NET binding) | ydotnet is a **community-maintained** binding now running the full client-side merge and snapshotting. The shared `Nyxite.Crdt` glue also backs the server's `Nyxite.CrdtConformanceTests` harness, so interop with Yjs (web) / ykt (Android) is validated by shared vectors. **Spike ydotnet maturity early** ([09 §9.10](09-realtime-collaboration.md), [19](19-open-questions.md)); fallback is a thin own binding over `yffi`. |

## 2.9 Observability & quality

| Concern | Choice |
|---------|--------|
| Logging | `Microsoft.Extensions.Logging` behind a thin `Logger` facade with **content/key scrubbing** — never log plaintext, keys, tokens, fragments, or full share URLs ([17](17-security.md)). |
| Crash/perf | Self-hosted/opt-in only; **no third-party content-touching SDKs**. Default off for privacy. **[P]** |
| Testing | xUnit, FluentAssertions, NSubstitute/Moq, Avalonia.Headless for UI, EF Core SQLite in-memory/file tests, **shared crypto + CRDT conformance vectors** ([18](18-build-ci-testing.md)). |
| Static analysis | Roslyn analyzers + `dotnet format` + an architecture test (`NetArchTest`) for layer/boundary rules ([01 §1.2](01-architecture.md)). |

## 2.10 Packaging & update

| OS | Format | Notes |
|----|--------|-------|
| Windows | **MSIX** installer (+ optional **winget** manifest) | Signed; self-contained .NET; per-user install ([18 §18.4](18-build-ci-testing.md)). |
| Linux | **AppImage** (portable) + **Flatpak** + **`.deb`** | AppImage for any distro; Flatpak (Flathub) and `.deb` for managed installs. |
| Update | In-app update check + signed artifacts | MSIX auto-update on Windows; Flatpak/AppImage updater on Linux ([18 §18.4](18-build-ci-testing.md)). |

## 2.11 Dependency policy

- Every dependency is reviewed for maintenance, license compatibility (PolyForm-noncommercial app, but dependencies must be redistributable), and whether it could exfiltrate content.
- No analytics/ad/attribution SDKs. No dependency that sends data off-device by default.
- Pin versions via Central Package Management; enable **NuGet package signature + lock-file (`packages.lock.json`) verification**; scan in CI ([18](18-build-ci-testing.md)).
