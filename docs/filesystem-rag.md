# Filesystem Collections

!!! info "v1.2.0-ce preview"
    This feature is in the **v1.2.0-ce** release. If you're on an earlier
    version, upgrade with:

    ```bash
    curl -fsSL https://get.synapcores.com/install.sh | sh
    ```

    Screenshots of the Web UI surface are coming with the v1.2.0 release.

A **Filesystem Collection** is a folder on disk that the gateway watches.
Drop a file in, and SynapCores ingests it: CSVs become SQL tables, PDFs
and text files get chunked and embedded for RAG, images get OCR'd and
CLIP-embedded, and audio/video gets transcribed via the embedded
inference engine. Everything is queryable from SQL the moment it lands.

It's a thin operator-friendly path to the same primitives that already
power the Web UI's media gallery and document collections — except the
upload step is "save the file."

## Use cases

- **Knowledge bases** — point a collection at the team's research folder;
  every PDF, Markdown note, and meeting transcript becomes
  vector-searchable from SQL or AI Chat.
- **Document search over a shared drive** — sync an SMB or NFS mount
  into a watched path; ingest is automatic as the drive updates.
- **Log ingestion** — point a collection at a directory of CSV log
  exports; each file becomes a SQL table you can query and join
  immediately.
- **Image gallery from a synced folder** — drop screenshots or product
  photos in; CLIP embeddings make them visually searchable, OCR
  captures any embedded text.
- **Podcast or meeting archive** — drop `.mp3` or `.mp4` files in;
  Whisper transcribes the audio track, the transcript is embedded, and
  the original is linked from the Web UI media gallery.

## How files get processed

The processor dispatches on file extension. Each extension has a single
deterministic path; you don't configure it per-file.

| Extension | What happens |
|---|---|
| `.csv` | Schema-inferred from the header row, becomes a SQL table you can query directly. |
| `.pdf`, `.txt`, `.md` | Text extracted, chunked (default 512 tokens), embedded; queryable via `SELECT ... WHERE COSINE_SIMILARITY(...)`. |
| `.jpg`, `.png`, `.webp` | OCR run for any text content (Tesseract), CLIP embedding for visual search, indexed in the gallery. |
| `.mp3`, `.wav`, `.m4a` | Whisper transcription, transcript chunked and embedded. |
| `.mp4`, `.mov`, `.webm` | Frames sampled at fixed intervals (CLIP-embedded), audio track Whisper-transcribed; both go into the same collection index. |
| anything else | Metadata-only entry — filename + path + size embedded so you can still find it by name. |

Files outside the collection's `extensions` allowlist are ignored
entirely — no metadata entry, no log line beyond a debug-level skip.

## Setting up a watched folder

### Prerequisites

- The folder must exist and be readable by the `synapcores` system user
  (the user the gateway runs as).
- The path must be canonical — symlinks pointing outside the path are
  rejected. This is a hard security guard, not a config option.
- For first ingest with embeddings, the gateway needs outbound network
  access to Hugging Face (one-time model download — see
  [First-run AI setup](ai-setup.md)).

### Option A — From the Web UI

1. Log in (see [Web UI](web-ui.md)).
2. Open **Filesystem Collections** in the left nav.
3. Click **Create**, fill in:
    - **Name** — your handle for the collection (e.g. `team_kb`)
    - **Path** — server-side absolute path (e.g. `/data/incoming`)
    - **Extensions** — multi-select from the supported list
    - **Recursive** — watch subdirectories too
    - **Chunk size** — for text files, tokens per chunk (default 512)
    - **Embed model** — `minilm` (default), `bert-base`, or `bert-large`
4. Click **Create**. The collection enters `active` state immediately
   and a startup scan kicks off.

> Screenshots coming with v1.2.0 release.

### Option B — From SQL

```sql
CREATE COLLECTION my_docs
  FROM FILESYSTEM '/data/incoming'
  WITH (
      extensions   = 'pdf,csv,jpg,mp4,txt',
      recursive    = true,
      auto_index   = true,
      chunk_size   = 512,
      embed_model  = 'minilm'
  );
```

`CREATE COLLECTION ... FROM FILESYSTEM` is a SynapCores extension to
SQL. It's parsed through the same dual-path parser routing as
`CREATE IMMUTABLE TABLE`, so it works identically from the CLI, the
REST API, and the Web UI's query editor.

### What you'll see during the first ingest

In the gateway logs:

```
filesystem_collections: created collection "my_docs" id=fc_01HAB...
filesystem_collections: scanning "/data/incoming" (recursive=true, 47 candidate files)
filesystem_collections: processed report-q3.pdf (12 chunks embedded, 0.84s)
filesystem_collections: processed sales.csv (1,203 rows → table sales, 0.31s)
filesystem_collections: processed launch.mp4 (transcribed 8m12s, 14 chunks, 22.4s)
filesystem_collections: collection "my_docs" caught up — 47 files, 0 failed
```

After that, the watcher takes over. Drop another file in and you'll
see a single processed line within a second or two of the OS event.

## Querying the results

### Text RAG (PDFs, .txt, .md)

```sql
SELECT name, content_chunk
FROM   my_docs
WHERE  COSINE_SIMILARITY(embedding, EMBED('quarterly revenue trends')) > 0.7
ORDER  BY similarity DESC
LIMIT  5;
```

### CSV-as-table

When a CSV is ingested, the collection registers a SQL table with the
file's basename. Query it like any other table:

```sql
SELECT region, SUM(revenue) AS total
FROM   sales
GROUP  BY region
ORDER  BY total DESC;
```

If you want the full join syntax across collections, the table is
addressable as `my_docs.sales` too.

### Image search

```sql
SELECT name, path
FROM   my_docs
WHERE  file_type = 'image'
  AND  COSINE_SIMILARITY(embedding, EMBED('whiteboard photo with handwritten diagrams')) > 0.6
LIMIT  10;
```

OCR text is in the `ocr_text` column if you'd rather grep it directly:

```sql
SELECT name, ocr_text
FROM   my_docs
WHERE  file_type = 'image'
  AND  ocr_text ILIKE '%invoice%';
```

### Audio / video transcripts

```sql
SELECT name, content_chunk, start_time, end_time
FROM   my_docs
WHERE  file_type IN ('audio', 'video')
  AND  COSINE_SIMILARITY(embedding, EMBED('roadmap commitments for Q4')) > 0.7
ORDER  BY similarity DESC
LIMIT  10;
```

`start_time` / `end_time` are in seconds within the source media —
useful for deep-linking back into the original.

### Cross-collection queries (within a tenant)

A tenant can query across all of its collections in a single statement:

```sql
SELECT collection_name, name, content_chunk
FROM   filesystem_collections.documents
WHERE  COSINE_SIMILARITY(embedding, EMBED('open security incidents')) > 0.7
ORDER  BY similarity DESC
LIMIT  20;
```

## Configuration reference

### `CREATE COLLECTION` options

| Option | Type | Default | Description |
|---|---|---|---|
| `extensions` | comma-separated string | `'pdf,csv,txt,md,jpg,png,mp3,mp4'` | Allowlist of file extensions to ingest. |
| `recursive` | bool | `true` | Watch subdirectories of the root path. |
| `auto_index` | bool | `true` | Build the vector index as files arrive. Set `false` to ingest now and build the index later via `REINDEX COLLECTION`. |
| `chunk_size` | int | `512` | Tokens per chunk for text files. |
| `embed_model` | string | `'minilm'` | Embedding model: `minilm` (384-d), `bert-base` (768-d), `bert-large` (1024-d). |

### REST endpoints

All endpoints require `Authorization: Bearer <JWT>` and are
tenant-scoped automatically.

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/filesystem-collections` | Create a new collection. |
| `GET` | `/v1/filesystem-collections` | List collections for the authenticated tenant. |
| `GET` | `/v1/filesystem-collections/:id` | Detail view with recent ingest events. |
| `PATCH` | `/v1/filesystem-collections/:id` | Pause / resume — body `{"paused": true}` or `{"paused": false}`. |
| `DELETE` | `/v1/filesystem-collections/:id` | Delete. Optional query string `?retain_data=true` keeps the ingested rows / vectors. |
| `GET` | `/v1/filesystem-collections/:id/documents` | List ingested files with status, size, and last-processed timestamp. |

Example — create from curl:

```bash
TOKEN="<JWT from /v1/auth/login>"
curl -X POST http://localhost:8080/v1/filesystem-collections \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "my_docs",
           "path": "/data/incoming",
           "extensions": ["pdf", "csv", "txt"],
           "recursive": true,
           "auto_index": true,
           "chunk_size": 512,
           "embed_model": "minilm"
         }'
```

### WebSocket progress

Live progress events for a single collection:

```
GET /ws/filesystem-collections/:id/progress
```

The server emits one JSON message per ingest lifecycle event
(`scan_started`, `file_processing`, `file_done`, `file_failed`,
`scan_complete`, `paused`, `resumed`). The Web UI uses this stream to
power its real-time event log.

## Tenant isolation

Filesystem Collections are tenant-scoped end to end:

- Collection metadata is keyed by tenant in RocksDB.
- The vector index is created per-collection-per-tenant, with no
  shared name space.
- Two tenants can legitimately watch overlapping paths on the host.
  Each gets its own ingestion, its own index, its own results — they
  don't see each other's collections, documents, or queries.
- The gateway optionally enforces a per-tenant allowed-roots list in
  `gateway.toml`; if set, collection paths must be under one of those
  roots.

## Limits & quotas

- **Max file size**: 100 MB per file by default. Files above the cap
  are skipped with a logged warning. Override per-collection in v1.3.
- **Max collections per tenant**: subject to the standard CE caps —
  see [Limits & quotas](limits.md).
- **Embedding throughput**: bounded by the embedding model's
  per-second rate. The pipeline backpressures gracefully; under
  sustained overload the queue policy is drop-oldest-pending with a
  warning.
- **Vectors per collection**: same 50,000,000 ceiling as any other
  vector collection in CE.

## Troubleshooting

### "I dropped a file but it doesn't appear"

Walk the checklist:

1. **Tail the gateway log** during the drop:

    ```bash
    sudo journalctl -u synapcores -f | grep filesystem_collections
    ```

    No log line at all? The watcher didn't see the OS event.

2. **Confirm the extension is in the collection's allowlist.** A `.docx`
   file will not be ingested if the collection only watches `pdf,csv,txt`.

3. **Check file permissions.** The `synapcores` user must be able to
   read the file:

    ```bash
    sudo -u synapcores cat /data/incoming/report.pdf > /dev/null
    ```

    Permission denied? Fix the owner or group on the file.

4. **Confirm the watched folder still exists.** If the directory was
   deleted and recreated under the same path, the kernel inode changed
   and the old watch may be dead. Pause and resume the collection to
   re-arm it.

### "Embedding model download is slow / hangs"

The first call to `EMBED()` triggers a Hugging Face download into
`~/.cache/huggingface/`. On a fresh install, expect ~90 MB for the
default MiniLM model. Subsequent calls are cached and run offline.

If the download fails, the gateway logs a clear error and the file
stays in `pending` state — no data loss, just retry once connectivity
is back. To force a retry without waiting for the next OS event,
pause and resume the collection.

### "Watched folder must exist when collection is created"

Yes — by design. Creating a collection over a path that doesn't exist
returns a `400 Bad Request`. If the folder gets deleted out from under
a live collection, the next scan logs a clear error and the collection
moves to `failed` state until the path is restored.

## What's not supported in v1.2.0-ce

- **Cloud storage watching** (S3, GCS, Azure Blob) — Enterprise feature.
- **Cross-tenant collections** — collections are strictly tenant-scoped;
  there is no "shared" mode in CE.
- **OCR for handwritten content** — the embedded Tesseract pipeline
  handles printed text only.
- **Real-time collaborative editing of ingested files** — ingest is
  read-only. Edit the source file at the OS layer and the watcher will
  re-ingest it; SynapCores doesn't write back to the watched directory.

## Where to next

- [Web UI](web-ui.md) — full UI tour, including the media gallery and
  vector search surfaces that Filesystem Collections feed into.
- [First-run AI setup](ai-setup.md) — wire up an LLM provider so the
  ingested content is reachable from AI Chat and NL2SQL.
- [Limits & quotas](limits.md) — the per-tenant caps that apply.
