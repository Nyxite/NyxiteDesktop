# 09 — Real-time Collaboration

Live multi-user editing of text documents over an **encrypted relay**: the server stores and forwards encrypted CRDT updates but never merges or reads them; **the desktop runs the Yrs engine (ydotnet) and merges locally** ([server 05](https://github.com/Nyxite/NyxiteServer)). The desktop is also the platform's **reference snapshotter** ([§9.6](#96-snapshotting--compaction-client-driven)). The ydotnet binding is the highest-risk client dependency and must be spiked first ([19](19-open-questions.md)).

## 9.1 Components

- **`CrdtEngine`** (`Nyxite.Desktop.Crdt`, over `Nyxite.Crdt`/ydotnet): holds the per-document Yrs doc, applies/encodes updates, computes state vectors, serializes snapshots. This is the **same CRDT glue** that backs the server-side `Nyxite.CrdtConformanceTests` harness.
- **`RelayClient`** (`Nyxite.Desktop.Network`, SignalR .NET client): connects to the `RelayHub`, joins per-document rooms, submits/receives **encrypted** updates and awareness, manages reconnection. Speaks ciphertext only.
- **`CollabRepository`** (`Nyxite.Desktop.Data.Collab`): orchestrates join → bootstrap → live loop → snapshot → leave, bridging `CryptoEngine` ⇄ `CrdtEngine` ⇄ `RelayClient`.

## 9.2 Transport

- **SignalR over WebSocket**, single hub `RelayHub`, groups keyed `file:{fileId}`. Use the official **`Microsoft.AspNetCore.SignalR.Client`** wrapped behind `IRelayClient` — a first-class, fully-async .NET client (no RxJava-style bridge needed, unlike Android). Configure the **MessagePack** or JSON hub protocol to match the server's hub.
- **Upgrade auth**: the server's access token (bearer) for users; a short-lived **share token** for guests ([14](14-authentication.md)). The token authorizes *relay access*; the **decryption key never comes from the server** (it is the user's unwrapped FK, or the guest's fragment key).
- **Fallback**: when the socket is unavailable, use the REST encrypted-update endpoints ([08 §8.4](08-sync-engine.md)).

## 9.3 Hub contract (client side)

Mirror the server `IRelayHub`. All payloads are ciphertext.

Server → client callbacks the client handles:
- `OnUpdates(fileId, EncryptedUpdate[])` — everything after the client's cursor (join bootstrap).
- `OnUpdate(fileId, EncryptedUpdate)` — a peer's relayed update.
- `OnAwareness(fileId, byte[] awarenessCipher)` — encrypted ephemeral presence/cursors.
- `OnPresence(fileId, PresenceDto[])` — join/leave roster (identities only).
- `OnError(fileId, code, message)`.

Client → server hub methods:
- `JoinDocument(fileId, sinceSeq)`
- `SubmitUpdate(fileId, EncryptedUpdate)`
- `SubmitAwareness(fileId, awarenessCipher)`
- `LeaveDocument(fileId)`

`EncryptedUpdate { seq, ciphertext, keyId }` (from `Nyxite.Contracts`).

## 9.4 Join handshake

1. Open the (authenticated) socket; `JoinDocument(fileId, localSinceSeq)`.
2. Server ACL-checks (read to receive, write to submit) and returns `OnUpdates` with all encrypted updates after the cursor, plus a pointer to the latest encrypted snapshot.
3. Client bootstraps the Yrs doc: decrypt+load snapshot, decrypt+apply updates, reconcile via the **locally reconstructed state vector** (the server computes no diff).
4. Client is added to the room; `OnPresence` delivers the roster.

## 9.5 Update flow

- **Local edit** → Yrs produces an update → `CryptoEngine.Seal(update, fk, kind=Crdt, fileId)` → `SubmitUpdate`. The server assigns `seq`, persists ciphertext, broadcasts `OnUpdate` to the room.
- **Inbound** `OnUpdate` → `Open` → apply to the local Yrs doc → recompute editor text → marshal new UI state to the Avalonia UI thread. CRDT merge is deterministic and order-independent.
- **Outbox**: locally generated updates are persisted (`CrdtUpdateEntity`, [04](04-local-data-model.md)) until acked, so edits survive disconnects and are submitted on reconnect or via REST fallback.
- An update that fails to decrypt within a single FK generation is a **hard error** surfaced to the user (should not happen); on `keyrotate` the client refetches keys ([07 §7.6](07-key-and-device-management.md)).

## 9.6 Snapshotting & compaction (client-driven)

The server can't compact an encrypted log. **The client periodically snapshots**: serialize the merged Yrs doc, `Seal` it with the FK, upload as a content-addressed blob + create a `file_versions` row ([12](12-version-history.md)). Triggers: **≥ 200 updates since the last snapshot**, **a 5-minute time threshold**, or the **last participant leaving** the room. Run in `SnapshotService`; any participant may snapshot; the server may then prune updates older than the snapshot's `seq`, retaining the safety tail it defines (last 7 days OR the most recent 1000 updates, whichever is larger — [server 05 §5.6](https://github.com/Nyxite/NyxiteServer)).

> **Reference snapshotter.** Because the desktop holds the full corpus and ample CPU, it snapshots reliably and is the natural fallback when only mobile/web peers (more resource-constrained) are present. Where a desktop is in the room it should generally take the snapshot, reducing load on lighter clients — a soft preference, not a protocol requirement (any participant may snapshot, so correctness never depends on a desktop being present).

## 9.7 Awareness & presence

- **Awareness** (cursor, selection, display label/color) is **encrypted** with the FK and relayed via `SubmitAwareness`/`OnAwareness`; **never persisted**. Rendered as remote carets/selections in the editor ([10](10-editors.md)).
- **Presence roster** (`OnPresence`) shows who is in the room by account display name (from the user's account) or "guest"; no content. Rendered as avatar chips.
- Throttle awareness emission (e.g. on cursor move, coalesced ~20–30 Hz max via an Rx `Throttle`/`Sample`) to limit traffic.

## 9.8 Guest mode (this app joining a link share)

- Entry: the app opens a `https://nyxite.app/share/{token}#k=<fragmentKey>` link (handled via the registered `nyxite://` scheme / web link handler and single-instance forwarding, [15 §15.1](15-ui-and-navigation.md)). The **fragment key is parsed locally and never sent** to the server.
- `GET /share/{token}` bootstraps; the relay socket connects at `/share/{token}/ws` with a short-lived share session token authorizing relay access only.
- The FK comes from the fragment; the client decrypts/edits exactly as a member would.
- Read-only links: the client receives `OnUpdate`/`OnAwareness` but is rejected on `SubmitUpdate` — the editor enters view-only mode. Guest-authored updates store `author_id = null` server-side.
- A guest has no Nyxite account; the app runs in a constrained "guest session" without the full key/device subsystem.
- **Guest storage model**: a guest runs **without the per-account SQLCipher DB**. Guest content is held in an **ephemeral in-memory cache only** and is **never written to any account database or on-disk store**; decrypted views are transient and discarded when the guest session ends ([14 §14.5](14-authentication.md)).

## 9.9 Reconnection & lifecycle

- Exponential backoff reconnect (the SignalR client's `WithAutomaticReconnect` plus an app-level supervisor); on reconnect, re-`JoinDocument` with the current `sinceSeq` and drain the outbox.
- On closing the document/editor, `LeaveDocument` and tear down; on OS **sleep**, the socket drops and reconnects on **wake**.
- Connection state surfaced in the editor (live / reconnecting / offline-editing).

## 9.10 Wire-protocol conformance (critical)

Text edited on desktop must merge identically on web (Yjs) and Android (yrs/UniFFI). The client ships a **`CrdtConformance` test suite** that replays the shared Yrs wire-protocol vectors and asserts identical merged state and identical encoded updates across bindings ([18 §18.5](18-build-ci-testing.md)) — the same vectors the server's `Nyxite.CrdtConformanceTests` uses. Because desktop and server share `Nyxite.Crdt`, a desktop conformance pass is also the server-side guarantee. If ydotnet fails interop or is too immature, fall back to a thin own binding over `yffi` — decide during the Phase-1 spike ([19](19-open-questions.md)).
