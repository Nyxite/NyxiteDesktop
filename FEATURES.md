# Nyxite Desktop — Features

Avalonia (C#) client for Windows and Linux.

## Editing

- Markdown, handwritten ink, and plain-text editors

## Organization

- Project and folder navigation

## Sync

- Per-file sync policy controls: server-default, pinned-local, excluded
- Local storage for pinned-local and excluded files
- Offline editing

## Collaboration

- Live collaborative editing (shares the Ycs CRDT implementation with the server)

## Version history

- Browse version history and view diffs

## Sharing

- Create and manage share links

## Authentication

- Keycloak login with TOTP

## Open questions

- Sync policy semantics on the client: pinned-local = always-offline + synced; excluded = device-only, never uploaded? (from spec; default: as written)
- Offline conflict handling for last-write-wins content (ink and binary) on reconnect
- Ink capture and on-disk format — define once and share with Android for parity
- Ycs version and compatibility contract between desktop and server as both evolve
