# 15 — UI & Navigation

Avalonia 11 + Fluent theme, a resizable multi-window desktop layout, with the editors and full-corpus search as the showcase. Visual identity uses the Nyxite icon/brand assets from the master repo.

## 15.1 Shell, windows & navigation

- **Main window** uses a **list/detail** layout: a navigation pane (projects/folders tree + search) and a content pane (the open file). On wide windows both panes show; the pane split is user-adjustable.
- **Multiple documents**: the content pane supports **tabs**, and a file can be **torn out into its own window** (multi-window) for side-by-side editing — a desktop-class affordance the mobile/web clients lack. Each open text document runs its own relay room and Yrs doc ([09](09-realtime-collaboration.md)).
- **Navigation** is a typed in-app router driving the content pane; deep entry points (`nyxite://`, `https://nyxite.app/share/*`) resolve to the relevant view.
- **Single-instance**: a named mutex (Windows) / Unix domain socket lock (Linux) enforces one instance per OS user; a second launch (e.g. opening a share link) **forwards its arguments** to the running instance, which focuses and routes them ([13 §13.3](13-sharing.md), [09 §9.8](09-realtime-collaboration.md)).
- **System tray** (optional, default on): keep sync/collaboration alive in the background with a tray icon; closing the window minimizes to tray unless the user chose "quit on close." Tray menu: open, sync now, account switcher, quit.

### Top-level destinations

- **Auth** — `Login` (native: password or passkey/WebAuthn), `TOTP` step (native, verified by the Nyxite server), with **enterprise Keycloak SSO** as an optional path (system browser), `DeviceEnroll`, `DeviceApprove`, `RecoverySetup`, `RecoveryRestore`.
- **Browse** — `Projects`, `ProjectDetail`, `Folder` (file/folder list), `Search`.
- **Editor** — `TextEditor(fileId)`, `InkEditor(fileId)` (chosen by `contentType`), in tabs/windows.
- **History** — `Versions(fileId)`, `DiffViewer(fileId, fromSeq, toSeq)`.
- **Share** — `ShareManage(targetType, targetId)`, `OpenLink(token)` (deep-link entry).
- **Settings** — `Settings`, `Security`, `Devices`, `Storage`, `Accounts`, `About`.

**Account switcher**: because the app is **multi-account from v1.0.0** ([14 §14.7](14-authentication.md)), a persistent account switcher (window header / nav-pane header) shows the active account and lets the user switch or add one. Switching re-roots Browse to the newly active account's data and disposes the prior account's in-memory session ([01 §1.8](01-architecture.md)).

## 15.2 Key screens

| Screen | Contents |
|--------|----------|
| **Projects / Folder list** | Decrypted names, folders + files, content-type icons, sync/cache badges ([08 §8.8](08-sync-engine.md)), create/move/rename/delete (incl. drag-and-drop), **keep-on-device toggle at file/folder/project (cascading)** ([16 §16.2](16-offline-and-storage-policies.md)), exclude-from-sync, search box, refresh (delta sync). |
| **Text editor** | View/edit toggle, formatting toolbar (markdown), presence chips + remote carets ([09](09-realtime-collaboration.md)), connection state, snapshot/history/share/export actions, tabs/tear-out. |
| **Ink editor** | Skia canvas, tool palette (pen/highlighter/eraser, color, width), pages, undo/redo, zoom/pan, palm rejection; share/history. |
| **Version history** | Timeline, author, fetch+view a version, diff two versions (side-by-side), restore ([12](12-version-history.md)). |
| **Share manage** | Account + link shares, create/revoke, permissions/expiry, key fingerprint ([13](13-sharing.md)). |
| **Search** | Global query box + dedicated results view with snippets + scope hint; near-complete coverage on desktop ([11](11-search.md)). |
| **Settings** | Keep-on-device defaults (full-corpus default + opt-out subtrees), convenience-cache toggle/limit + clear, storage usage breakdown ([16 §16.3](16-offline-and-storage-policies.md)), security (app lock, OS-credential/Windows Hello, secure window), devices, recovery-key status, accounts (add/switch/remove), per-account instance host, **export / external-editor** options ([16 §16.7](16-offline-and-storage-policies.md)). |
| **Onboarding/enrollment** | Login, device enroll/approve, recovery-key creation with hard warnings ([07](07-key-and-device-management.md)). |

## 15.3 Status surfacing

- Compact per-file badges for sync state (synced/pending/offline/conflict/rotating/error) and cache state (kept / cached / title-only). Hovering/clicking explains and offers actions.
- Global indicators: offline banner, "N changes pending", relay connection state in editors, full-corpus prefetch/index progress.
- Conflicts (LWW ink/binary) surfaced as a non-destructive "two versions" prompt linking to history ([12 §12.5](12-version-history.md)).

## 15.4 Theming & brand

- Fluent theme with a Nyxite color scheme. **Dark theme first-class** (notes app, night use; fits the "Nyx" night branding); honor the OS light/dark preference by default.
- App icon, tray icon, and window icons from `Nyxite/icons/desktop` ([03 §3.5](03-project-structure.md)).
- Typography legible for long-form reading and code; respect OS font scaling / DPI on both platforms.

## 15.5 Accessibility & input

- Avalonia automation peers / screen-reader labels on all controls; sufficient contrast; OS text-scaling support.
- **Full keyboard model**: shortcuts for navigation, search (`Ctrl+K`), new file/folder, save/snapshot, view/edit toggle, tab switching; a command palette is a nice-to-have. Mouse, trackpad, and stylus coexist in the ink editor ([10 §10.4](10-editors.md)).
- Respect reduced-motion and the secure-window setting ([17](17-security.md)).

## 15.6 Empty/error/loading states

Every screen defines explicit loading, empty (no projects/files/results), offline, locked (needs unlock/enrollment), and error states; no silent blank screens.
