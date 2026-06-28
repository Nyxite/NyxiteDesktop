# 01 — Architecture

## 1.1 Shape: offline-first, layered, unidirectional

The app is **offline-first**: the local encrypted database is the source of truth the UI renders from; the network is a background reconciler. The user can read, edit, draw, organize, and search with no connectivity; changes queue and sync when the relay/REST is reachable.

It follows a **Clean-architecture layering** with **MVVM** (unidirectional state flow) in the presentation layer:

```
┌──────────────────────────────────────────────────────────────────┐
│ Presentation (Avalonia 11, Fluent theme, XAML)                    │
│   Views ── ViewModels (MVVM: observable state + commands) ── Nav  │
│   CommunityToolkit.Mvvm default · ReactiveUI in editor/collab     │
└───────────────▲───────────────────────────────────────┬──────────┘
                │ IObservable / INotifyPropertyChanged    │ commands
┌───────────────┴───────────────────────────────────────▼──────────┐
│ Domain (pure C#, no UI/IO deps) — from Nyxite.Domain + app use    │
│   Use cases (interactors) · Entities · Repository interfaces      │
└───────────────▲───────────────────────────────────────┬──────────┘
                │                                         │
┌───────────────┴───────────────────────────────────────▼──────────┐
│ Data                                                              │
│  Repositories (impl) ── orchestrate:                              │
│   • LocalStore (EF Core / SQLite + SQLCipher) ← source of truth   │
│   • ApiClient (HttpClient + Refit)            ← REST over ciphertext│
│   • RelayClient (SignalR .NET client)         ← encrypted CRDT relay│
│   • CryptoEngine (Nyxite.Crypto)              ← all encryption     │
│   • CrdtEngine (Nyxite.Crdt / ydotnet)        ← text merge + snapshot│
│   • KeyStoreVault (DPAPI / Secret Service)    ← key material       │
│   • BlobStore (filesystem)                    ← cached/kept blobs  │
└──────────────────────────────────────────────────────────────────┘
```

The **domain layer is platform-free** so it can be unit-tested on the runtime without Avalonia, and so it can reuse `Nyxite.Domain`/`Nyxite.Contracts` types directly. It mirrors the server's discipline of keeping `Domain`/`Contracts`/`Crypto` free of host concerns ([server 01](https://github.com/Nyxite/server)).

## 1.2 The cardinal rule: plaintext never crosses the network boundary

There is exactly one place where plaintext becomes ciphertext and vice-versa: the **CryptoEngine** (the shared `Nyxite.Crypto` library), invoked by repositories. Enforced by construction:

- The `ApiClient` and `RelayClient` types accept and return **opaque byte arrays / framed blobs only** — they have no method that takes a domain content object and no reference to `CryptoEngine`. They cannot serialize plaintext by accident.
- Repositories are the only components that hold both a `CryptoEngine` and a network client. Every outbound content payload passes through `CryptoEngine.Seal(...)`; every inbound one through `CryptoEngine.Open(...)`.
- The local DB is encrypted at rest (SQLCipher) and the blob store is either stored as ciphertext or, for decrypted/kept plaintext, inside an app-private directory gated by the OS-keystore-bound master key ([17](17-security.md)).

An **architecture test** (`NetArchTest`/`ArchUnitNET`, [18](18-build-ci-testing.md)) fails the build if a network project references the crypto or domain-content projects, or if a DTO carrying a plaintext field reaches a network client.

## 1.3 Presentation: MVVM

The presentation layer is MVVM with a single decision per screen on which toolkit backs the ViewModel ([README](README.md), [02 §2.2](02-tech-stack-and-libraries.md)):

- **CommunityToolkit.Mvvm (default)** for the majority of screens (browse, settings, history, sharing, auth, search results). State is exposed as `[ObservableProperty]` fields; user actions as `[RelayCommand]` (async, with generated `CanExecute`). Low ceremony, source-generated, easy to read.
- **ReactiveUI (selectively)** in the **editor and collaboration** screens, where state is genuinely stream-shaped: inbound CRDT updates, outbound awareness throttling, presence rosters, connection state, and debounced live search compose far more cleanly as `IObservable<T>` (`WhenAnyValue`, `ReactiveCommand`, `Throttle`, `ObserveOn`).

Common to both:

- Each ViewModel exposes an immutable-ish **observable state** that fully describes what to render (loading/empty/error/content, decrypted names/content already resolved) and **commands** as the only mutation entry points (`OpenFile`, `EditText`, `StrokeAdded`, `KeepOnDevice`, `CreateShareLink`).
- One-shot **effects** (navigation, dialogs, share-sheet, OS credential prompt) are surfaced via a mediator/`Interaction` rather than driven from the View.
- ViewModels depend on **use cases**, never on data sources directly. Decryption happens in the data layer; the UI receives already-plaintext state.
- All UI mutation marshals to the **Avalonia UI thread** (`Dispatcher.UIThread`); ViewModels never touch crypto/CRDT/IO on it ([§1.6](#16-threading--background-work)).

## 1.4 Domain: use cases & repository interfaces

Representative use cases (each a single-responsibility class with an `InvokeAsync` / `operator`-style `ExecuteAsync`):

- Structure: `ObserveProjectTree`, `CreateFolder`, `MoveFile`, `RenameFile`, `SetSyncPolicy`, `SoftDeleteFile`.
- Content: `OpenFileForRead`, `OpenFileForEdit`, `ApplyTextEdit`, `AppendInkStroke`, `SaveInkPage`, `ExportWorkingCopy`.
- Sync: `RunDeltaSync`, `PullManifest`, `DownloadBlob`, `UploadBlob`, `ReconcileFile`.
- Collaboration: `JoinDocument`, `SubmitUpdate`, `BroadcastAwareness`, `LeaveDocument`, `SnapshotDocument`.
- Keys/devices: `EnrollDevice`, `ApproveDevice`, `GenerateRecoveryKey`, `RecoverFromKey`, `RotateFileKey`, `PublishPublicKeys`.
- Sharing: `CreateAccountShare`, `CreateLinkShare`, `OpenShareLink`, `RevokeShare`, `ListShares`.
- History/search: `ListVersions`, `DiffVersions`, `RestoreVersion`, `Search`, `Reindex`.

Repository interfaces (`IFileRepository`, `IStructureRepository`, `IKeyRepository`, `IShareRepository`, `ISyncRepository`, `ICollabRepository`, `ISearchRepository`, `IVersionRepository`) live in the domain layer; implementations live in data.

## 1.5 Data layer components

| Component | Responsibility | Backed by |
|-----------|----------------|-----------|
| `LocalStore` | Single source of truth for structure, sync state, cached plaintext metadata, FTS index | EF Core over SQLite + SQLCipher ([04](04-local-data-model.md)) |
| `ApiClient` | Typed REST over `/api/v1`, ciphertext bodies, problem+json errors, idempotency | `HttpClient` + Refit + `System.Text.Json` ([05](05-api-client.md)) |
| `RelayClient` | SignalR `RelayHub` connection, join/submit/awareness, reconnect | `Microsoft.AspNetCore.SignalR.Client` ([09](09-realtime-collaboration.md)) |
| `CryptoEngine` | Seal/open framed objects, HPKE wrap/unwrap, sign/verify, content address, recovery KDF | **`Nyxite.Crypto`** (shared with server) ([06](06-cryptography.md)) |
| `CrdtEngine` | Apply/encode Yrs updates, state vectors, snapshots | **`Nyxite.Crdt`** / ydotnet ([09](09-realtime-collaboration.md)) |
| `KeyStoreVault` | Wrap/unwrap the DB master key & identity-key store under an OS-keystore secret | DPAPI / Secret Service ([07](07-key-and-device-management.md)) |
| `BlobStore` | Store/evict kept ciphertext and decrypted blobs (ink/binary) | App-private filesystem ([16](16-offline-and-storage-policies.md)) |
| `AuthManager` | Server tokens (access/refresh), refresh, share-token minting | Native auth (password + TOTP / passkey) by default; enterprise Keycloak OIDC via `IdentityModel.OidcClient` ([14](14-authentication.md)) |

## 1.6 Threading & background work

- **Async model**: `async`/`await` over the thread pool plus `System.Threading.Channels` for outbox/work queues and `IObservable`/Rx in the editor. A small injected `IDispatcherProvider` distinguishes the **UI thread** (`Dispatcher.UIThread`) from background work; **crypto, CRDT merge, diff, and BLAKE3 hashing run on the thread pool, never on the UI thread**.
- **Hosting**: the app is wired with the **.NET Generic Host** (`Microsoft.Extensions.Hosting`). Long-running background reconcilers are `BackgroundService`/`IHostedService` instances, started when an account session is active:
  - `DeltaSyncService` — periodic + on-foreground/connectivity-regained delta sync; drains the outbox.
  - `BlobPrefetchService` — downloads/decrypts keep-on-device and (for the full-corpus default) the rest of the corpus, subject to the storage/network budget ([16](16-offline-and-storage-policies.md)).
  - `SnapshotService` — compacts open Yrs docs into encrypted snapshots on trigger ([09 §9.6](09-realtime-collaboration.md), [12](12-version-history.md)).
  - `KeyRotationService` — background FK rotation after revocation ([07 §7.6](07-key-and-device-management.md)).
  - `ReindexService` — rebuilds/updates the FTS index ([11](11-search.md)).
- **Active editing/collaboration**: while a document is open, the `RelayClient`'s SignalR connection stays up for the document's room; the desktop runs continuously while the window (or tray) is alive, so there is no mobile-style foreground-service constraint. On the OS going to **sleep/resume**, the relay reconnects with backoff ([09 §9.9](09-realtime-collaboration.md)).
- **Single-instance**: the app is single-instance per user session (a named mutex / Unix socket lock); a second launch (e.g. from a `nyxite://` link) forwards its arguments to the running instance ([15 §15.1](15-ui-and-navigation.md)).

## 1.7 Error & connectivity model

- The UI always renders from `LocalStore`; network failures degrade gracefully to "offline / pending changes N".
- A per-file **sync state machine** — canonical set `Synced`, `PendingPush`, `PendingPull`, `Downloading`, `Uploading`, `Conflicted`, `Rotating`, `Excluded`, `Error(code)` (identical to [04 §4.4](04-local-data-model.md) and [08 §8.8](08-sync-engine.md)) — is persisted and surfaced as small badges ([08](08-sync-engine.md)).
- Server `problem+json` codes map to a typed `NyxiteApiError` hierarchy (e.g. `key_generation_stale` → trigger refetch+rotate, `share_revoked`/`link_expired` → mark dead, `excluded_content` → never retry upload, `429` → backoff with `Retry-After`). See [05 §5.4](05-api-client.md).

## 1.8 Dependency injection & account scoping

**`Microsoft.Extensions.DependencyInjection`** (via the Generic Host) wires the graph. Lifetimes: `Singleton` for stateless engines and process-wide infrastructure; transient/scoped ViewModels per screen; and an **account scope** for everything tenant-specific.

Because the app is **multi-account from v1.0.0** ([14 §14.7](14-authentication.md)), per-account state must not leak across accounts. An **account scope** (a child `IServiceScope` keyed by `accountId`) owns that account's **`UserSession`** (the unlocked identity-key handle, created after login + key-unlock and cleared on lock/logout/switch), its server tokens, its repositories, and its account-scoped data sources (the account's SQLCipher DB connection, `BlobStore` subtree, FTS index). The active account's scope is created on switch-in and disposed — **zeroizing in-memory key material** — on switch-out. No use case can touch decrypted key material before unlock, and no account can read another's data. See [04 §4.1](04-local-data-model.md), [07](07-key-and-device-management.md), and [17](17-security.md).
