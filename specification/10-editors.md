# 10 — Editors

Three content forms, each with a distinct **view mode** and **edit mode**: markdown, plain text, and handwritten ink.

## 10.1 Shared editor scaffold

- A `Feature.EditorText` / `Feature.EditorInk` screen is driven by an `EditorViewModel` exposing editor state (mode, decrypted content, sync/collab state, peers/awareness) and commands (`EnterEditMode`, `ApplyEdit`, `ToggleView`, `Snapshot`, `KeepOnDevice`, `Share`, `ExportWorkingCopy`, `Back`). Editor/collab ViewModels use **ReactiveUI** for the stream-shaped state ([01 §1.3](01-architecture.md)).
- Opening a file: `OpenFileForEdit` resolves the FK ([07](07-key-and-device-management.md)), bootstraps content (snapshot + CRDT log for text; latest blob for ink), and (for text) joins the relay room ([09](09-realtime-collaboration.md)).
- Saving is **continuous** for CRDT text (every edit is an update); for ink it is an explicit/auto `PUT` of the encrypted page ([08 §8.5](08-sync-engine.md)).
- View ↔ edit toggle is instant and preserves scroll/caret. View mode is the default for shared/received files.
- Desktop supports **multiple open documents** (tabs and/or multiple windows, [15 §15.1](15-ui-and-navigation.md)); each open text doc holds its own relay room and Yrs doc.

## 10.2 Markdown editor

- **Edit mode**: an **AvaloniaEdit** text surface bound to the CRDT-backed document. Local keystrokes apply to the Yrs doc → encrypted update → relay; inbound updates re-render. Provide a formatting toolbar and keyboard shortcuts (bold/italic/heading/list/link/code/checkbox) that insert markdown syntax, and optional live inline styling.
- **View mode**: rendered markdown via **Markdig** → Avalonia controls — headings, lists, tables, code blocks, task lists, links, images (images only when the referenced blob is available/decrypted). **No remote network fetch from markdown content** (privacy): only embedded/attached encrypted blobs render.
- **Collaboration overlays**: remote carets/selections from awareness ([09 §9.7](09-realtime-collaboration.md)); presence chips in the title bar.
- **Large documents**: virtualize rendering; debounce the markdown parse in view mode.

## 10.3 Plain-text editor

Same CRDT backbone and collaboration as markdown, without rendering, over AvaloniaEdit. Monospace optional; used now for `plaintext` and later for `sourcecode` (Phase 5), where AvaloniaEdit's **syntax highlighting** is enabled in view/edit mode.

## 10.4 Ink editor (pen/stylus)

Goal: **low-latency, pressure-aware** handwriting that round-trips faithfully with the Android client and stores in the shared encrypted vector format.

### Capture
- A **custom Avalonia control** drawing through the **Skia** backend. Read pointer data from Avalonia pointer events: position, `PointerPointProperties.Pressure`, `XTilt`/`YTilt`, `Twist` where the OS/driver exposes them (Windows Ink, Linux libinput, Wacom drivers). Use `GetIntermediatePoints()` to capture sub-frame samples between event ticks.
- Distinguish pen vs mouse vs touch (`PointerType`) to support palm rejection and finger/mouse pan-zoom while the pen draws.
- Support eraser (pen eraser button / inverted stylus where reported), undo/redo, pen/highlighter/eraser tools, color and width, and pressure→width mapping.

> **Honest scope vs. Android**: Android is the **capture leader** (S-Pen pressure/tilt, low-latency front buffer). Desktop ink quality depends on the OS/driver; pressure is captured where available and degrades to constant-width with mouse. The two clients share the **storage format**, so an ink note drawn on either opens on the other.

### Page model
- A file is a sequence of **pages**; each page holds **strokes**. A stroke is a polyline of timestamped sample points `(x, y, pressure, tilt, orientation, t)` plus tool/brush attributes (color, base width, blend).
- Editing operations: add stroke, erase stroke(s), move/transform selection, insert/delete page. v1.0.0 ink sync is **LWW per page/file** (not CRDT) — concurrent edits resolve by last-write-wins with the loser retained as a sibling version in history. A per-file **version-vector** (`{ deviceId -> counter }`, incremented on each local committed ink edit) rides in encrypted `metadata_enc` and classifies edits as equal/ancestor/concurrent ([08 §8.5](08-sync-engine.md)).

### Rendering
- Render committed strokes from the page model; render the in-progress stroke with minimal latency, then commit. Smooth/segment strokes (Catmull-Rom / the ink model's smoothing) for natural curves. Honor pressure in width/opacity.

## 10.5 Ink storage format (shared, versioned, encryptable)

- The **Nyxite ink vector format** is a compact, versioned, **deterministic CBOR** serialization of pages → strokes → samples + brush metadata, with the exact field schema **`[P]` co-designed with Android** ([Android 10 §10.5](https://github.com/Nyxite/android), [19 §19.4](19-open-questions.md)). It must be **byte-stable** so the BLAKE3 content address is reproducible, and **shared with Android** so an ink file drawn on one opens on the other ([master `docs/SPECIFICATION.md`](https://github.com/Nyxite/Nyxite)).
- The serialized page/file is **sealed with the FK** ([06 §6.3](06-cryptography.md)) and stored as an encrypted blob; the version-vector for LWW rides in encrypted `metadata_enc` ([04](04-local-data-model.md)).
- The format carries a `version` for forward evolution (new brush types, recognition data later). The format is specified in its own shared document; this client must conform exactly. **Desktop and Android co-own the format spec** ([19 §19.4](19-open-questions.md)).
- **Out of scope v1.0.0**: handwriting recognition / searchable ink text, Samsung `.sdoc` import (separate migration item) — but reserve a place in the format for recognized-text layers so ink can later feed the search index ([11](11-search.md)).

## 10.6 Performance & input quality

- Keep the ink render path off UI-thread-blocking work; commit strokes incrementally.
- Batch sample persistence; do not seal/encrypt per sample — seal per committed stroke-set / page save.
- Provide palm rejection and zoom/pan that don't interfere with the pen; honor high-refresh displays where present.

## 10.7 Accessibility & cross-cutting

- Text editors: scalable fonts, screen-reader labels (Avalonia automation peers), selection/clipboard, find-in-document (local), full external-keyboard support and shortcuts.
- Ink: provide alternatives where possible; ensure controls are reachable without a stylus (mouse/keyboard).
- All editors honor the **secure-window / app-lock** settings ([17](17-security.md)) and the **export-working-copy** action's explicit warnings ([16 §16.7](16-offline-and-storage-policies.md)).
