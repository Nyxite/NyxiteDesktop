# 13 — Sharing & Access Control

Two-layer model ([server 09](https://github.com/Nyxite/NyxiteServer)): a **server-enforced ACL** (who may reach the ciphertext/relay) plus a **cryptographic layer** (who can decrypt). The client manages both: it calls the share API for the ACL and performs the key wrapping for the crypto layer. Two share kinds: **account shares** (HPKE-wrapped FK) and **link shares** (FK in the URL fragment).

## 13.1 Account share (user grant)

1. The sharer enters/looks up the grantee (by email/userId). The client fetches the grantee's **public keys** from the directory: `GET /keys/directory?email=` (cache as `DirectoryKeyEntity`, [04](04-local-data-model.md)); verify the entry's **hybrid Ed25519 + ML-DSA-65** self-signature — both halves must verify ([06 §6.7](06-cryptography.md)).
2. The client **HPKE-wraps the file key** to the grantee's **hybrid X25519 + ML-KEM-768** public key (suite `X25519MLKEM768`, [06 §6.2](06-cryptography.md)).
3. `POST /shares` with `{ targetType, targetId, kind: user_grant, granteeId, permission: read|write }` and `POST /files/{id}/keys` (or the combined create-share payload) carrying the **wrapped key**.
4. The grantee's device later `GET /files/{id}/keys` → unwraps with its identity private key → can decrypt.
5. For a folder/project target, the relevant per-file keys are wrapped to the grantee (batched/lazily) so the subtree decrypts. The full-corpus desktop, which already holds and can unwrap every file in the subtree, is well-suited to perform this **bulk wrap** efficiently.

The server stores only the opaque wrapped blob; it never sees the FK.

## 13.2 Link share (anonymous / guest)

1. The client mints a high-entropy link **token** (≥128-bit) and uses the FK as the **fragment key** (≥256-bit), via `RandomNumberGenerator`.
2. It builds the URL: `https://nyxite.app/share/{token}#k=<base64url(FK)>`. The fragment after `#` is **never sent to the server**.
3. `POST /shares` with `{ targetType, targetId, kind: link, permission, expiresAt?, linkTokenHash }` — only the **hash** of the token is stored; the FK is not stored at all.
4. The client offers the URL via copy-to-clipboard and the OS share mechanism where available. The app warns that anyone with the link can access the file (read or write per the share) until it expires or is revoked, and that the key lives in the link.

## 13.3 Opening a link on this device (guest or member)

- The app registers the **`nyxite://` custom scheme** and a web-link handler for `https://nyxite.app/share/*` so links open the desktop app; single-instance forwarding routes the URL to the running instance ([15 §15.1](15-ui-and-navigation.md)). On open, it **parses the `#k=` fragment locally** (never transmits it), resolves `GET /share/{token}`, fetches ciphertext / joins the relay with a short-lived share session token, and decrypts with the fragment key ([09 §9.8](09-realtime-collaboration.md)).
- If the user is signed in, opening a link can optionally "save to my account" by creating an account share to themselves (wrapping the FK to their own public key). Otherwise it runs as a constrained guest session.
- Read-only links open the file view-only; write links allow editing.

## 13.4 Managing shares

- A share-management panel per file/folder/project: list active shares (`GET /shares?targetType={file|folder|project}&targetId={id}`), each with kind, grantee or "link", permission, expiry. Change permission/expiry via `PATCH /shares/{id}`; revoke via `DELETE /shares/{id}`.
- Locally created link URLs may be re-displayed only if the user chose to keep them (stored in the OS keystore / encrypted prefs and clearly labeled); otherwise the full URL (with fragment) cannot be reconstructed by the server or app after creation.

## 13.5 Revocation (two-layer)

1. **Instant ACL cutoff**: `DELETE /shares/{id}` removes the grant; the removed member/link can no longer fetch ciphertext or join the relay — enforced at fetch/join/submit.
2. **Forward-secrecy rotation**: the client (a remaining member) **rotates the file key** so future content is unreadable to the removed party ([07 §7.6](07-key-and-device-management.md)): new FK, re-encrypt head + new snapshot, re-wrap to remaining members, `POST /files/{id}/keys/rotate`. Runs in `KeyRotationService`; the file shows `Rotating`.
- The UI is honest: already-downloaded content can't be recalled; rotation protects only future edits.

## 13.6 Trust & limits (v1.0.0)

- Directory trust is **TLS + hybrid Ed25519 + ML-DSA-65 self-signature** on entries; **key transparency / safety-number verification is deferred to Phase 6**. Until then, surface the grantee's key fingerprint so cautious users can compare out-of-band.
- The client respects server rate limits on `GET /keys/directory`, `POST /shares`, and `/share/{token}` access (`429` backoff, [05](05-api-client.md)).
- Fragment keys (≥256-bit) and link tokens (≥128-bit) are generated with a CSPRNG; the app warns that links in chat/history can leak the fragment, so prefer account shares for sensitive material and use short expiries for links.

## 13.7 Group sharing (enterprise/family)

Beyond one-off account and link shares, files can be shared with a **group** via the group-key layer ([06 §6.10](06-cryptography.md), [07 §7.10](07-key-and-device-management.md); feature [groups.md](https://github.com/Nyxite/Nyxite), build steps **P4.4-DSK-1/2**). It serves two archetypes — **family** (every member holds a distinct personal key but all read the same shared data) and **enterprise** (a **managers** group reads all of a team's files; a worker reads only their own) — and **coexists** with per-file account/link shares (groups for team/family scale; account shares for one-off single-person shares). Keys are **flat** (no nesting) and **scoped per project/time-period**.

### Group-management UI
- A **Groups** panel (in `Feature.Sharing`, [15](15-ui-and-navigation.md)) lets a user **create a group** (generate the group keypair, publish its public half — [07 §7.10](07-key-and-device-management.md)), list its members and scopes, **enroll** a member, and **remove** a member. It surfaces the group's public-key fingerprint and each member's transparency-verified status.
- **Enroll** shows the newcomer's identity-key fingerprint and its **key-transparency inclusion-proof result (Phase 4.3)**; enrollment is blocked with a clear error if the proof fails (a directory-substituted key), since one bad key would expose the whole group's corpus ([07 §7.10](07-key-and-device-management.md)). A successful enroll is **one grant blob** — the UI states it instantly grants every file the group can read (no per-file wait), unlike a subtree account share (§13.1, batched per file).
- The panel reflects the **server-enforced group-size limit** (and the admin per-group override): an over-limit enroll is rejected server-side and surfaced.

### Wrapping a file / subtree to a group
- **Grant one file to a group**: `WrapDekToGroup` seals the file's DEK to the group public key ([06 §6.10](06-cryptography.md)) and stores one DEK-to-group blob via the wrapped-key API; the group ACL is set alongside (the two-layer model, as §13.1).
- **Grant a subtree**: for a folder/project target, wrap each in-scope file's DEK to the group (batched/lazily). The full-corpus desktop already holds every FK in the subtree, so it drives this **bulk wrap** efficiently — but note enrollment itself never needs this: adding a *member* is O(1) (one group-key grant), and only *first-time* group grants on files cost a per-file wrap.

### Reader-group attachment & auto-wrap on create
- A **project or folder** may carry a **reader-group attachment** naming a group whose **public key new files are auto-wrapped to** on creation, in addition to the author's own key — this drives the enterprise "manager reads all" path. It rides the **existing per-project/folder/file cascade** — `inherit` / a specific group / none — the same inheritance as sync policy and plaintext-at-rest ([16 §16.8](16-offline-and-storage-policies.md)).
- **On file create** (extends the create path of [07 §7.5](07-key-and-device-management.md)): the client resolves the effective attachment for the file's scope and, if a group is attached, wraps the new FK to **the author's public key AND the attached group's public key** (`WrapDekToGroup` against the directory public key — no membership in that group required). A worker's file is thus readable by the worker and by the managers group; another worker has **no key path** (cryptographically locked out). Client-enforced; the server stores only the opaque attachment as structure metadata.

### Revocation (honest UI)
Removing a group member reuses the two-layer model of §13.5 at the group-key level, driven by `GroupKeyRotationService` ([07 §7.10](07-key-and-device-management.md)):
1. **Instant ACL cutoff / soft** — delete the member's group-key grant (`DELETE /groups/{id}/members/{uid}`); they can no longer fetch new group blobs.
2. **Forward-secrecy rotation** — a remaining member rotates the **affected scope's** group key (new generation, re-wrap to remaining, optional DEK re-seal); the group/scope shows `Rotating`. Only that scope is touched.
- The UI is **honest, exactly as §13.5**: already-decrypted content **can't be recalled**; rotation protects only content produced after it. This caveat is surfaced in the removal dialog.
- **Recovery composes for free**: a member who recovers their identity key regains group access automatically (grants unwrap under the recovered personal key); a member who lost everything is **re-enrolled** by a group admin (one new grant) — [07 §7.10](07-key-and-device-management.md).
