# Nyxite Desktop — Features

Avalonia (C#) client for Windows and Linux. Shares domain models, DTOs, encryption, and CRDT code with the server.

**Privacy first / full E2EE.** Encryption, decryption, CRDT merge, and snapshotting happen **on the client**. The server only ever sees ciphertext. The desktop holds the user's keys, decrypts content for display, and re-encrypts edits before upload. With room for the full local corpus, the desktop is the **primary full-text search surface** and the reference for "search everything."

## Editing

- Markdown, handwritten ink, and plain-text editors
- View and edit modes

## Organization

- Project and folder navigation (names decrypted locally; stored encrypted on the server)
- **Trash** — deleted items move to a visible Trash with **one-click restore** for the retention window (default 30 d), then leave the UI; a delete never purges immediately (staged Trash → grace → purge, DL-1–DL-5). A share-recipient's delete removes only their own copy; the owner's delete removes it for everyone. See [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (DL-1–DL-5).

## Sync

- Per-file sync policy controls: server-default, excluded — plus a client-local "keep on device" pin the zero-knowledge server never sees
- **Background auto-sync service** — auto-pulls server changes, configurable per project/folder/file (the client-local cascade; the server never sees it)
- **Plaintext-at-rest (desktop only)** — optional toggle to keep synced files as **plaintext on local disk**, tri-state (`inherit` / `plaintext` / `encrypted`) cascading per project/folder/file, root default encrypted. Client-local only; the server still stores ciphertext. For toggled scopes plaintext **replaces** the local copy; **version history stays encrypted**; §15 data-safety applies (server ciphertext is the durable backstop, integrity-checked on read). (See [SPECIFICATION §6, §15](https://github.com/Nyxite/Nyxite/blob/main/docs/SPECIFICATION.md).)
- Local storage for pinned and excluded files; offline editing
- Content is encrypted locally before upload and decrypted after download (the server moves ciphertext only)

## Collaboration

- Live collaborative editing — **client-side CRDT merge** (ydotnet; the shared C# CRDT glue also backs the server-side CRDT conformance test harness `Nyxite.CrdtConformanceTests`, not live merging); the server is a blind **encrypted relay**
- The client produces the **encrypted snapshots** that compact the update log and feed version history (the server cannot, since it can't read the log)

## Search

- **Client-side** full-text search over the full local corpus (e.g. a local encrypted index); the primary, complete search experience across the platform

## Version history

- Browse version history with **client-side diffs** (snapshots decrypted and diffed locally); restore

## Sharing

- Create and manage share links (link key in the URL fragment; account shares wrap the file key to the recipient's public key via HPKE)

## Group sharing (enterprise/family)

- Group management ([features/groups.md](https://github.com/Nyxite/Nyxite/blob/main/features/groups.md)): generate a group keypair, enroll/remove members, and unwrap the group key → unwrap DEKs, all client-side; enrollment **verifies the member's public key against the key-transparency log** before wrapping
- Wrap a file/subtree DEK to a group public key; honor the per-project/folder **reader-group attachment** — auto-wrap new files to the attached group's public key (the enterprise "manager reads all" path) via the shared C# crypto code
- `GroupKeyRotationService` — scope-scoped group-key rotation on member removal (re-wrap to remaining, optional DEK re-seal); honest UI that already-decrypted content can't be recalled
- Recovering the identity key restores group access automatically (group grants unwrap under the recovered personal key)

## Encryption & keys

- Local AES-256-GCM content encryption and HPKE wrap/unwrap, via the shared C# crypto code
- Device enrollment and identity-key handling; recovery via the user's recovery phrase unwrapping the client-encrypted recovery blob; keys protected by the OS keystore where available

## Authentication

- **Native login** — password + required TOTP, or **passkeys (WebAuthn)** (account auth; decryption governed by local keys). Enterprise Keycloak/OIDC SSO is a pluggable option. (See [SPECIFICATION §10](https://github.com/Nyxite/Nyxite/blob/main/docs/SPECIFICATION.md).)

## Bug reporting & support

- **"Report a bug"** — an in-app report composer, shown when the instance has reporting enabled (a server `support.enabled` capability flag, **on by default** — every instance can file, routing to the maintainer's central desk or, if the operator opted into one, their own desk — SUP-9 / SUP-10–SUP-13).
- **Screenshot capture + destructive redaction** — optionally attach a screenshot (a native grab of the app window); a redaction editor with **black-box + blur** tools **flattens redacted regions into the pixels before upload**, so the original image and mask never leave the device (SUP-2).
- **Consent + destination notice** — before sending, a clear notice that, unlike your files, the report is **not end-to-end encrypted** and goes to the **Nyxite maintainer**, plus a GDPR disclosure (SUP-1); a **user-reviewable diagnostic envelope** (app version/build, platform, locale, current screen id — never content, scrubbed logs, connection state) is editable before send.
- **"My tickets"** — track your own reports' status and support replies with in-app notifications; submission goes through the server as an **authenticating relay** (the client never contacts the helpdesk directly — SUP-3/SUP-7).
- Runs on the **consensual, non-E2EE support plane** — disjoint from content, carrying no content key or content-plane ciphertext. Detailed in [Nyxite Support](https://github.com/Nyxite/Nyxite/blob/main/features/support.md) / the `NyxiteSupport` repo `specification/`; decisions in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (SUP-1–SUP-13).

## Open questions

See [../docs/OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md). Desktop-specific:

- Local key storage (DPAPI on Windows / Secret Service or keyring on Linux), device enrollment, and recovery-phrase UX
- Offline conflict handling for last-write-wins content (ink/binary) on reconnect
- Client-side snapshot/compaction policy for the encrypted CRDT log (thresholds, who snapshots)
- Local full-text index technology and incremental update as encrypted content syncs in
- Ink capture and on-disk **encrypted** vector stroke format — define once and share with Android
- ydotnet binding maturity — validate early (community-maintained); now used for client-side merge and snapshotting
