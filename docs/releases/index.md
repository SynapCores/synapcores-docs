# Releases

This page tracks SynapCores Community Edition releases. The latest release is always installed by the [one-liner installer](../quickstart.md):

```bash
curl -fsSL https://get.synapcores.com | sh
```

To pin a specific version:

```bash
curl -fsSL https://get.synapcores.com | SYNAPCORES_VERSION=v1.10.0-ce sh
```

Or via Docker:

```bash
docker pull synapcores/community:v1.10.0-ce
```

Release artifacts (binaries + SHA-256 sidecars + the install script) live on the [SynapCores/synapcores-releases](https://github.com/SynapCores/synapcores-releases/releases) GitHub repository.

## Release history

| Version | Date | Headline |
|---|---|---|
| [**v2.4.0-ce**](v2.4.0-ce.md) | 2026-10-27 | `COPY ... TO LAKE` — materialize any `SELECT`/`WITH` result as plain, Hive-partitioned foreign Parquet in your object store in one statement (streamed, no REST cap; `overwrite_partitions`/`append`; `zstd`/`snappy`/`gzip`/`lz4`), gated by a fresh full-write principal + explicit destination `write_allowlist`, with durable publication recovery and `CREATE EXTERNAL TABLE` round-trip. Also: fresh installs answer database questions on the chat model by default (empty `database_ask` allowlist), native model eviction/swap (`keep_alive=0`), Ollama-compat `/api/version` pre-auth + `/api/tags` for scoped keys, `SUM(INT)`→`BIGINT`, and Learning Loop console fixes. Seamless upgrade from v2.3.x |
| [**v2.3.3-ce**](v2.3.3-ce.md) | 2026-10-26 | Calendar date math you can trust — `INTERVAL` `YEAR`/`MONTH` do true leap-aware, month-end-clamped arithmetic (previously every interval collapsed to an approximate integer number of days, and a `YEAR`/`MONTH` suffix was ignored), fixed-duration units keep sub-day precision, and `CAST(... AS DATE)` is strictly validated; `SQRT`/`POWER` propagate `NULL` and accept exact `DECIMAL`, and `FALSE AND` / `TRUE OR` short-circuit; reads over third-party Parquet reconcile nullability and union additive columns to typed `NULL`s; plus two engine-stability fixes (empty-body CTE schema, deep join/aggregate/CTE worker-stack) caught by validating the release artifact. 177/187 recipe certification with zero regressions vs v2.3.2; seamless upgrade from v2.3.x |
| [**v2.3.2-ce**](v2.3.2-ce.md) | 2026-10-24 | Agent RBAC — database-scoped agent privileges (`AGENT MONITOR`/`OPERATE`/`APPROVE`/`CONFIGURE`), roles and role-bound expiring API keys, database ownership and a per-decision audit receipt — paired with a 47-finding correctness pass and the full RBAC console (Learning Loop, roles & grants). Public debut of the DAVINCI line on `:latest` |
| [**v2.3.1-ce**](v2.3.1-ce.md) | 2026-10-22 | DAVINCI — governed autonomous learning in the database: define a goal and a deterministic success check, and agents improve recurring workflows from verified outcomes (off by default, approval-gated writes, per-goal budgets, a durable kill switch that survives restarts); every recorded decision enters an append-only, cryptographically hash-linked ledger the engine verifies end to end, including individual record lineage; a new elastic HNSW vector store pages beyond RAM within a configured memory budget (1.5× budget, 3 tenants, 64 checkpoints, zero OOM; latency within ±6%); native Ollama API compatibility (ollama-python, LangChain, LlamaIndex); scoped database RAG that returns the generated SQL and evidence; agent memory sized to your embedding model (768-dim) with correct supersession; a stronger `qwen2.5:7b` default for RAG and agent tasks; seamless upgrade from v2.2.0-ce. Autonomy stays off until explicitly enabled |
| [**v2.1.3-ce**](v2.1.3-ce.md) | 2026-10-14 | Parallel analytics, on by default — `SELECT DISTINCT`, `GROUP BY` and aggregate scans execute across every core automatically (189k-row `DISTINCT` 44 ms → 3.8 ms single-query, 121 → 851 q/s under concurrency), scaling with physical core count; the GenAI inference scheduler auto-tunes thread allocation, concurrency and queue depth to the detected CPU/memory and absorbs `EMBED`/`GENERATE` bursts with bounded backpressure instead of capacity errors; the Linux binary now ships stripped (334 MB → 254 MB) and crash-looping installs stop at `failed` instead of restarting forever. Point lookups and single-row writes are at v2.1.2 parity — the speedups are analytical only |
| [**v2.1.2-ce**](v2.1.2-ce.md) | 2026-10-12 | Truth verification lands in SQL — `NLI_CLASSIFY`, `NLI_VERIFY`, and `NLI_ANCHORS` gate what agents store or show against trusted evidence in one statement, on the same in-process pinned model as the REST NLI endpoints; verification itself is 26–33× faster (394 ms → 15 ms per pair, 6.2 s → 0.19 s for 16 anchors) with bit-identical scores; the result-cache view check short-circuits when no views exist; macOS installs fail fast with working guidance instead of a mid-install error |
| [**v2.1.1-ce**](v2.1.1-ce.md) | 2026-10-11 | The graph engine grows up — Cypher secondary indexes make `MERGE` up to ~2,300× faster and point lookups sub-millisecond (fastest of SynapCores, Kuzu and Neo4j CE on traversal shapes); `POST /v1/graph/bulk` ingests ~112k entities/s in one call; query timeouts enforced server-side; fixes the v2.1.0 regression that broke the bundled UI and both official SDKs |
| [**v2.0.0-ce**](v2.0.0-ce.md) | 2026-10-08 | Durable transactions — an acknowledged `COMMIT` survives a power cut, with real isolation and `SAVEPOINT`; `FOREIGN KEY` and table-level `UNIQUE` are now enforced rather than accepted and ignored; 19 new machine-learning algorithm paths (32 total) including clustering with no labels and time-series forecasting, plus scheduled and drift-triggered retraining; MySQL → S3 Parquet export on a schedule with read-back verification; a readiness probe that executes SQL instead of only proving the listener is up. Single-row writes are ~2× slower — that is the cost of the durability guarantee, and batching amortises it |
| [**v1.18.0-ce**](v1.18.0-ce.md) | 2026-10-06 | Faster lake analytics — Parquet queries now beat DuckDB on 9 of 12 shapes (median 0.63× its time) through filter pushdown into the reader, filters carried across joins, `IN` pushdown and early `LIMIT` exit; point lookups no longer slow down as unrelated tables grow (~2,000 ms → ~25 ms for a key that is not present); `ROLLBACK` reverses writes and `BACKUP` produces a real checkpoint; non-Latin text handled safely everywhere |
| [**v1.14.5-ce**](v1.14.5-ce.md) | 2026-08-23 | Agent event bindings survived concurrency at last — a user write committing during any agent run silently lost its binding (0/2 in-flight before, 2/2 after); recipes executed against the master store instead of the tenant's; reloading any UI page froze for up to 30 minutes on a busy gateway |
| [**v1.14.4-ce**](v1.14.4-ce.md) | 2026-08-21 | Ten fixes for failures that looked like success — an indexed `AND` returned rows the `WHERE` excluded (no error raised), index definitions did not survive a restart, and the installer wrote a 30s query timeout that made `GENERATE()` fail on CPU-only hosts; plus six interface fixes |
| [**v1.14.3-ce**](v1.14.3-ce.md) | 2026-08-18 | Persistent agent memory — `CREATE MEMORY` makes multi-turn agent memory a database object (identity-scoped, authority-resolved conflicts, engine-side consolidation); plus cloud LLM providers (`anthropic`/`gemini`) now work for SQL `GENERATE()`, not just the REST endpoints |
| [**v1.14.2-ce**](v1.14.2-ce.md) | 2026-08-10 | Agentic reliability — durable agents stateless by default, personas as first-class objects, honest run audit; plus three index-maintenance fixes that returned silently wrong results |
| [**v1.14.1.1-ce**](v1.14.1.1-ce.md) | 2026-08-07 | Critical patch — explicit-projection NULL reads, oversized-prompt gateway abort, and Llama-model agentic abort all fixed |
| [**v1.14.1-ce**](v1.14.1-ce.md) | 2026-08-06 | MongoDB Data Sync — replicate a Mongo collection into a queryable SQL table (`_id`→PK, nested fields→JSON); data sync on by default; graph queries ~20× faster |
| [**v1.14.0-ce**](v1.14.0-ce.md) | 2026-07-29 | Native Vision — `POST /v1/multimodal/describe` interprets an image with a vision LLM entirely in-process (native `libmtmd`/LLaVA); no cloud, no API key, no daemon; CPU-friendly; the image never leaves the box |
| [**v1.11.0-ce**](v1.11.0-ce.md) | 2026-07-18 | MySQL wire protocol — connect Metabase / DBeaver / DataGrip / any MySQL client with just a connection string (OOB admin login); `[mysql_wire]` listener + `SHOW …` / `information_schema` introspection |
| [**v1.10.0-ce**](v1.10.0-ce.md) | 2026-07-16 | MySQL compatibility — `ON DUPLICATE KEY UPDATE`, JSON accessors, `ENUM`, `ON UPDATE CURRENT_TIMESTAMP`, `FULLTEXT`/`MATCH`, type aliases, mysqldump import, and index-backed constraint enforcement |
| [**v1.9.1-ce**](v1.9.1-ce.md) | 2026-07-14 | Tamper-evident agent decision lineage — every `_system_agent_runs` record hash-chained; detect tampering with `WHERE verified = false` |
| [**v1.9.0-ce**](v1.9.0-ce.md) | 2026-07-13 | The Autonomous Database — `CREATE AGENT` durable agents (schedule/event activations, governance, audit) + native cross-encoder reranker + reliability batch |
| [**v1.8.9-ce**](v1.8.9-ce.md) | 2026-07-10 | Agent memory upsert — `MEMORY_UPSERT` (revise/retract beliefs) + `AGENT_RUN` per-call limits + security hardening |
| [**v1.8.7-ce**](v1.8.7-ce.md) | 2026-06-19 | Controllable generation — `GENERATE(prompt, options)` sampling/JSON-output knobs + `json_object()` |
| [**v1.8.6.1-ce**](v1.8.6.1-ce.md) | 2026-06-18 | Docker OOB EMBED + AGENT_RUN restored (Dockerfile-only fix on v1.8.6 engine; carries the `#388` negative-vector-literal SQL fix) |
| [**v1.8.5-ce**](v1.8.5-ce.md) | 2026-06-15 | Agent-memory primitives — `MEMORY_STORE` / `MEMORY_RECALL` / `MEMORY_FORGET` |

For every prior release (v1.0 → v1.8.4) with short user-facing summaries — what was fixed, what to watch out for if you're on that exact tag — see the [**Full changelog**](changelog.md). Per-release deep-dive pages are added going forward starting with v1.8.5.
