# 05 — REST API Client

Implements the server REST surface ([server 04](https://github.com/Nyxite/server)) under `Nyxite.Desktop.Network`. The client speaks **JSON for structure/metadata** (via `Nyxite.Contracts` DTOs) and **binary streams for ciphertext**; it never serializes plaintext content ([01 §1.2](01-architecture.md)).

## 5.1 Configuration

- **Base URL**: `https://{host}/api/v1` — configurable (the instance host; default `nyxite.app`). The API base, public-share base, and (for enterprise-SSO instances) the OIDC authority are all configurable at build/runtime and are **per-account** (multi-account, [14 §14.7](14-authentication.md)).
- **Auth**: `Authorization: Bearer <the server's access token>` on all `/api/v1/**` except public share endpoints. Guests use a short-lived **share token** ([14](14-authentication.md)). **Token lifetimes** (match server): access token ~5 min; guest share-session token 15 min (renewable); relay socket ticket single-use, 60 s.
- **IDs** UUIDv7; **timestamps** RFC 3339 UTC.
- **Pagination**: cursor-based — `?cursor=<opaque>&limit=<n>` (default `50`, max `200`); responses are `{ items, nextCursor }`. The cursor is an **opaque base64url** token, persisted as-is (`SyncCursorEntity.Cursor`) and **never parsed** by the client.
- **Idempotency**: `Idempotency-Key` on POST creates; keys are **(user, endpoint)-scoped for 24 h**. A replay with the same key returns the original response; a mismatched body with the same key returns `409 idempotency_conflict`.
- **TLS**: standard validation against the public chain (Cloudflare-fronted `nyxite.app`). Optional **certificate/public-key pinning** via a `SocketsHttpHandler` `ServerCertificateCustomValidationCallback` (pin set configurable, documented rotation, [17](17-security.md)). For admin/origin-direct hosts using a private/Origin CA, the OS trust store or a configured CA bundle applies — out of scope for normal users.

## 5.2 HttpClient / Refit setup

- A single named `HttpClient` per account scope (via `IHttpClientFactory`) with a **DelegatingHandler** pipeline, in order:
  1. **AuthHandler** — attach bearer/share token; on `401 token_expired` trigger a single refresh + retry ([14](14-authentication.md)).
  2. **IdempotencyHandler** — attach `Idempotency-Key` for create calls from the outbox.
  3. **ProblemJsonHandler** — parse `application/problem+json` into typed errors.
  4. **ResilienceHandler** (`Microsoft.Extensions.Http.Resilience`/Polly) — backoff on `429`/`5xx` honoring `Retry-After`.
  5. (debug) logging handler with **strict redaction**: ciphertext sizes and opaque IDs may be logged; **never plaintext, keys, tokens, or fragments** ([17](17-security.md)).
- **Refit** generates typed clients from the interfaces below, using a `System.Text.Json` content serializer for JSON and a raw `Stream`/`HttpContent` path for ciphertext streams.
- **OpenAPI** at `/openapi/v1.json` is the contract; generate or hand-write the typed API and verify it against the spec in CI ([18](18-build-ci-testing.md)). Where possible, the `Nyxite.Contracts` package already provides the DTOs, removing drift.

## 5.3 Endpoint coverage (Refit services)

Grouped to match the server resource map. Bodies marked *(ciphertext)* are opaque byte streams; *(json)* are metadata DTOs with `nameEnc`/`metadataEnc` carried as base64/byte fields.

### IStructureApi
- `GET/POST /projects`, `GET/PATCH/DELETE /projects/{id}` *(json, `nameEnc`)*
- `GET /projects/{id}/folders`
- `POST/GET/PATCH/DELETE /folders`, `/folders/{id}` *(json, `nameEnc`,`parentFolderId`)*
- `GET /folders/{id}/files`
- `POST/GET/PATCH/DELETE /files`, `/files/{id}` *(json: `nameEnc`,`contentType`,`syncPolicy`,`parentFolderId`,`metadataEnc`,`crdtDocId`)* — `POST` sets `contentType` (**immutable**) and the client-allocated `crdtDocId` (UUIDv7; required for text types, null otherwise); `PATCH` updates **only** `nameEnc`, `syncPolicy`, `parentFolderId` (move), `metadataEnc`. `syncPolicy` is the two-value enum `server-default`|`excluded` only.

### IFileContentApi
- `GET /files/{id}/blob` *(ciphertext stream; supports `If-None-Match: <contentHash>`)*
- `PUT /files/{id}/blob` *(ciphertext; ink/binary LWW; `If-Match: <parentSeq>`)*
- `GET /files/{id}/crdt/log?since={seq}` *(encrypted updates — REST fallback)*
- `POST /files/{id}/crdt/log` *(submit encrypted update(s) — REST fallback)*
- `GET /files/{id}/snapshot` *(latest encrypted snapshot pointer/stream)*

### IFileKeysApi
- `GET /files/{id}/keys` *(caller's wrapped FK(s))*
- `POST /files/{id}/keys` *(upload wrapped FK(s) for members)*
- `POST /files/{id}/keys/rotate` *(register new generation: re-wrapped keys + new head ciphertext ref)*
- `POST /files/keys:batch` *(subtree share fan-out: `{ grants: [ { fileId, keyId, memberId, wrappedKey } ] }`; idempotent, per-item partial success — see [13](13-sharing.md))*

### IKeysDevicesRecoveryApi
- `GET /keys/directory?userId=` | `?email=` *(public keys)*
- `PUT /keys` *(publish/rotate own public identity keys)*
- `GET/POST/DELETE /devices`, `/devices/{id}` — `POST /devices { label, pubkey }` → `{ deviceId, status: "pending", pairingCode, qrPayload }` (QR primary + 6–8 digit numeric code fallback, [07 §7.3](07-key-and-device-management.md))
- `POST /devices/{id}/approve { wrappedIdentityKey }` — approving device sends `wrappedIdentityKey = HPKE-seal(newDevicePubkey, identity bundle)`
- `GET /devices/me/enrollment` → `{ wrappedIdentityKey }` once approved — **single-use**; the server deletes the blob after this fetch
- `GET/PUT /recovery` *(server-opaque escrow blob + `kdf_params`)*

### IVersionsApi
- `GET /files/{id}/versions` *(paginated, newest first)*
- `GET /files/{id}/versions/{seq}` *(metadata)*
- `GET /files/{id}/versions/{seq}/blob` *(ciphertext)*
- `POST /files/{id}/restore` *(body `{ seq }` only; new head derived from version `seq`, [12 §12.4](12-version-history.md))*

### ISharesApi
- `GET /shares?targetType={file|folder|project}&targetId={id}`
- `POST /shares { targetType, targetId, kind, ... }` *(account: wrapped keys; link: `linkTokenHash` only)*
- `PATCH /shares/{id}` *(permission/expiry)*
- `DELETE /shares/{id}` *(revoke; client rotates key separately)*

### ISyncApi
- `POST /sync/changes` *(delta; cursor in/out)*
- `GET /sync/manifest?projectId=`

### IMeApi
- `GET /me`
- `GET/PUT /me/settings` *(settings body is **ciphertext**)*

### IPublicShareApi (no bearer)
- `GET /share/{token}` *(resolve link → guest bootstrap)*
- `GET /share/{token}/blob` *(ciphertext)*
- relay WS at `/share/{token}/ws` (handled by `RelayClient`, [09](09-realtime-collaboration.md))

> The client implements **no** `/search`, `/diff`, `/content` (decoded), or `/crdt/state` calls — none exist. Search and diff are local ([11](11-search.md), [12](12-version-history.md)).

## 5.4 Error model → typed errors

Parse RFC 9457 `problem+json` `code` into a `sealed`/abstract `NyxiteApiError` hierarchy and map to behavior:

| HTTP / code | Type | Client behavior |
|---|---|---|
| 400 `validation_failed`,`bad_sync_policy` | `Validation` | Surface; do not retry blindly. |
| 401 `unauthenticated`,`token_expired` | `Unauthenticated` | Refresh token once, else re-login ([14](14-authentication.md)). |
| 403 `acl_denied` | `Forbidden` | Surface "no access". |
| 403 `2fa_required` | `TwoFactorRequired` | Route to TOTP step. |
| 404 `not_found` | `NotFound` | Mark missing/removed. |
| 409 `conflict`,`version_conflict` | `Conflict` | LWW reconcile ([08 §8.5](08-sync-engine.md)). |
| 409 `excluded_content` | `ExcludedRejected` | Never retry upload of excluded files. |
| 409 `address_exists` | `AddressExists` | Treat as success (write-once idempotent). |
| 409 `idempotency_conflict` | `IdempotencyConflict` | Same `Idempotency-Key` reused with a different body; surface as a bug, do not retry. |
| 410 `share_revoked`,`link_expired` | `ShareDead` | Mark share/link dead ([13](13-sharing.md)). |
| 412 `key_generation_stale` | `KeyGenerationStale` | Refetch keys, re-encrypt with current gen, retry ([07](07-key-and-device-management.md)). |
| 413 `payload_too_large` | `TooLarge` | Chunk (Phase 5) or reject. |
| 429 `rate_limited` | `RateLimited(retryAfter)` | Backoff per `Retry-After`. |
| 5xx `internal`,`unavailable` | `ServerError` | Backoff + retry from outbox. |

## 5.5 Reliability: outbox + idempotency

- All mutating operations go through the **outbox** (`PendingOpEntity`, [04 §4.2](04-local-data-model.md)). Each create carries a stable client-generated `Idempotency-Key` (24h, (user,endpoint)-scoped server-side); safe to retry after crashes/network loss — a replay returns the original response, while a mismatched body under the same key is rejected `409 idempotency_conflict`.
- **Content addressing** makes blob/snapshot writes idempotent: re-`PUT` of the same address returns `address_exists` → treated as success.
- Conditional GET (`If-None-Match: contentHash`) avoids re-downloading unchanged ciphertext.
- The `DeltaSyncService` drains the outbox with exponential backoff + jitter, honoring `Retry-After` ([08](08-sync-engine.md)).

## 5.6 What the client must enforce locally (server can't)

Because the server validates only structure/ACL/write-once and **cannot read plaintext**:

- The client computes the **BLAKE3 content address honestly** and verifies downloaded ciphertext decrypts and matches the claimed address before trusting it ([06 §6.6](06-cryptography.md)).
- The client must not rely on the server to validate CRDT update contents — malformed updates simply fail to apply locally ([09](09-realtime-collaboration.md)).
- Size limits (100 MB inline) are checked client-side before upload to avoid `413`; chunked upload for larger is a Phase-5 item ([16 §16.5](16-offline-and-storage-policies.md)).
