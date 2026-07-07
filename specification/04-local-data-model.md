# 04 — Local Data Model

The local database is the **offline-first source of truth** for the UI ([01 §1.1](01-architecture.md)). It mirrors the server's structure graph, holds decrypted-on-device names/metadata (because the DB itself is encrypted at rest), tracks per-file sync state, caches CRDT/version pointers, and backs the **full-corpus** full-text index ([11](11-search.md)).

## 4.1 Storage engine & at-rest encryption

- **EF Core (SQLite provider)** over **SQLCipher** (`SQLitePCLRaw.bundle_e_sqlcipher`, keyed with `PRAGMA key`). The app is **multi-account from v1.0.0** ([14 §14.7](14-authentication.md)), so there is **one SQLCipher database per account** (`nyxite-{accountId}.db`), each with its **own 256-bit DB master key** generated on first sign-in of that account and stored **wrapped by the OS keystore** (DPAPI on Windows / Secret Service on Linux, [07](07-key-and-device-management.md), [17](17-security.md)), unwrapped only after app unlock. Account databases are never cross-queried; each account scope opens only its own DbContext ([01 §1.8](01-architecture.md)).
- A tiny separate **account registry** is a **minimal standalone SQLCipher DB** (encrypted, itself OS-keystore-wrapped, consistent with the per-account DBs) so the app can show the account switcher and pick a DB to open *before* any account DB is unlocked. Minimal schema, one row per known account: `accountId` (PK), `host` (instance base host), `displayName`, `lastUsedAt`, `dbFileName` (the `nyxite-{accountId}.db` file to open). It holds no content, names, or keys.
- Because the whole DB is encrypted at rest, decrypted **names**, small **metadata**, and the **FTS index** are stored as plaintext *inside* the DB. They are still protected by SQLCipher + OS keystore + (optional) app-lock gate. Identity **private keys** are **not** kept in the DB — they live in the dedicated key store ([07](07-key-and-device-management.md)).
- Large content blobs (ink/binary ciphertext, decrypted ink for kept files, cached snapshots) live on the filesystem in `BlobStore`, not in the DB ([16](16-offline-and-storage-policies.md)).

## 4.2 Entities (EF Core) — mirroring the server domain

IDs are server-issued **UUIDv7** values (stored as `TEXT`/`BLOB`); created-on-device rows generate a UUIDv7 client-side and reconcile. All `*Enc`/name fields are decrypted on read and stored decrypted in this (encrypted) DB; the **plaintext never leaves the device**. Domain enums come from `Nyxite.Domain` where shared.

### ProjectEntity
| Column | Type | Notes |
|--------|------|-------|
| `Id` | TEXT PK | UUIDv7 |
| `OwnerId` | TEXT | |
| `Name` | TEXT | decrypted project name |
| `MetadataJson` | TEXT? | decrypted metadata |
| `KeepOnDeviceDefault` | TEXT? | per-device keep-on-device default (`keep`/`dontKeep`/null=account default); cascades to descendants ([16 §16.2](16-offline-and-storage-policies.md)) |
| `CreatedAt`,`UpdatedAt` | INTEGER | epoch millis (or `DateTimeOffset` ticks) |
| `DeletedAt` | INTEGER? | soft delete |

### FolderEntity
`Id` PK, `ProjectId`, `ParentFolderId?` (null = project root), `Name`, `MetadataJson?`, `KeepOnDeviceDefault` (per-device `keep`/`dontKeep`/null=inherit), timestamps, `DeletedAt?`. Index on `ProjectId`, `ParentFolderId`. Acyclic parent chain enforced client-side. `KeepOnDeviceDefault` cascades to contained folders/files and seeds new descendants ([16 §16.2](16-offline-and-storage-policies.md)).

### FileEntity
| Column | Type | Notes |
|--------|------|-------|
| `Id` | TEXT PK | UUIDv7 |
| `ProjectId`,`FolderId?`,`OwnerId` | TEXT | folder null = project root |
| `Name` | TEXT | decrypted file name |
| `ContentType` | TEXT | enum: `markdown`,`plaintext`,`ink`,`sourcecode`,`office`,`image`,`binary` ([08](08-sync-engine.md)) — immutable |
| `SyncPolicy` | TEXT | server-side policy enum: `server-default`,`excluded` only ([08 §8.2](08-sync-engine.md)). The server never sees `KeepOnDevice`. |
| `KeepOnDevice` | TEXT? | **client-local, per-device** offline-pinning setting: `keep`/`dontKeep`/null=`inherit` from folder/project ([16 §16.2](16-offline-and-storage-policies.md)); never sent to the server. On desktop the **account default is effectively "keep" (full-corpus)** unless the user opts a subtree out ([16 §16.2](16-offline-and-storage-policies.md)). |
| `CurrentVersionSeq` | INTEGER? | head pointer |
| `CrdtDocId` | TEXT? | **client-allocated UUIDv7 at file creation** (required for text types, null otherwise); sent in the create-file request |
| `KeyGeneration` | INTEGER | monotonic FK rotation counter (1:1 with the current `KeyId`); drives `412 key_generation_stale` handling. The `KeyId` (uuid) is the stable frame/crdt/version reference for a specific file-key; `KeyGeneration` is the rotation generation. |
| `MetadataJson` | TEXT? | decrypted per-file metadata. For ink/binary it carries the per-file LWW **version-vector**: a `{ deviceId -> counter }` map (see [08 §8.5](08-sync-engine.md)), incremented on each local committed ink edit and compared component-wise to classify equal/ancestor/concurrent. |
| `CreatedAt`,`UpdatedAt`,`DeletedAt?` | INTEGER | |
| `ContentHash` | BLOB? | BLAKE3 of head plaintext (cache) |
| `SyncState` | TEXT | local state machine ([§4.4](#44-sync-state)) |
| `CacheState` | TEXT | `none`,`metadataOnly`,`ciphertextCached`,`plaintextCached` |
| `LastOpenedAt` | INTEGER? | for convenience-cache eviction of not-kept files ([16 §16.3](16-offline-and-storage-policies.md)) |

Indexes: `ProjectId`, `FolderId`, `OwnerId`, `SyncState`, `KeepOnDevice`, `(KeepOnDevice, LastOpenedAt)`. Effective keep-on-device is resolved by walking `File.KeepOnDevice` → nearest ancestor `KeepOnDeviceDefault` → account default (**keep**, on desktop).

### FileKeyEntity (locally unwrapped/wrapped cache)
`Id` (TEXT PK, **UUIDv7 surrogate key** — mirrors the server C1 fix), `FileId`, `KeyId`, `Generation`, `MemberId?`, `ShareId?`, `WrappedKey` BLOB (as fetched from server, for re-upload/rotation), and a transient unwrapped handle that is **never persisted in plaintext** — the unwrapped FK is held only in memory / re-derived on demand via the identity key.

- **Invariant (CHECK)**: exactly one of `MemberId` / `ShareId` is non-null (a wrapped key targets either an account member or a link share, never both).
- **Indices**: unique `(FileId, KeyId, MemberId)` filtered to `MemberId IS NOT NULL`, and unique `(FileId, KeyId, ShareId)` filtered to `ShareId IS NOT NULL`.

### FileVersionEntity
`Id` PK, `FileId`, `Seq`, `ContentHash` BLOB, `BlobRef` TEXT, `SizeCipher`, `KeyId`, `AuthorId?`, `CreatedAt`. Unique `(FileId, Seq)`. Index `(FileId, Seq DESC)`.

### CrdtUpdateEntity (local log mirror / outbox)
`Id` autogen, `CrdtDocId`, `Seq?` (null until server-assigned for outgoing), `UpdateEnc` BLOB, `KeyId`, `AuthorId?`, `CreatedAt`, `Direction` (`inbound`/`outbound`), `Acked` BOOL. Backs offline catch-up and the submit outbox ([09](09-realtime-collaboration.md)).

### ShareEntity
`Id` PK, target (`FileId?`/`FolderId?`/`ProjectId?`), `Kind` (`user_grant`/`link`), `GranteeId?`, `LinkTokenHash?` BLOB, `Permission` (`read`/`write`), `CreatedBy`, `CreatedAt`, `ExpiresAt?`, `RevokedAt?`. The **link token + fragment key are never stored** server-side; locally the app may keep a created link (token + fragment) only if the user chose to, in the OS keystore / encrypted prefs, clearly marked.

### DeviceEntity
`Id` PK, `Label`, `Pubkey` BLOB, `EnrolledAt`, `RevokedAt?`, `IsThisDevice` BOOL.

### DirectoryKeyEntity (cached public keys of other users)
`UserId` (or `Email`), `KeyId`, `HpkePubkey` BLOB (hybrid X25519 + ML-KEM-768), `SignPubkey` BLOB (hybrid Ed25519 + ML-DSA-65), `Generation`, `FetchedAt`. Used to wrap shares; refresh-on-use ([13](13-sharing.md)). Suite pinned by `alg_id` ([06 §6.2](06-cryptography.md)).

### Outbox / pending operations
`PendingOpEntity`: `Id`, `Kind` (`structure`,`blobUpload`,`crdtSubmit`,`delete`,`keyRotate`,`shareCreate`,`shareRevoke`), `TargetId`, `PayloadRef`, `IdempotencyKey`, `Attempts`, `NextAttemptAt`, `LastError?`. Drives reliable, retryable, idempotent sync ([05 §5.5](05-api-client.md), [08](08-sync-engine.md)).

### SyncCursorEntity
`Scope` (e.g. `project:{id}` or `global`) PK, `Cursor` TEXT, `UpdatedAt`. Stores the opaque delta-sync cursor ([08 §8.3](08-sync-engine.md)).

### AccountEntity / SessionEntity
The owning account of *this* database: `UserId`, `SubjectId` (the server's user subject; sourced from native auth by default, or from the enterprise Keycloak IdP), `DisplayName`, `Email`, `Role`, `InstanceHost`, token-expiry hints (tokens themselves are in the OS keystore, [14](14-authentication.md)). Because each account has its own DB, this is effectively a single row identifying the tenant; the cross-account list lives in the separate account registry ([§4.1](#41-storage-engine--at-rest-encryption)).

## 4.3 Full-text search (FTS5)

A contentless/external-content **FTS5** virtual table `FileContentFts(fileId UNINDEXED, title, body)` indexes decrypted titles and text content of every file present locally ([11](11-search.md)). On desktop the default is the **full corpus**, so the index aims to cover **everything** the account can read, not a subset. Maintained incrementally by `Data.Search` as content is decrypted/updated; never uploaded. Ink files index their recognized/typed text only if/when handwriting recognition exists (not v1.0.0) — otherwise only their title.

## 4.4 Sync state

`FileEntity.SyncState` is a persisted enum driving badges and the engine ([08](08-sync-engine.md)):

`Synced` · `PendingPush` · `PendingPull` · `Downloading` · `Uploading` · `Conflicted` (LWW loser retained) · `Rotating` (key rotation in progress) · `Excluded` · `Error(code)`.

## 4.5 Migrations & schema versioning

- EF Core migrations are **versioned**; checked in under `Nyxite.Desktop.Database/Migrations` and asserted by tests ([18](18-build-ci-testing.md)). Migrations must run against the **SQLCipher-keyed** connection.
- Forward-only migrations, matching the server's forward-only discipline.
- The DB stores a `SchemaMeta` row with app version + crypto-frame version expectations so a client can detect and refuse to misread newer encrypted frames ([06 §6.3](06-cryptography.md)).

## 4.6 What is **not** stored

- No server-side search index (none exists).
- No plaintext outside the encrypted DB / OS-keystore-gated blob store (the **only** exception is a user-initiated plaintext export, which is an explicit, clearly-warned action, [16 §16.7](16-offline-and-storage-policies.md)).
- No identity private key or recovery key in the DB (they live in the key store, [07](07-key-and-device-management.md)).
- No bearer/refresh tokens in the DB (OS keystore, [14](14-authentication.md)).
