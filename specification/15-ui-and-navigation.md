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
- **Support** — `ReportBug`, `MyTickets` (present **only when the server advertises `support.enabled`**; see [§15.7](#157-bug-reporting--support)).

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

- The UI implements the shared **NyxiteDesign** system ([OPEN-DECISIONS DS](https://github.com/Nyxite/Nyxite), [`NyxiteDesign`](https://github.com/Nyxite/NyxiteDesign)) end-to-end: **Layer A** = the token set (deep-purple accent, semantic light/dark themes, spacing/radius/shadow/motion) applied to Avalonia's standard controls (Button, TextBox, ToggleSwitch, Slider, TabControl, ComboBox, DataGrid, …); **Layer B** = the Nyxite app shell — the Document/Presentation/Spreadsheet **rail**, the three toolbar densities (classic/slim/minimal), and the canvas. Bespoke Nyxite chrome (rail, toolbar modes, segmented control, command palette, comment thread, presence) is built as **custom Avalonia `ControlTheme`s**, not one-off styling.
- Token values (colors, typography, spacing) are **CI-generated from `nyxite-tokens.json`**, not hand-authored, and consumed as `DynamicResource`; **light/dark** is Avalonia `ThemeVariant`, honoring the OS preference by default ([02 §2.2](02-tech-stack-and-libraries.md), [03 §3.5](03-project-structure.md)). **Dark theme first-class** (notes app, night use; fits the "Nyx" night branding).
- App icon, tray icon, and window icons from `Nyxite/icons/desktop` ([03 §3.5](03-project-structure.md)).
- Typography is **Manrope** (UI) + **Source Serif 4** (document content), legible for long-form reading and code; respect OS font scaling / DPI on both platforms. **Fonts are bundled with the app** (self-hosted assets) and **never fetched from Google Fonts or any external CDN at runtime** — a privacy requirement, since a runtime font fetch would leak usage to a third party ([17](17-security.md)).

## 15.5 Accessibility & input

- Avalonia automation peers / screen-reader labels on all controls; sufficient contrast; OS text-scaling support.
- **Full keyboard model**: shortcuts for navigation, search (`Ctrl+K`), new file/folder, save/snapshot, view/edit toggle, tab switching; a command palette is a nice-to-have. Mouse, trackpad, and stylus coexist in the ink editor ([10 §10.4](10-editors.md)).
- Respect reduced-motion and the secure-window setting ([17](17-security.md)).

## 15.6 Empty/error/loading states

Every screen defines explicit loading, empty (no projects/files/results), offline, locked (needs unlock/enrollment), and error states; no silent blank screens.

## 15.7 Bug reporting & support

In-app bug reporting routing to the maintainer-run `NyxiteSupport` helpdesk. This runs on the project's **one deliberate, consensual non-E2EE support plane**, disjoint from the content plane — see [17 §17.9](17-security.md), the master feature [support.md](https://github.com/Nyxite/Nyxite), the [NyxiteSupport `specification/02`](https://github.com/Nyxite/NyxiteSupport), and [OPEN-DECISIONS SUP-1–SUP-9](https://github.com/Nyxite/Nyxite).

- **Capability-gated surface (SUP-9).** The **"Report a bug"** entry (in the nav / `About` / help affordances) and the **"My tickets"** view appear **only when the server advertises the `support.enabled` capability flag** (v1 = the maintainer's official instance(s) only). Where the flag is absent the surfaces are **simply absent** — no disabled control, no hint.
- **Report composer.** Free-text **title + description**, plus an optional screenshot and the diagnostic envelope below.
- **Screenshot capture + destructive redaction (SUP-2).** Optional capture of the current view via a **native app-window grab**, opened in a redaction editor offering **black-box** and **blur** tools. Redaction is **destructive and client-side**: the redacted regions are **flattened into the pixels (re-encoded) before upload**, EXIF/metadata stripped; the **original image and any redaction mask are never sent** — there is no peel-back layer. Redaction is manual (the user decides what to hide).
- **Consent + destination notice before send (SUP-1).** Before a report can transmit, the user must confirm a clear notice that — **unlike their files — this report is *not* end-to-end encrypted and goes to the Nyxite maintainer**, shown alongside a GDPR disclosure. A report carries **no content key and no content-plane ciphertext**.
- **User-reviewable, editable diagnostic envelope.** A non-content technical bundle shown for **review + edit** before send: app version/build, platform/OS, locale, the **current screen/route id** (a UI location, never a file/project name or any content), **scrubbed** client-side error logs / recent stack traces, and coarse connection/relay state. Nothing is attached that is not represented in this envelope.
- **"My tickets" view (SUP-3).** An account user sees their own reports' status, the operator's **public replies**, and gets **in-app notifications** of updates — a genuine two-way thread, not fire-and-forget. Guests (share/guest sessions) instead supply an email + optional name and receive replies via email + a tokenized no-login link (SUP-6).
- **Submission via the server as an authenticating relay (SUP-7).** The client **never contacts the helpdesk directly**: it submits to its own `NyxiteServer`, which authenticates the submitter and relays to `NyxiteSupport` tagged with the instance fingerprint + an opaque user reference. Submission is best-effort and off the critical path; a transport failure surfaces a retryable error and never blocks the app.
