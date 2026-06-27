# 03 — Project Structure

## 3.1 Solution / project graph

Multi-project to enforce the layering in [01](01-architecture.md) at compile time (a UI project physically cannot reference a network client; a network project physically cannot reference crypto/domain-content types) and to keep builds fast. Shared server packages are consumed as **NuGet references**, not vendored.

```
NyxiteDesktop/
├── Directory.Packages.props            # central package versions
├── Directory.Build.props               # shared build settings, nullable, analyzers
├── NyxiteDesktop.slnx
│
├── src/
│   ├── Nyxite.Desktop.App/             # entry point: Avalonia App, Generic Host, DI graph, navigation host, tray, single-instance
│   │
│   ├── core/
│   │   ├── Nyxite.Desktop.Common/      # Result types, IDispatcherProvider, time, base64url, logging facade
│   │   ├── Nyxite.Desktop.Crypto/      # thin adapter over Nyxite.Crypto (CryptoEngine) + OS-keystore wrapping
│   │   ├── Nyxite.Desktop.Crdt/        # thin adapter over Nyxite.Crdt/ydotnet (CrdtEngine): snapshots, state vectors
│   │   ├── Nyxite.Desktop.KeyStore/    # KeyStoreVault: DPAPI (Windows) / Secret Service (Linux)
│   │   ├── Nyxite.Desktop.Database/    # EF Core DbContext, entities, FTS, migrations (SQLite + SQLCipher)
│   │   ├── Nyxite.Desktop.Network/     # Refit ApiClient, SignalR RelayClient, DTO adapters (ciphertext-only)
│   │   └── Nyxite.Desktop.Ui/          # Avalonia theme, design tokens, shared controls, the ink canvas control
│   │
│   ├── Nyxite.Desktop.Domain/          # use cases + repository interfaces (references Nyxite.Domain + Common)
│   │
│   ├── data/
│   │   ├── Nyxite.Desktop.Data.Structure/   # projects/folders/files CRUD + tree
│   │   ├── Nyxite.Desktop.Data.File/        # content read/write, blob store, addressing
│   │   ├── Nyxite.Desktop.Data.Sync/        # manifest/delta engine, hosted services, state machine
│   │   ├── Nyxite.Desktop.Data.Collab/      # relay session orchestration, awareness, snapshot triggers
│   │   ├── Nyxite.Desktop.Data.Keys/        # identity/device/recovery, key directory, rotation
│   │   ├── Nyxite.Desktop.Data.Share/       # account + link shares, revocation
│   │   ├── Nyxite.Desktop.Data.Search/      # full-corpus FTS indexer/queries
│   │   └── Nyxite.Desktop.Data.Version/     # version listing, diff, restore
│   │
│   └── feature/                        # Avalonia Views + ViewModels (reference Domain + Ui)
│       ├── Nyxite.Desktop.Feature.Auth/        # login, TOTP handoff, device enrollment, recovery
│       ├── Nyxite.Desktop.Feature.Browse/      # project/folder/file navigation
│       ├── Nyxite.Desktop.Feature.EditorText/  # markdown + plaintext editor (view/edit)
│       ├── Nyxite.Desktop.Feature.EditorInk/   # pen/stylus ink editor
│       ├── Nyxite.Desktop.Feature.Collab/      # presence/awareness overlays
│       ├── Nyxite.Desktop.Feature.Share/       # create/manage shares, open link, guest mode
│       ├── Nyxite.Desktop.Feature.History/     # version history + diff + restore
│       ├── Nyxite.Desktop.Feature.Search/      # search UI
│       └── Nyxite.Desktop.Feature.Settings/    # config, keep-on-device, storage, accounts, security, export
│
├── tests/                              # mirrors src/ (see 18)
└── packaging/                          # MSIX, AppImage, Flatpak manifest, .deb control (see 18)
```

### Consumed shared packages (NuGet, from the `server` repo)

- `Nyxite.Domain`, `Nyxite.Contracts` → referenced by `Domain` / `Network`.
- `Nyxite.Crypto` → referenced only by `Nyxite.Desktop.Crypto`.
- `Nyxite.Crdt` → referenced only by `Nyxite.Desktop.Crdt`.

### Dependency rules (enforced by analyzers + architecture test)

- `Feature.*` → `Domain`, `Ui`, `Common`. **Never** → `Data.*`, `Network`, `Database`, `Crypto`.
- `Data.*` → `Domain`, `core/*`. They are the only projects wiring crypto + network + db together.
- `Domain` → `Nyxite.Domain`, `Common` only. No Avalonia, no IO.
- `Network` must **not** reference `Crypto`, `Crdt`, or domain-content types (it speaks bytes/DTOs only).
- `App` → everything (composition root, Host builder, navigation host).

## 3.2 Namespaces

Root namespace: **`Nyxite.Desktop`** (e.g. `Nyxite.Desktop.Crypto`, `Nyxite.Desktop.Feature.EditorText`, `Nyxite.Desktop.Data.Sync`). Project file name == root namespace == assembly name, so the namespace map is unambiguous.

## 3.3 Naming conventions (code)

- Match the master `docs/NAMING.md`: product is **Nyxite**; primary domain `nyxite.app`.
- Types: `PascalCase`; the public API of each `core/*` engine is an interface (`ICryptoEngine`, `ICrdtEngine`, `IRelayClient`, `IKeyStoreVault`) with an impl named `*Engine`/`Ydotnet*`/`SignalR*`/`Dpapi*`/`SecretService*` to signal the backing tech.
- Use cases: verb-first (`OpenFileForEdit`), `Task<T> ExecuteAsync(...)`.
- DTOs (network): suffix `Dto` (or reuse `Nyxite.Contracts`); map to/from domain entities in `Data.*` adapters — DTOs never escape `Network`/`Data.*`.
- EF Core entities: suffix `Entity`; never exposed above the data layer.
- Avalonia: view `XView` (`.axaml` + code-behind), view model `XViewModel`, state types as records, commands as `IRelayCommand`/`ReactiveCommand`.

## 3.4 Build infrastructure

- **Central Package Management** (`Directory.Packages.props`) pins every version once; `Directory.Build.props` enables nullable, analyzers, deterministic builds, and the shared assembly metadata.
- A small set of shared MSBuild props per project *kind* (library / Avalonia / test) keeps each `.csproj` minimal.
- `dotnet format` + analyzer rules + the `NetArchTest` architecture test gate the build ([18 §18.3](18-build-ci-testing.md)).

## 3.5 Resource & asset strategy

- App/tray/window icons ship from the master [`Nyxite/icons/desktop`](https://github.com/Nyxite/Nyxite) set; wire them in `App` and the packaging manifests.
- Strings centralized (resx or a strings service) for future localization; no user content in resources.
- Conformance test vectors (CRDT wire, crypto KATs) live under `tests/.../Resources` and are the **same shared vector files** co-owned with server/web/android ([18 §18.6](18-build-ci-testing.md)).

## 3.6 Repository files

The `desktop` repo keeps, per the org convention ([master `docs/LICENSING-AND-REPOS.md`](https://github.com/Nyxite/Nyxite)): `README.md`, `FEATURES.md` (graduated from `Nyxite/features/desktop.md`), `LICENSE.md` (PolyForm Noncommercial 1.0.0 with filled copyright), this `specification/` folder, and the .NET solution.
