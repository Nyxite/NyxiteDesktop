# 07 — Key & Device Management

Covers the identity keypair, device enrollment, the recovery key, FK rotation, and how all of this is protected on-device by the **OS keystore** (Windows DPAPI / Linux Secret Service). This is the Phase-0 foundation; nothing later retrofits it.

## 7.1 Key hierarchy (client view)

```
OS keystore secret (DPAPI on Windows / Secret Service on Linux)
   └─ wraps → DB master key (SQLCipher) and the Identity Key Store
                 └─ holds → Identity private keys: X25519 (HPKE) + Ed25519 (sign)
                              └─ unwraps → File Keys (FK, per file)
Recovery key (user-held phrase) ──Argon2id──► wrapping key ──► escrow of identity private key (server-opaque)
Device key (per device) ──► used to approve/enroll new devices
```

The OS-keystore secret is the on-device root of trust. **Caveat vs. mobile**: desktop platforms generally lack a hardware-isolated, attestable keystore equivalent to Android StrongBox / iOS Secure Enclave. DPAPI keys are bound to the **user account** (and machine) and unsealed transparently once the OS user is logged in; Secret Service unlocks with the login keyring. This is honestly weaker than hardware-backed mobile storage and is documented as such ([17 §17.2](17-security.md), [19 §19.2](19-open-questions.md)); the **app lock** ([§7.2](#72-on-device-key-storage), [17 §17.3](17-security.md)) is what gates use of the unwrapped keys at rest.

## 7.2 On-device key storage (`Nyxite.Desktop.KeyStore`)

The `IKeyStoreVault` abstraction has two implementations behind a common interface:

- **Windows — DPAPI** (`System.Security.Cryptography.ProtectedData`, `DataProtectionScope.CurrentUser`): wrap the DB master key and the Identity Key Store blob with the current-user data-protection key. Optionally bind an additional **app-lock passphrase / Windows Hello** factor so the wrap is not unsealed by mere OS login ([§7.2](#72-on-device-key-storage) app-lock, [17 §17.3](17-security.md)).
- **Linux — Secret Service** (libsecret over D-Bus; the GNOME Keyring / KWallet backend): store the wrapping secret in the login keyring collection. Where no Secret Service provider is present (headless/minimal desktops), fall back to a **passphrase-derived** wrap (Argon2id over a user app-lock passphrase) so keys are never written unprotected — surfaced clearly to the user ([19 §19.2](19-open-questions.md)).

The OS-keystore secret wraps:
- the **DB master key** (SQLCipher `PRAGMA key`),
- the **Identity Key Store** blob (the X25519+Ed25519 private keys, themselves serialized and AES-GCM-encrypted).

The unlocked identity private keys live only in the in-memory `UserSession` ([01 §1.8](01-architecture.md)); on lock/logout/account-switch the session is disposed and buffers zeroized (`CryptographicOperations.ZeroMemory`).

**App lock**: an optional (recommended-on) lock gates unwrapping the DB master key and identity key on launch and after an idle timeout — biometric/credential via **Windows Hello** where available, or an app passphrase (Argon2id-stretched) cross-platform. Default idle timeout **10 minutes**, configurable (1–60 min, or every-launch), consistent with the mobile lock model ([17 §17.3](17-security.md)).

## 7.3 Device enrollment

### First device / first sign-in
1. User authenticates (native password + TOTP or passkey by default; enterprise Keycloak SSO optional) → the server's access token ([14](14-authentication.md)).
2. App checks the key directory (`GET /keys/directory?userId=me`): if the user has **no identity keypair yet** (brand-new account), generate X25519 + Ed25519, store privates in the OS-keystore-wrapped identity store, and **publish publics** via `PUT /keys`.
3. Generate a **device keypair**; register the device via `POST /devices { label, pubkey }` → `{ deviceId, status: "pending", pairingCode, qrPayload }`.
4. Generate the **recovery key** and escrow ([§7.4](#74-recovery-key)). Force the user through the recovery-key flow before completing setup.

### Additional device (user already has an identity keypair elsewhere)
This device has the account token but **not** the identity private key. Two paths:

- **(A) Device-to-device approval.** New device registers (`POST /devices { label, pubkey }` → `status: "pending"` + `pairingCode`/`qrPayload`) and displays a **QR code** (primary) and a 6–8 digit numeric code (fallback). An already-enrolled device approves via `POST /devices/{id}/approve { wrappedIdentityKey }`, where `wrappedIdentityKey = HPKE-seal(newDevicePubkey, identity bundle)`; the server relays the opaque blob. The new device fetches it once via `GET /devices/me/enrollment` → `{ wrappedIdentityKey }` (**single-use**; the server deletes the blob after the fetch) and unwraps with its device private key. Interactive confirmation required on the enrolled device. On desktop the QR may be **scanned by another device** or, between two of the user's desktops, the numeric code typed across.
- **(B) Recovery-key unwrap.** User enters the recovery phrase; the app fetches the escrow blob (`GET /recovery` + `kdf_params`), derives the wrapping key (Argon2id), unwraps the identity private key, then stores it OS-keystore-wrapped and proceeds. This is the path when no other device is available.

Until enrolled with the identity private key, the device can authenticate and see structure (encrypted) but **cannot decrypt content**; the UI shows a clear "enroll this device" state.

## 7.4 Recovery key

- Generated as a **BIP39 24-word phrase** (256-bit, checksummed, vetted word list) **shown once**. The app must make the user record it (copy, write down, confirm by re-entry) and warn unmistakably: **losing all devices and this phrase = permanent, unrecoverable data loss** (no server/admin escrow exists).
- The app derives a wrapping key via **Argon2id** (m = 64 MiB, t = 3, p = 1, tunable — [06 §6.8](06-cryptography.md)) and **seals the identity bundle (X25519 priv ‖ Ed25519 priv) with AES-256-GCM** under that derived key (DECISION: AES-GCM, **not** HPKE — symmetric key, [06 §6.4](06-cryptography.md)). The recovery-blob shape is `{ version, kdf:{alg:"argon2id", m, t, p, salt(16)}, nonce(12), ciphertext, tag(16) }` with **AAD = `userId ‖ version`**. It uploads the **opaque** escrow + non-secret `kdf_params` via `PUT /recovery`. The server stores it but cannot open it.
- Re-issuing a recovery key (rotation) re-wraps the escrow under a new phrase and re-uploads; the old phrase is invalidated client-side by overwrite.
- The recovery key is **never** stored on the device in plaintext; if the user opts to keep a copy, it is their responsibility outside the app.

## 7.5 File-key lifecycle

- **Create**: generate FK (CSPRNG), encrypt initial content, wrap FK to the owner's public key, `POST /files` with `OwnerWrappedKey` + initial ciphertext ([05](05-api-client.md)).
- **Open**: `GET /files/{id}/keys` → the caller's `wrapped_key`; HPKE-unwrap with the identity private key → `FileKeyHandle`; cache the wrapped form locally, hold the unwrapped handle in memory only ([04 §4.2](04-local-data-model.md)).
- **Generation check**: every content frame carries `key_id` (FK + generation). If the server returns `412 key_generation_stale`, refetch keys and re-encrypt with the current generation ([05 §5.4](05-api-client.md)).

## 7.6 Key rotation & revocation (client-driven forward secrecy)

When a share is revoked ([13](13-sharing.md)), a remaining member's client must rotate the FK so removed members cannot read **future** content:

1. Generate a new FK (new `keyId`, `generation + 1`).
2. Re-encrypt the current head (and produce a fresh snapshot) under the new FK.
3. Re-wrap the new FK to every **remaining** member's public key.
4. `POST /files/{id}/keys/rotate { newKeyId, generation, wrappedKeys[], newHeadRef }`; the server bumps `key_generation`. If another member's rotation already won, the server returns **`409`** — abandon this attempt and refetch the new generation.
5. Peers see the `keyrotate` change in delta sync ([08](08-sync-engine.md)) and refetch their wrapped key.

The server simultaneously drops the removed member's ACL grant (instant cutoff) even before rotation completes. Already-downloaded ciphertext cannot be un-seen (inherent). Rotation runs in the background via `KeyRotationService`; the file shows `Rotating` state ([04 §4.4](04-local-data-model.md)). Because the desktop is the **reference snapshotter**, it is frequently the member that drives rotation.

## 7.7 Device loss / compromise

- A user can **revoke a device** (`DELETE /devices/{id}`) from another device; the revoked device's enrollment is invalidated server-side.
- If a device may be **compromised**, rotate the **identity keypair** (`PUT /keys` new generation) and re-wrap FKs in the background. This is heavier (touches every file the user holds) and is a Phase-4 security action with clear UX; the full-corpus desktop is well-placed to perform the bulk re-wrap.

## 7.8 UX requirements

- A **Security/Keys** settings area shows: this device, other enrolled devices (with revoke), recovery-key status (set / re-issue), identity-key generation, and the **app-lock** configuration.
- Enrollment and recovery flows are first-class screens in `Feature.Auth` ([15](15-ui-and-navigation.md)), with explicit, non-dismissable warnings about the no-escrow recovery model.
- Approving a new device requires an explicit confirmation gesture and shows the new device's label + fingerprint.

## 7.9 Ratified decisions & remaining validation

Decisions (mirroring the master tracker and [19 §19.2](19-open-questions.md)): **OS keystore for the wrapping secret** (DPAPI / Secret Service), with a passphrase-derived fallback where no Secret Service exists; **app-lock with a 10-minute idle window** (configurable 1–60 min, or every-launch); **Argon2id m = 64 MiB, t = 3, p = 1**; **BIP39 24-word** recovery phrase with scrubbed re-entry; **device-to-device approval via QR (primary) + numeric code (fallback)** showing the new device's label + fingerprint.

Remaining validation (spike, [19 §19.2](19-open-questions.md)): Secret Service availability across target Linux desktops and the headless fallback UX; DPAPI behavior across Windows roaming-profile setups; whether Windows Hello / `libfido2` can strengthen the at-rest wrap beyond plain OS-login unseal.

## 7.10 Group keys (enterprise/family sharing)

Group sharing ([13 §13.7](13-sharing.md), [groups.md](https://github.com/Nyxite/Nyxite)) adds a **group keypair** whose private half is wrapped once per member — a third managed key alongside identity keys and FKs. The crypto is [06 §6.10](06-cryptography.md); this section covers the client key-management lifecycle (build steps **P4.4-DSK-1/2**). Group keys depend on **key transparency (Phase 4.3)** and reuse the Phase-2.3 rotation machinery ([§7.6](#76-key-rotation--revocation-client-driven-forward-secrecy)) at the group-key level.

### Group key generation
- The creator generates a group **X25519 + Ed25519** keypair (`GenerateGroupKeyPair`, [06 §6.10](06-cryptography.md)), **publishes the public half** to the directory (`POST /groups { group_pubkey, ed25519_pubkey, scope_kind, max_members? }`), and holds the private half only in-session, OS-keystore-wrapped like the identity store ([§7.2](#72-on-device-key-storage)). Keys are **scoped per project/time-period** (`scope_kind`); a file wraps to the group key **of its scope**.

### Member enrollment (transparency-verified)
Adding a member is **O(1)** — one grant blob, no file touched:
1. The enrolling client (which already holds the group key) fetches the newcomer's identity public key from the directory and **verifies it against the key-transparency log (Phase 4.3)** — an inclusion proof, stronger than the TLS + Ed25519-self-signature model used for one-off account shares ([13 §13.6](13-sharing.md)). Because one substituted key would expose the whole group's corpus (not a single file), **enrollment refuses to wrap to an unverified key**.
2. On success, `WrapGroupKeyToMember` HPKE-seals the group private key under the verified public key ([06 §6.10](06-cryptography.md)) and writes **one** append-only **group-key grant** via `POST /groups/{id}/members` — `group_id | scope_id | member_id | generation | alg_id | hpke_ct`. No file DEK is re-wrapped; the grant instantly opens every file the group can read.
3. Enrollment is **server-gated** by the group-size limit (membership-row count only) and audited; an over-limit add is rejected server-side ([13 §13.7](13-sharing.md)).

### Grant handling (read path)
- To open a group-readable file, `GET /groups/{id}/keys` → the caller's grant; `UnwrapGroupKey` with the identity private key → `GroupKeyHandle`; then `UnwrapDekViaGroup` on the file's DEK-to-group wrap → `FileKeyHandle` ([06 §6.10](06-cryptography.md)). The wrapped grant is cached locally (`DirectoryKeyEntity`-style); the unwrapped `GroupKeyHandle` lives only in the `UserSession` and is zeroized on lock/logout ([§7.2](#72-on-device-key-storage)).
- Grants are **append-only** and generation-tagged, giving an auditable "who could read what when" history; the client always unwraps with the **current** generation for its scope.

### `GroupKeyRotationService` (removal / forward secrecy)
Removal is two-layer, mirroring [§7.6](#76-key-rotation--revocation-client-driven-forward-secrecy) but at the group-key level and **scoped**:
1. **Soft** — `DELETE /groups/{id}/members/{uid}` deletes the departing member's grant (instant ACL cutoff). One delete.
2. **Rotate (forward secrecy)** — a remaining member's `GroupKeyRotationService` generates a **new group key for the affected scope**, re-wraps it to each **remaining** member (`generation + 1`), and commits `POST /groups/{id}/keys/rotate { scopeId, generation, wrappedGroupKeys[] }`. New files use the new generation; the removed member reads nothing new.
3. **Full (seal history)** — additionally **re-seals the affected scope's DEKs** under the new group key (re-wraps the small DEK-to-group blobs only, never file ciphertext).
- **Concurrency**, identical to `KeyRotationService` ([§7.6](#76-key-rotation--revocation-client-driven-forward-secrecy)): a concurrent rotation loser gets **`409`** (abandon, refetch the new generation); an in-flight **old-key** wrap after commit gets **`412 key_generation_stale`**, then re-seals under the current key. Only the **affected scope** is touched — other scopes are untouched, bounding the manager-revocation blast radius.
- The service runs in the background; the group/scope shows a `Rotating` state. As the full-corpus reference snapshotter, the desktop is frequently the member that drives group rotation. **Honest caveat (unchanged):** nothing retracts what a keyholder already decrypted; rotation only guarantees they read nothing new ([13 §13.7](13-sharing.md)).

### Recovery restores group access (for free)
- Because each group private key is wrapped **under the member's identity public key**, recovering the identity key (recovery phrase, [§7.4](#74-recovery-key)) automatically restores group access — the group grants simply unwrap under the recovered personal key on the fresh device, **no special path**.
- A member who loses **all devices and the recovery phrase** cannot self-recover (the no-escrow model, [§7.4](#74-recovery-key)); they are **re-enrolled by a group admin** — the admin issues **one new grant** to the member's new identity public key (transparency-verified as in *Member enrollment*). This is the only group-specific recovery seam.
