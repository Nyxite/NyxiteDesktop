# 06 — Cryptography

This is the most consequential module (`Nyxite.Desktop.Crypto`). On desktop it is a **thin adapter over the shared `Nyxite.Crypto` package** — the *same code the server is built from* ([README](README.md)) — so the framing and HPKE suite are **byte-identical to the server by construction**, not by re-derivation. The adapter adds only desktop concerns: buffer handling, streaming for large blobs, and integration with the OS keystore ([07](07-key-and-device-management.md)). Everything here mirrors [server 07](https://github.com/Nyxite/server).

## 6.1 Posture

The desktop is a full cryptographic peer. It generates file keys, encrypts/decrypts all content, wraps/unwraps keys, computes content addresses, signs and verifies, and produces snapshots. The server is blind: it stores ciphertext, wrapped keys, public keys, opaque IDs, sizes, timestamps — and nothing that lets it read content.

## 6.2 Algorithm table (the shared canonical set)

These are the server's canonical primitives, reproduced for reference. Because the desktop consumes `Nyxite.Crypto`, it inherits them exactly; the right-hand column names the .NET provider inside that shared package.

| Purpose | Algorithm | .NET provider (in `Nyxite.Crypto`) |
|---------|-----------|-------------------------------------|
| Content / CRDT / snapshot encryption | **AES-256-GCM** (96-bit nonce, 128-bit tag) | `System.Security.Cryptography.AesGcm` (BCL) |
| Public-key wrap (file-key to a member, device enrollment to a device pubkey) | **HPKE**: DHKEM(X25519, HKDF-SHA256) / HKDF-SHA256 / AES-256-GCM — RFC 9180 IDs `KEM=0x0020`, `KDF=0x0001`, `AEAD=0x0002` | the shared HPKE impl (BouncyCastle / vetted RFC 9180) |
| Symmetric wrap (recovery blob) | **AES-256-GCM** under an Argon2id-derived key | `AesGcm` ([§6.4](#64-hpke-wrapping)) |
| Identity key agreement | **X25519** | shared lib (BouncyCastle/NSec) |
| Signing (updates, key-directory entries) | **Ed25519** | shared lib (BouncyCastle/NSec) |
| Recovery-key derivation | **Argon2id → wrapping key** (m = 64 MiB, t = 3, p = 1; tunable, stored in `recovery_blobs.kdf_params`) | `Konscious.Security.Cryptography.Argon2` / libsodium |
| Plaintext hashing (content address) | **BLAKE3-256** | `Blake3` (official Rust binding) |

These values are **pinned to the server's canonical ledger** and are identical across clients; because the desktop and server share the implementation, drift would require changing `Nyxite.Crypto` itself, which **breaks the shared conformance vectors and fails CI on all clients** ([18 §18.5](18-build-ci-testing.md)).

**System rule — HPKE vs AES-256-GCM**: use **HPKE wherever the target is a public key** (file-key wrap to members, device enrollment to a device pubkey); use **AES-256-GCM wherever the key is symmetric** (all content + the recovery blob).

## 6.3 Encrypted object framing

Every ciphertext object (blob, snapshot, CRDT update, encrypted name/title, encrypted settings/metadata) uses the shared frame, implemented once in `Nyxite.Crypto`:

```
magic(4) | version(1) | key_id(16) | nonce(12) | ciphertext(...) | gcm_tag(16)
```

- `magic` — fixed 4-byte Nyxite identifier: ASCII **`"NYXC"`** (`0x4E 0x59 0x58 0x43`).
- `version` — 1-byte frame/crypto-agility version, currently **`0x01`**; the client refuses to *misread* a newer version it doesn't understand and surfaces an "update required" state.
- `key_id` — 16-byte identifier (uuid) of the **specific file key** used. The server cannot resolve it to a usable key; the client maps it to a local FK handle. `key_id` is 1:1 with the integer `KeyGeneration` rotation counter ([04 §4.2](04-local-data-model.md)) and is the stable frame/crdt/version reference.
- `nonce` — 96-bit, **unique per (key, message)**. Use a CSPRNG; never reuse a nonce under the same key (a hard invariant for GCM). For high-volume CRDT updates under one FK, ensure nonce uniqueness via random 96-bit nonces (collision-negligible) — documented and tested.
- **AAD** = `magic ‖ version ‖ key_id ‖ file_id(16) ‖ object_kind(1)`, binding the frame to its file and object kind. Decryption supplies the same AAD; a mismatch fails authentication, preventing cross-context reuse of ciphertext. The `object_kind` enum (1 byte): `blob = 1`, `snapshot = 2`, `crdt = 3`, `name = 4`, `metadata = 5`, `awareness = 6`, `settings = 7`.

`ICryptoEngine` (the desktop adapter) exposes:
```csharp
byte[] Seal(ReadOnlySpan<byte> plaintext, FileKeyHandle fk, ObjectKind kind, Guid fileId);   // → frame
byte[] Open(ReadOnlySpan<byte> frame, FileKeyHandle fk, ObjectKind kind, Guid fileId);        // → plaintext
// streaming overloads (Stream → Stream) for large ink/binary blobs (§6.6, 16 §16.5)
```

## 6.4 HPKE wrapping (account shares & key recovery)

> Naming note: this section also covers the **symmetric** recovery wrap (AES-256-GCM); the rule is HPKE for public-key targets, AES-256-GCM for symmetric keys (§6.2).

- To share a file to a user (account share), wrap an FK to the owner, or enroll a new device, the client fetches the recipient's **X25519 public key** from the directory ([13](13-sharing.md)) and HPKE-seals the payload to it. The result is the `wrapped_key`/`wrappedIdentityKey` blob stored server-side; only the recipient's device can HPKE-open it with the identity private key.
- HPKE **suite is DHKEM(X25519, HKDF-SHA256) / HKDF-SHA256 / AES-256-GCM** — RFC 9180 IDs `KEM=0x0020`, `KDF=0x0001`, `AEAD=0x0002`. Since this is the shared server implementation, suite agreement is automatic; a **conformance test still wraps with desktop-`Nyxite.Crypto` and unwraps with the server's/Android's/web's HPKE (and vice-versa)** using fixed vectors ([18](18-build-ci-testing.md)).
- **Recovery escrow uses AES-256-GCM, not HPKE** (DECISION, [OD recovery-wrap]). The recovery blob is the identity bundle (X25519 priv ‖ Ed25519 priv) sealed with **AES-256-GCM under the Argon2id-derived wrapping key** ([07 §7.4](07-key-and-device-management.md)). This follows the system rule (§6.2): HPKE only where the target is a public key; AES-256-GCM where the key is symmetric.

## 6.5 File keys

- An FK is a **256-bit AES-GCM key generated by a CSPRNG** (`RandomNumberGenerator`) on the device when a file is created.
- It encrypts that file's bytes, CRDT updates, snapshots, and encrypted name. It is **never sent unwrapped**; it is stored on the server only as `wrapped_key` rows (one per member) and/or carried in a link fragment.
- **No convergent encryption** — the FK is random, not derived from content (resists confirmation-of-file attacks). Intra-file dedup still works because identical plaintext under the same FK yields identical ciphertext at the same content address.
- In memory, an unwrapped FK is held as a `FileKeyHandle` wrapping a pinned/zeroizable buffer; it is never persisted in plaintext ([04 §4.2](04-local-data-model.md), [17](17-security.md)).

## 6.6 Content addressing

- `address = BLAKE3-256(plaintext)`, computed **on the client**. The ciphertext (framed) is stored under that address; the server treats the address as opaque and enforces write-once.
- On download, the client decrypts, recomputes BLAKE3 over the plaintext, and **verifies it matches the claimed address** before trusting the blob (defends against a tampering/withholding relay returning the wrong bytes).
- BLAKE3 is chosen for speed on large blobs (ink/binary, snapshots); the desktop **streams** hashing and encryption for large inputs to bound memory ([16 §16.5](16-offline-and-storage-policies.md)).

## 6.7 Signing & verifying (tamper detection)

- The client **signs** its key-directory entry (its published X25519/Ed25519 public keys) and, where the protocol calls for it, CRDT updates / share grants with **Ed25519**, so peers can detect a relay that swaps keys or injects updates.
- The client **verifies** Ed25519 signatures on directory entries it fetches before wrapping a share to them, and on relayed updates where signed. (Full key-transparency/safety-numbers is deferred to Phase 6, [13 §13.6](13-sharing.md); v1.0.0 trust is TLS + signature checks.)

## 6.8 Recovery-key derivation

- The recovery key is a high-entropy phrase shown once. The wrapping key is derived via **Argon2id** with parameters **m = 64 MiB, t = 3, p = 1** (tunable; the actual values are stored non-secret in `recovery_blobs.kdf_params`). The client reads those params for unwrap and uses them when (re)generating the escrow so any device can recover ([07 §7.4](07-key-and-device-management.md)).
- The derived key then seals the escrow with **AES-256-GCM** (not HPKE — symmetric key, per §6.4). The recovery-blob shape is `{ version, kdf:{alg:"argon2id", m, t, p, salt(16)}, nonce(12), ciphertext, tag(16) }` with **AAD = `userId ‖ version`** ([07 §7.4](07-key-and-device-management.md)).
- Desktop has ample CPU/RAM, so the canonical `(m=64 MiB, t=3, p=1)` runs comfortably; the client must still honor the **server-stored params** rather than its own, so a blob written by a mobile device (possibly tuned lower) round-trips ([Android 06 §6.8](https://github.com/Nyxite/android)).

## 6.9 Implementation rules

- **One code path** for framing (`Seal`/`Open`), provided by `Nyxite.Crypto` and used by every content type and every transport (REST blob, relay update, snapshot, name, settings). No ad-hoc encryption elsewhere.
- **No homemade crypto.** Only the shared `Nyxite.Crypto` package and the vetted primitive libraries in [02 §2.7](02-tech-stack-and-libraries.md), behind the `ICryptoEngine` interface for agility.
- **Constant-time** comparisons for tags/tokens (use `CryptographicOperations.FixedTimeEquals`).
- **Zeroize** key material buffers after use (`CryptographicOperations.ZeroMemory`); hold unwrapped keys only in the `UserSession`.
- **Known-answer tests** for AES-GCM framing, HPKE wrap/unwrap, Ed25519, X25519, BLAKE3, and Argon2id, plus **cross-client conformance vectors** shared with server/android/web ([18 §18.5](18-build-ci-testing.md)). Because the desktop *is* the server's crypto code, these tests double as the canonical guard that a `Nyxite.Crypto` change stays interoperable.

## 6.10 Group-key wrapping (enterprise/family sharing)

Group sharing inserts **one optional middle layer** into the wrap hierarchy so a file readable by a whole group is wrapped once instead of per member ([13 §13.7](13-sharing.md), [07 §7.10](07-key-and-device-management.md); feature [groups.md](https://github.com/Nyxite/Nyxite), build step **P4.4-DSK-1/2**):

```
personal key  →  wraps  →  group key  →  wraps  →  DEK (FK)  →  encrypts  →  file
```

This adds **no new primitive** — a group public key is just another HPKE target, exactly like a member or device public key. Everything below is the same `Nyxite.Crypto` code and the same pinned suite as §6.2/§6.4; the framing (§6.3) and the content/DEK layer (§6.5) are unchanged.

- **Group keypair.** A group is an **X25519 (HPKE) + Ed25519 (sign)** keypair generated client-side by the creator via `RandomNumberGenerator` / `Nyxite.Crypto` — the *same* keygen as an identity keypair (§6.7). Its **public** half is published as a directory HPKE target anyone may wrap to; its **private** half is never sent unwrapped — it lives on the server only as per-member wrapped blobs.
- **Wrap the group private key to a member (the group-key grant).** `HPKE-seal(memberX25519Pubkey, groupPrivBundle)` where `groupPrivBundle = X25519 priv ‖ Ed25519 priv` — the DHKEM(X25519,HKDF-SHA256)/HKDF-SHA256/AES-256-GCM suite (§6.2). The resulting **grant blob** carries `group_id | scope_id | member_id | generation | alg_id | hpke_ct` and is **append-only** (rotation bumps `generation` and appends, §7.10). No grant ever contains a plaintext key.
- **Wrap a DEK to a group (DEK-to-group).** `HPKE-seal(groupX25519Pubkey, FK)` produces a wrapped-key row whose **principal is a group** (per scope/generation), stored alongside the existing per-member `wrapped_key` rows (§6.5). A worker can wrap to the managers-group public key with **no membership in it** — only the directory public key is needed (the enterprise "manager reads all" path, [13 §13.7](13-sharing.md)).
- **Unwrap chain (read path).** `HPKE-open` the group-key grant with the **identity private key** → recover the group private key → `HPKE-open` the DEK-to-group wrap with the group private key → recover the `FileKeyHandle` → `Open` the frame (§6.3). Both unwraps run entirely on-device; the server holds only opaque blobs.
- **`alg_id`-tagged wrap format.** Every group-wrapped blob (grant **and** DEK-to-group) carries an explicit **`alg_id`** naming the wrap algorithm, for crypto-agility. Because a group key wraps *many* DEKs it concentrates any future post-quantum blast radius, so the format is algorithm-agile from day one even though v1 ships the classical pinned suite — the same `ICryptoEngine`-behind-an-interface agility rule as §6.9.

`ICryptoEngine` gains group overloads, still delegating to the shared package:
```csharp
GroupKeyPair GenerateGroupKeyPair();                                             // X25519 + Ed25519, via Nyxite.Crypto
byte[] WrapGroupKeyToMember(GroupKeyPair group, ReadOnlySpan<byte> memberPubkey); // → group-key grant (alg_id-tagged)
GroupKeyHandle UnwrapGroupKey(ReadOnlySpan<byte> grant, IdentityKeyHandle identity);
byte[] WrapDekToGroup(FileKeyHandle fk, ReadOnlySpan<byte> groupPubkey);          // → DEK-to-group wrap (alg_id-tagged)
FileKeyHandle UnwrapDekViaGroup(ReadOnlySpan<byte> dekToGroup, GroupKeyHandle group);
```
Group private keys and unwrapped `GroupKeyHandle`s follow the same zeroize-after-use / in-session-only rules as identity keys and FKs (§6.9, [07 §7.2](07-key-and-device-management.md)); the group wrap/unwrap pairs join the **cross-client conformance vectors** (§18.5) so a desktop-wrapped grant/DEK opens on server/android/web and vice-versa.
