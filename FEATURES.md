# Nyxite Desktop — Features

Avalonia (C#) client for Windows and Linux. Shares domain models, DTOs, encryption, and CRDT code with the server.

**Privacy first / full E2EE.** Encryption, decryption, CRDT merge, and snapshotting happen **on the client**. The server only ever sees ciphertext. The desktop holds the user's keys, decrypts content for display, and re-encrypts edits before upload. With room for the full local corpus, the desktop is the **primary full-text search surface** and the reference for "search everything."

## Editing

- Markdown, handwritten ink, and plain-text editors
- View and edit modes

## Organization

- Project and folder navigation (names decrypted locally; stored encrypted on the server)

## Sync

- Per-file sync policy controls: server-default, excluded — plus a client-local "keep on device" pin the zero-knowledge server never sees
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

## Encryption & keys

- Local AES-256-GCM content encryption and HPKE wrap/unwrap, via the shared C# crypto code
- Device enrollment and identity-key handling; recovery via the user's recovery phrase unwrapping the client-encrypted recovery blob; keys protected by the OS keystore where available

## Authentication

- Keycloak login with TOTP (account auth; decryption governed by local keys)

## Open questions

See [../docs/OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md). Desktop-specific:

- Local key storage (DPAPI on Windows / Secret Service or keyring on Linux), device enrollment, and recovery-phrase UX
- Offline conflict handling for last-write-wins content (ink/binary) on reconnect
- Client-side snapshot/compaction policy for the encrypted CRDT log (thresholds, who snapshots)
- Local full-text index technology and incremental update as encrypted content syncs in
- Ink capture and on-disk **encrypted** vector stroke format — define once and share with Android
- ydotnet binding maturity — validate early (community-maintained); now used for client-side merge and snapshotting
