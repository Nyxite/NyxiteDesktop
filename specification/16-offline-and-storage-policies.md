# 16 — Offline & Local Storage

Each file is encrypted under its **own key** and fetched independently, so local storage is a set of **per-file local copies the user controls**. Unlike the conservative mobile default, the **desktop default is full-corpus**: keep, decrypt, and index everything the account can read — the desktop is the platform's "everything is here, searchable, offline" surface ([11](11-search.md)). The user can still opt heavy subtrees out. Everything kept on-device stays **encrypted at rest** ([17](17-security.md)).

## 16.1 What is stored where

| Data | Location | Lifetime |
|------|----------|----------|
| Structure, names, metadata, sync state, FTS index | SQLCipher DB | Persistent (the whole tree is always available) |
| **Kept** files (text + ink), decrypted for use, **encrypted at rest** | App-private files / DB, OS-keystore-gated | Kept until the user opts the subtree out ([§16.2](#162-keep-on-device-the-core-control)) |
| Opened-but-not-kept files (in `dontKeep` subtrees) | Convenience store | Until closed, or per the convenience-cache setting ([§16.3](#163-on-demand-fetch--optional-convenience-cache)) |
| Fetched version snapshots | Convenience store | Evictable; re-fetchable |
| Decrypted working copy of the open file | Memory + temp | Session only |
| **User-exported plaintext working copy** | A user-chosen path (outside the boundary) | Until the user/round-trip removes it ([§16.7](#167-plaintext-export--external-editor-interop-desktop-only)) |
| Keys/tokens | OS keystore | Per [07](07-key-and-device-management.md), [14](14-authentication.md) |

Data paths are per-OS: Windows `%LOCALAPPDATA%\Nyxite`, Linux `$XDG_DATA_HOME/nyxite` (default `~/.local/share/nyxite`); config under `%APPDATA%` / `$XDG_CONFIG_HOME`. All of the above are **per-account** ([14 §14.7](14-authentication.md)): each account has its own DB, file store, FTS index, and key/token stores, isolated by `accountId`; removing an account frees all of its local storage.

## 16.2 Keep-on-device (the core control)

The user controls what is kept on this device, at **three granularities**, with cascading — but on desktop the **account default is `keep`** (full corpus), the inverse of mobile:

- **File** — a per-file keep/don't-keep toggle.
- **Folder** — keep or don't-keep the whole folder; applies to all files and **becomes the default for new files** added to it.
- **Project** — keep or don't-keep the whole project; cascades to all folders/files and to anything added later.

Cascade rules:
- The **account default is keep**; a user typically opts *out* specific heavy subtrees (`dontKeep`) rather than opting in.
- A file's effective state = its own explicit setting if present, else the nearest ancestor's setting, else the account default (**keep**).
- New files inherit their parent's effective setting.

Behavior of a **kept** file: proactively downloaded, decrypted, **indexed for search** ([11](11-search.md)), and available fully offline. It remains encrypted at rest; "keep on device" controls *availability*, not whether it's encrypted.

This is a **client-local, per-device** choice (each of the user's devices keeps its own selection) and is **never sent to the server**. It is *not* a server sync policy: every kept and not-kept file is `server-default` server-side ([08 §8.2](08-sync-engine.md)). The separate server **`excluded`** choice ("device-only, never upload to the server") remains available per file for content the user never wants to leave the device. (`pinned-local` is **not** a sync policy — offline pinning is exactly this `KeepOnDevice` field.)

Persisted as a `KeepOnDevice` setting on files and an inherited default on folders/projects ([04 §4.2](04-local-data-model.md)).

## 16.3 On-demand fetch & optional convenience cache

- Files in a **`dontKeep`** subtree are fetched and decrypted **on open**, then discarded when closed.
- An optional **convenience cache** (default **on**, user-toggleable, with an optional size cap) retains *recently opened* not-kept files and fetched version snapshots so reopening is instant. Evicting an item only means it's re-downloaded next time (the server always holds the full encrypted copy). It is **never** used for kept files (those are guaranteed) and never holds `excluded` content.
- On eviction of a not-kept file's plaintext, drop its FTS **body** but keep its **title** so it stays discoverable; reopening re-indexes it ([11 §11.3](11-search.md)).
- A **Storage settings** screen shows usage (kept corpus vs. convenience cache vs. index), lets the user toggle/limit or clear the convenience cache, surfaces keep-on-device toggles at file/folder/project level, and estimates the disk cost of "keep everything" so the full-corpus default is an informed choice.

## 16.4 Network constraints

- `BlobPrefetchService` (downloading the corpus + kept subtrees) honors a configurable network budget: pause on **metered connections** where the OS reports them, optional "sync large blobs on unmetered only," and a manual "sync now."
- Delta sync of **structure** runs on focus/connectivity-regained and periodically ([08 §8.7](08-sync-engine.md)); content download honors the keep selection and budget.
- Awareness/relay traffic is throttled ([09 §9.7](09-realtime-collaboration.md)); collaboration sockets stay up while documents are open and reconnect across OS sleep.

## 16.5 Large files & memory

- Stream encryption/decryption and BLAKE3 hashing for large ink/binary blobs to bound memory ([06 §6.6](06-cryptography.md)); avoid loading whole multi-MB blobs into a single buffer.
- Enforce the 100 MB inline ciphertext limit client-side ([05 §5.6](05-api-client.md)); chunked upload for larger is a Phase-5 item.

## 16.6 First-run & low-storage behavior

- On first run / new device, sync structure + names quickly (small) so the whole tree is browsable immediately; then begin **full-corpus prefetch** in the background per the budget, with visible progress.
- Under low device storage, pause not-kept prefetch and the convenience cache, warn the user, and prioritize the open file and explicitly-kept content. The full-corpus default is **adaptive**: if disk is constrained, prompt the user to opt subtrees out rather than silently failing — and never silently drop a subtree the user explicitly kept.

## 16.7 Plaintext export / external-editor interop (desktop-only)

This capability is intentionally a **desktop feature, deferred from Android** ([Android 16 §16.7](https://github.com/Nyxite/NyxiteAndroid)). It lets a power user take an encrypted note **out** of the boundary deliberately — to edit in an external app, attach elsewhere, or archive — with the security trade-off made explicit and consensual.

- **Export a working copy**: `ExportWorkingCopy` decrypts the current head and writes plaintext (`.md`/`.txt`, or the rendered/exported form) to a **user-chosen path**. A clear, non-dismissable warning states that the exported file is **outside Nyxite's encryption** and is the user's responsibility. The action is audited locally and never automatic.
- **Open in external editor (round-trip)** *(opt-in)*: the app may write the plaintext to a **temporary working file**, launch the OS-default (or user-chosen) editor, **watch the file for changes**, and on save **re-import** the bytes — applying them as a CRDT edit (text) or a new blob head (ink/binary). On close it **securely deletes** the temp file (best-effort overwrite + unlink; the spec is honest that secure deletion on modern filesystems/SSDs is not guaranteed). This is off by default and gated behind the same warnings.
- **Boundaries**: exported/temp plaintext is **never** placed in the synced store, the FTS index, or any backup the app controls; it is not tracked as Nyxite content. Guests cannot export. The feature respects the app-lock state (no export while locked).
- **Why desktop-only**: desktops have a real filesystem, external-editor ecosystems, and user expectation of local file workflows; phones do not, and the mobile spec keeps content strictly inside the boundary. The two clients agree that this seam exists **only** here.

## 16.8 Reader-group attachment (cascade cross-reference)

The **reader-group attachment** that drives enterprise "manager reads all" group sharing ([13 §13.7](13-sharing.md), [07 §7.10](07-key-and-device-management.md)) is a **per-project/folder/file cascade** — `inherit` (default) / a specific group / none — resolved exactly like the two policy cascades this client already uses: **keep-on-device** ([§16.2](#162-keep-on-device-the-core-control)) and desktop **plaintext-at-rest** ([17](17-security.md)). A file's effective attachment = its own explicit setting if present, else the nearest ancestor's, else none; new files inherit their parent's effective setting.

- **What differs**: unlike keep-on-device and plaintext-at-rest, which are **client-local and never sent to the server**, the reader-group attachment is a **shared structure setting** — the server stores it (as an **opaque** field) so every author's client resolves the same auto-wrap group on create. It carries no key and no plaintext group name.
- **Effect on storage/sync**: none beyond the wrap itself — auto-wrapping a new file's DEK to the attached group's public key on create ([13 §13.7](13-sharing.md)) adds one opaque DEK-to-group blob; the ciphertext, content addressing, and keep/plaintext behavior of the file are unchanged.
