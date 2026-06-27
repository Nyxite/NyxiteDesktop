# 11 — Search (Client-Side, Full-Corpus)

The server holds **no index** (it can't read content). Search lives where the keys and plaintext live — on the device ([server 11](https://github.com/Nyxite/server)). The **desktop is the platform's full-corpus search surface**: with room to hold the whole library it indexes **everything the account can read**, and is the reference "search everything" experience that web and Android (subset-only) defer to.

## 11.1 Scope on Desktop

| Included in the index | Excluded |
|---|---|
| **The full corpus** — every `server-default` file (the desktop default is keep-on-device, [16 §16.2](16-offline-and-storage-policies.md)) | `excluded` files on *other* devices (never reach this one) |
| `excluded` files created/held on **this** device | Files the user explicitly marked `dontKeep` and has never opened (title indexed; body fetched on open) |
| Decrypted titles of all files known in the structure | Content that genuinely cannot be decrypted (missing key) |

The default is **complete coverage**: the `BlobPrefetchService` proactively pulls, decrypts, and indexes the corpus ([08 §8.7](08-sync-engine.md)). The only gaps are subtrees the user has deliberately opted out of keeping; for those, **titles are still indexed** and the UI offers "open to search inside." Search UX states the scope honestly when any subtree is opted out ("N files are not kept on this device — open them or mark them keep-on-device to include their contents").

## 11.2 Index

- **SQLite FTS5** virtual table `FileContentFts(fileId UNINDEXED, title, body)` inside the SQLCipher DB ([04 §4.3](04-local-data-model.md)). The index is encrypted at rest with the rest of the DB and **never uploaded**.
- Tokenizer (DECISION): **FTS5 `unicode61`** (diacritic folding) **with `porter` stemming for English**; recorded and kept consistent across rebuilds. Support prefix queries.
- **Indexed content**: markdown/plaintext body (the decrypted current head), and titles for every file. Ink files index title only in v1.0.0 (no recognition yet; reserve a hook for recognized-text later, [10 §10.5](10-editors.md)).
- The full-corpus index can be large; FTS5 with `contentless`/external-content keeps it compact. The index lives in the per-account DB and is subject to the same encryption and storage accounting ([16 §16.3](16-offline-and-storage-policies.md)).

## 11.3 Indexing lifecycle

- On decrypt: whenever a file's plaintext head is produced (open, sync pull, snapshot, CRDT merge to a quiescent point), `Data.Search` upserts its FTS row.
- On keep-on-device / prefetch: `BlobPrefetchService` decrypts kept content → index it ([08 §8.7](08-sync-engine.md), [16 §16.2](16-offline-and-storage-policies.md)). Because the desktop default is full-corpus, this is the steady state — most files are indexed proactively.
- On opt-out / eviction: when a `dontKeep` file's plaintext is evicted, drop its body from the index but keep its title (so it still appears as a result that must be fetched to read).
- On delete/rotate: remove or refresh rows accordingly.
- `ReindexService` can rebuild the index (schema change, corruption) from locally available plaintext; on a full-corpus desktop this rebuild covers nearly everything.

## 11.4 Query & UX

- A global search box (always reachable; `Ctrl+K`/`Ctrl+F`-style) and a dedicated search view; debounce input (Rx `Throttle`); run FTS off the UI thread.
- Results: title + snippet (`snippet()`/`highlight()` from FTS5) + project/folder breadcrumb + sync/cache badge. Opening a result opens the file (fetching+decrypting on demand if only the title was indexed).
- Filters: by project/folder, by content type, by "available offline" (kept/cached). Sort by relevance or recency.
- Because coverage is normally complete, distinguish the rare "title match only — open to search inside" case clearly from the common "found in content" case.

## 11.5 Privacy

- The index contains decrypted content; it is protected exactly like the DB (SQLCipher + OS keystore + optional app lock, [17](17-security.md)).
- No query text or results ever leave the device. There is no server round-trip for search.
