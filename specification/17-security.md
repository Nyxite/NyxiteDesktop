# 17 — Security (Device-Side)

The server-side threat model is covered by [server 13](https://github.com/Nyxite/server) (zero-knowledge: DB/blob theft, malicious operator, curious admin all yield no content). This document covers the **device-side** threats the desktop client must defend, since on a desktop the plaintext and keys *do* live locally — and the desktop, holding the **full corpus**, is the highest-value at-rest target across the platform.

## 17.1 Threats considered

| Threat | Mitigation |
|--------|------------|
| Lost/stolen unlocked machine | App lock (OS credential / Windows Hello / app passphrase), secure window, idle auto-lock. |
| Lost/stolen machine at rest | OS-keystore-wrapped DB master + identity keys; SQLCipher whole-DB encryption; app-lock factor beyond mere OS login. |
| Other OS users on a shared machine | Per-OS-user data paths + DPAPI `CurrentUser` / per-user Secret Service; app lock so another logged-in user (or one with the OS password) still can't unseal without the app factor. |
| Malicious/curious server or operator | E2EE: only ciphertext leaves the device; BLAKE3 verify on download; Ed25519 verify on directory entries/updates. |
| Tampering/withholding relay | Content-address verification + signature checks; updates that don't verify are rejected. |
| Backup/cloud-sync exfiltration | Store under app-private, non-roaming paths; document excluding the data dir from OS/file-sync backups; never write plaintext outside an explicit user export. |
| Screenshot/screen-share capture | Best-effort secure-window / exclude-from-capture where the OS supports it (configurable); honest that this is weaker on desktop than mobile. |
| Clipboard leakage | Mark sensitive copies; avoid auto-copying keys; clear copied share links from the clipboard after a timeout where feasible. |
| Other apps / IPC | Validate `nyxite://` / link inputs; single-instance forwarding only accepts well-formed share URLs; no content in logs. |
| Memory scraping | Zeroize key buffers (`CryptographicOperations.ZeroMemory`); hold unwrapped keys only in the `UserSession`. |
| User-initiated plaintext export | Explicit, warned, audited; never automatic; temp files best-effort securely deleted ([16 §16.7](16-offline-and-storage-policies.md)). |

## 17.2 Key & data protection

- **Root of trust**: the **OS keystore** (DPAPI `CurrentUser` on Windows / Secret Service on Linux) wraps the DB master key and the identity-key store ([07 §7.2](07-key-and-device-management.md)). Because these unseal on OS login, the **app lock** factor (OS credential / Windows Hello / Argon2id app passphrase) is what protects keys at rest beyond the login boundary — **default on, 10-minute idle window** (configurable 1–60 min, or every-launch), consistent with [07 §7.2](07-key-and-device-management.md) and [§17.3](#173-app-lock).
- **DB at rest**: SQLCipher with the keystore-protected passphrase; covers structure, decrypted names, metadata, and the FTS index. With multi-account ([14 §14.7](14-authentication.md)) there is **one DB and one master key per account**, so a compromise scoped to one account's key cannot read another's data; account removal destroys that account's DB, cache, index, tokens, and keystore-wrapped keys.
- **Blob store**: ciphertext is safe as-is; decrypted kept plaintext lives in the app-private store and is treated as sensitive (excluded from backup, evictable, cleared on "forget account/device"). The full corpus being present makes disciplined at-rest handling especially important.
- **Tokens**: OIDC tokens in the OS keystore, never in the DB/logs ([14](14-authentication.md)).
- **Recovery key & fragment keys**: never persisted in plaintext; recovery phrase shown once with copy/confirm and hard warnings ([07 §7.4](07-key-and-device-management.md)).

## 17.3 App lock

- Optional but recommended on by default: an OS-credential / Windows Hello / app-passphrase prompt on cold start and after a configurable idle timeout, before the DB/identity key is unlocked. This is the **primary at-rest defense** on desktop because the OS keystore alone unseals on login.
- A locked app shows no content; clear sensitive in-memory state on lock.

## 17.4 Network security

- TLS validation against the public chain; optional **certificate/public-key pinning** (`SocketsHttpHandler` SSL callback) with documented rotation ([05 §5.1](05-api-client.md)).
- Never trust the server to have validated content; verify locally ([06](06-cryptography.md), [08 §8.9](08-sync-engine.md)).

## 17.5 Logging & telemetry

- A scrubbing `Logger` facade over `Microsoft.Extensions.Logging`: never log plaintext, keys, tokens, fragments, recovery phrases, or full share URLs. Ciphertext sizes/IDs are acceptable for diagnostics.
- No third-party content-touching analytics. Any crash reporting is opt-in and PII-stripped ([02 §2.9](02-tech-stack-and-libraries.md)).

## 17.6 Secure UI & input

- Best-effort **exclude-from-capture** on editor/viewer/recovery/key windows where the OS exposes it (configurable); the spec is honest that desktop screenshot protection is weaker than mobile's `FLAG_SECURE`.
- Hide content under app lock; clear sensitive view state on lock.
- Validate all deep-link inputs (`nyxite://`, share URL token format, fragment decode) before use; reject malformed links; the single-instance forwarder treats forwarded args as untrusted.

## 17.7 Build & supply chain

- Trimming/ILLink keep directives for crypto/CRDT/serialization reflection; do not strip security checks. If NativeAOT is used, validate the shared crypto/CRDT/SQLCipher native interop under it ([18](18-build-ci-testing.md), [19](19-open-questions.md)).
- **NuGet package signature + lock-file verification**, pinned versions (Central Package Management), and dependency scanning in CI ([18](18-build-ci-testing.md)).
- Signed release artifacts (Authenticode/MSIX signing on Windows; signed AppImage/Flatpak/deb on Linux); signing keys in CI secrets/HSM, never committed ([18 §18.4](18-build-ci-testing.md)).

## 17.8 Honest limits

- Already-downloaded content can't be recalled after a share revocation (rotation only protects future content, [13 §13.5](13-sharing.md)).
- Desktop at-rest protection is **honestly weaker than mobile's hardware-backed keystores** ([07 §7.1](07-key-and-device-management.md)): DPAPI/Secret Service unseal on OS login, so a privileged on-device attacker who owns the logged-in session can reach what the session can. The app lock raises the bar but a rooted/admin-compromised machine can defeat at-rest protections.
- The full-corpus default concentrates value on the desktop; users storing especially sensitive material should consider per-subtree `dontKeep` plus app lock with a short idle window.
- v1.0.0 directory trust is TLS + Ed25519 signatures, not full key transparency ([13 §13.6](13-sharing.md)); deferred to Phase 6.
