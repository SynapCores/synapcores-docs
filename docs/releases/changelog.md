# Full changelog

Every CE release ever cut, newest first. Use this if you're on an older version and seeing an issue — scan the entries between your version and the current latest to see whether your bug is already fixed.

Format per entry: short headline → bullet list of user-visible fixes / features / known issues → link to the GitHub release page (binaries + SHA-256s).

For richer per-release pages (architecture deep-dives, validation summary, install tabs), see the [Releases overview](index.md). New per-release pages are added going forward; pre-v1.8.5 lives entirely in this changelog.

---

## v1.8 series

### [v1.8.5-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.5-ce) — 2026-06-15
**Agent-memory primitives.** Full notes: [v1.8.5-ce page](v1.8.5-ce.md).

- New SQL: `MEMORY_STORE(ns, content)`, `MEMORY_RECALL(ns, query, top_k)` (table-valued), `MEMORY_FORGET(ns, id)`. Per-tenant scoped; per-namespace auto-create of `_memory_<ns>` backing table; embeds via the configured embedding model (default `all-minilm:latest`, dim 384).
- Replaces ~150 LOC of hand-rolled `CREATE TABLE + EMBED + HNSW + COSINE_SIMILARITY` plumbing in every memory-having plugin.
- Ships with `@synapcores/sdk@0.5.0` (Node) + `synapcores 0.5.0` (Python) + `@synapcores/openclaw-memory@0.4.0` (OpenClaw plugin migrated to `client.memory.*`).
- Live-validated on the i5-10400F canary (non-AVX-512, no GPU). 17/19 Python SDK live integration tests pass.

### [v1.8.4-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.4-ce) — 2026-06-15
**Mac `chat_with_tools` SIGSEGV fix.** Carries all v1.8.3 security fixes.

- Migrated `grammar_lazy` to the non-deprecated `patterns` API (`llama_sampler_init_grammar_lazy_patterns`) — deep-copies trigger-word strings on the C side, eliminating the CString-lifetime use-after-free that crashed `AGENT_RUN` on macOS.
- Symptom on prior versions: Mac `.ips` crash with faulting address fragment of `</tool_call>`. v1.8.2's `into_raw()` leak workaround was not sufficient; this is the proper fix.
- Public Rust API unchanged.

### [v1.8.3-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.3-ce) — 2026-06-14
**4 security P0s + WS agentic-mode default flip.** Upgrade if you exposed the gateway to anything beyond `127.0.0.1`.

- **#339 (P0):** Hardcoded JWT fallback secret removed from the binary. Per-install random secret persisted at `<data_dir>/.jwt-secret` mode 0600. Tokens signed with the old fallback string now return HTTP 401.
- **#341 (P0):** `/admin/*` routes now require `super_admin` role and cross-check URL tenant vs. JWT claim.
- **#342 (P0):** `TenantContext` cross-checks JWT `company_id` claim against the stored user record. Forged claims no longer pick a tenant.
- **#343 (P0):** `tenant_id` UUID-only canonicalization rejects `..` path-traversal inputs.
- **WS parity:** `AIDB_CHAT_AGENTIC_MODE` default flipped to `false` (was `true` on WS, `false` on REST — inconsistent).

### [v1.8.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.2-ce) — 2026-06-12
**Mac chat segfault hotfix + small-model AGENT_RUN viability.**

- First attempt at the Mac `chat_with_tools` segfault — `CString` `into_raw()` leak workaround in `grammar_lazy`. Superseded by v1.8.4's proper `patterns` API migration; if you're on v1.8.2 and still seeing Mac crashes, jump straight to v1.8.4 or later.
- Tool-description compression in the agent prompt: smaller GGUFs (phi-3, qwen2.5:3b) now fit `AGENT_RUN` reliably.
- Documented the persona-classifier env var for operators who want deterministic persona routing.

### [v1.8.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.1-ce) — 2026-06-11
**Critical fixes** for the v1.8.0 cold-path.

- **CRITICAL panic:** AI chat route used `Lazy::force().unwrap()` on a missing default model. Every AI route 500'd until a model was pulled. Fixed: graceful path that prompts for a model pull instead.
- **HIGH:** UI menu still showed v1.5. Bumped version display.
- **HIGH:** Login screen allowed credential creation in single-tenant CE mode (should be sign-in only).
- **MED:** Image upload didn't generate vision text out-of-box — provider default improved + clearer error UX.
- **registry vs install path:** unified model-pull source so install + registry CLI both go through the same path.

### [v1.8.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.8.0-ce) — 2026-06-08
**Native inference baked into the engine — no Ollama sidecar.** Full announcement: [blog post](https://synapcores.com/blog/v1-8-release).

- `synapcores pull <model:tag>` — OCI Distribution v2 protocol against `registry.ollama.ai`. Multi-layer manifests, sha256-verified blobs, resume on interrupt, idempotent re-pulls. Content-addressed model store at `${data_dir}/models/blobs/<sha256>`.
- New SQL: `PULL_MODEL('name:tag')`, `LIST_MODELS()` table-valued, `DELETE_MODEL('name:tag')`. All listed in the `sql_manual` MCP tool.
- **Beats apples-to-apples CPU Ollama:** 20-recipe agent cert in 39:43 (20/20 PASS) vs. Ollama CPU 65:08 (0/20 PASS, timed out at internal 2-min limit). i5-10400F, no AVX-512, no GPU offload.
- Model-authoritative tool calling: GGUF chat templates rendered in Rust via `minijinja`. Every GGUF with a tool-aware chat template just works (Qwen Hermes, Llama 3 `<|python_tag|>`, Mistral `[TOOL_CALLS]`, generic JSON).
- Persistent `LlamaContext` sessions per chat persona, KV cache reuse via prefix matching, LRU eviction at a configurable resident-memory cap. Measured prefix-reuse rate in the cert: **91%**.
- Streaming chunk channel via `mpsc`. Per-call tracing visible at `RUST_LOG=info`.
- **PK-equality cross-collection bleed fix (#315)** rolled forward from `v1.7.0.2.1-ce`: `WHERE pk = N` no longer returns spurious `[None]` rows when N is ±1 from an existing integer PK.

---

## v1.7 series

### [v1.7.0.2.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.7.0.2.1-ce) — 2026-06-01
**OLTP perf hotfix** on top of v1.7.0.2. Upgrade if you saw a 24-31% point-select TPS drop after v1.7.0.2.

- Removed a redundant `tokio` task-local scope wrap around the SQL execution future on `/v1/query/execute`. The path already received `database_context` as an explicit parameter; the task-local was overhead with no behavior.
- `#225` per-request database context (multi-DB in one CE session) preserved.
- OLTP at or above the v1.7.0.1 baseline at every thread count.

### [v1.7.0.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.7.0.2-ce) — 2026-06-01
**Bug-fix bundle** on top of v1.7.0.1. Recipe cert 182/193 PASS.

- **#225:** Per-request database context via tokio task-local — multiple concurrent CE sessions can hit different databases without racing the process-global current-db. (Tuned again in v1.7.0.2.1 — see above.)
- **#181:** OOB `EMBED()` restored — works without `[query.ai_service]` configured (regression introduced in v1.6.6.7).
- **#205:** Chat-agent split provider/embedding URLs (in-DB agentic OOB without Ollama bridge gymnastics).
- **#180:** `storage_bridge` entry logs + RocksDB reclaim on `DROP TABLE`.
- **#239/#240/#241:** Dockerfile fixed for Debian bookworm + Rust 1.88 + cmake.
- Bundle of ~15 SQL engine bug fixes (#206–#218, #222–#234). 10 new anomaly-detection recipes shipped.
- Known deferred to v1.7.0.3: #224 alias-qualified columns, #232 AutoML inline-PREDICT planner rewrite, #220 RAG Document-not-found fallback (folded into later tags).

### [v1.7.0.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.7.0.1-ce) — 2026-05-29
**Provider + storage + HTTP row-cap fix bundle.**

- **#174 (P0):** OpenAI API key was being logged in plaintext in INFO logs. Redacted.
- **#175:** SQL `EMBED()` was routing to the completion model when `provider=openai` split-mode was configured. Now uses the embedding model.
- **#176:** Gateway hardcoded a 900s timeout on `/query`, ignoring `[server].request_timeout`. Now honors config.
- **#177:** `AGENT_RUN` with OpenAI provider returned raw JSON tool-call as text instead of executing the tool. Fixed.
- **#178:** Silent 1000-row truncation on `PK BETWEEN` when `AIDB_EMIT_INDEXRANGESCAN` was OFF. Default flipped to ON.
- **#169:** RocksDB disk + memory reclaim on `DROP TABLE` — was leaking ~30 MB on-disk / ~55 MB resident per dropped 100K-row table.

### [v1.7.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.7.0-ce) — 2026-05-28
**Agentic SQL with `AGENT_RUN` (major).** Headline feature. Supersedes the planned v1.6.6.10-ce — same code, larger version number to make the upgrade signal unmistakable.

- `SELECT AGENT_RUN(persona, task)` — first-class SQL function running a complete ReAct loop (reason → tool → observe → repeat) and returning TEXT.
- Agent has access to the same per-tenant DB as the calling session: `execute_query`, `list_tables`, `describe_table`, `rag_search` — all in one transaction.
- **Security hardening:** 12 generic system-level tools removed from the agent's registry (`shell`, `file`, `code`, `http`, `db`, `email`, `slack`, `calendar`, `browser`, `web_search`, `web_scraper`, `computer_vision`). Tool surface 17 → 5. They cannot be re-enabled via `agents.toml`.
- Default agent model: `phi3` → `qwen2.5-coder:7b` (in-code default).
- **Result-cache contract changed:** per-write invalidation now guaranteed. Old v1.6.6.x workarounds (`AIDB_ENABLE_RESULT_CACHE=false`, `AIDB_RESULT_CACHE_TTL_SECONDS=1`) are no longer needed — remove them when upgrading.

---

## v1.6.6 series

### [v1.6.6.9-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.9-ce) — 2026-05-27
**Same code as v1.7.0** — v1.7.0 was retagged from v1.6.6.9 with a major bump to signal the AGENT_RUN feature add. If you're on v1.6.6.9, upgrade to v1.7.0+ for any later fix.

- **#165 canary catch:** `[query.ai_service].base_url` was not threaded through `GatewayCompletionProvider` — silent fallback to `localhost:11434` when operators configured Ollama via `gateway.toml`. Fixed.
- 10 new `AGENT_RUN`-native recipes shipped to the homepage `/recipes/agents` repository (insurance claims triage, FinServ compliance, healthcare clinical Q&A, retail returns triage, SaaS support tier-1, sales briefing, devops incident triage, HR employee Q&A, legal contract review, logistics shipment exception).

### [v1.6.6.8-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.8-ce) — 2026-05-26
**6-branch bundle.** Docker model-cache EACCES + warm-on-boot + path-aware AI timeout + AutoML PREDICT clarity + ml-worker module fix + Ollama error msg.

- Recipe cert: 98/148 (+2 over v1.6.6.7).
- macOS included.
- ml-worker `sqlx` offline cache + RocksDB `Send` are pre-existing bugs deferred to v1.6.6.9.

### [v1.6.6.7-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.7-ce) — 2026-05-25
**Docker model-cache EACCES + recipe-import endpoint + frontmatter tolerance.**

- Fixed `EACCES` on the Docker model cache (write path was inside an immutable layer).
- `POST /v1/recipes/import` endpoint added (was referenced by frontend + homepage but not registered — silent no-op).
- Recipe-cert harness added to the release path (DEEP_SCAN).
- macOS held off the tag-push (deferred-mac rule).

### [v1.6.6.6-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.6-ce) — 2026-05-22
**`$N` positional parameter binding on `/v1/query/execute`** — server-side, injection-safe. Was passing raw strings through; now parameterizes via the parser.

### [v1.6.6.5-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.5-ce) — 2026-05-22
**Path-aware request timeout.** The global 30s `TimeoutLayer` was 408'ing legitimate large imports — a 13K-row LeadDelta CSV exceeded 30s and aborted mid-flight. `/import` and `/automl` paths now get a 900s budget; everything else keeps the configured default.

### [v1.6.6.4-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.4-ce) — 2026-05-22
**AutoML sled model-store lock handling.** Splits the error cases:

- `ENOLCK` ("filesystem can't lock" — e.g. Docker Desktop / WSL2 volume) → clear error, no longer quarantines-as-corrupt + recreates (which looped and destroyed models on restart).
- Transient `EAGAIN` → clear stale lock files + brief retry.

### [v1.6.6.3-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.3-ce) — 2026-05-21
**AVX2 baseline for x86_64 ggml — no AVX-512.** Hard guarantee.

- v1.6.6.2's `GGML_NATIVE=OFF` fix was correct but landed in the wrong CMake block. v1.6.6.3 explicitly pins `GGML_AVX512=OFF` (+VBMI/VNNI/BF16), `GGML_AVX2/FMA/F16C=ON`, `-march=x86-64-v3` for the desktop x86_64-Linux block.
- Verified on i5-10400F (AVX2 only): clean rebuild has zero AVX-512 instructions per `objdump`, 2895 AVX2; model eager-loads with no SIGILL.
- **This is now the permanent CE non-AVX-512 baseline.** Every release after this is gated on an i5-10400F canary boot.

### [v1.6.6.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.2-ce) — 2026-05-21
**OOB Docker / install fixes + portable x86_64 ggml (SIGILL fix).** First fix for the AVX-512 SIGILL on non-AVX-512 CPUs. Was incomplete — see v1.6.6.3.

### [v1.6.6.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6.1-ce) — 2026-05-21
**AutoML predict + self-updating models + OpenClaw first-boot key.**

- **AutoML predict now discriminates** by feature values. Was returning a constant `0.85` in v1.6.6. Cause: numeric temp-table typing + Float64 scaler cast + input-order preservation + model resolution. State-asserting AutoML section added to `feature_validator.py` so this class of bug can't ship again.
- `AUTOML.TRAIN`: async self-updating models (`SQL + AFTER INSERT trigger`) with single-flight coalescing and retrain-cooldown debounce to prevent storms.
- First-boot mints a FullAccess OpenClaw memory API key + macOS launchd service docs.

### [v1.6.6-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.6-ce) — 2026-05-20
**The SQL/Cypher parity push.** Three planned releases collapsed into one. `feature_validator` 79/0/0 on a bundled-UI build.

- **Unified Cypher:** runs on `/v1/query/execute` AND the MCP `execute` tool (was `/v1/graph/match` only). SQL `UNION` no longer misrouted into the Cypher planner.
- **SQL parity:** `INFORMATION_SCHEMA.{tables, columns, key_column_usage}`. `WITH RECURSIVE` (recursive CTEs with declared column lists). `CREATE VIEW` + `CREATE MATERIALIZED VIEW` + `DROP VIEW`. (Already in the line: `INSERT … RETURNING`, `FULL OUTER JOIN`, window frames, non-recursive CTEs, `DISTINCT ON`.)
- **Triggers — full feature set:** BEFORE/AFTER × INSERT/UPDATE/DELETE, FOR EACH ROW, WHEN conditions, multi-event (OR), recursion depth cap, persistence across restart, in-place `NEW.col` mutation in BEFORE triggers.
- **Stored procedures — true procedural language:** IF/ELSIF/CASE/WHILE/LOOP/REPEAT with LEAVE/ITERATE, DECLARE + SET variables, OUT/INOUT params, RETURN, RAISE + exception HANDLER.
- Companion: SDK Bug D (`graph.edges.delete`) fixed in `@synapcores/nodejs-sdk@0.4.1`.

---

## v1.6.5 series

### [v1.6.5.6-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5.6-ce) — 2026-05-19
**24x7 stability hotfixes.**

- SQL-executor panics caught at the entry point: any panic during query execution is logged server-side (with payload + SQL excerpt + incident id) and returned to the client as generic 500. Tokio runtime worker survives.
- `substr()` / `SUBSTRING()` no longer byte-indexed — uses character iteration. Em-dashes, smart quotes, emoji, CJK no longer trigger a `byte index N is not a char boundary` panic.
- Global panic hook at gateway boot (defense in depth).
- **EMBED() correctness fix:** `[query.ai_service].embedding_model` in `gateway.toml` is now actually used by `EMBED()`. Prior behavior silently fell back to the chat model's hidden state (e.g. 3584-dim Qwen2.5-Coder) — produced vectors that did not separate natural-language meaning. **If you ran EMBED() on v1.6.5.5 or earlier with split-mode Ollama, your embeddings are wrong; re-embed after upgrading.**
- 404 on unmatched `/v1/*` routes no longer blames the CE — says `Route not found: <path>` and points at `/v1/api-docs` + `/v1/openapi.json`.

### [v1.6.5.4-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5.4-ce) — 2026-05-19
**UI cleanup + carries v1.6.5.3 hot-fixes.** Removed the per-key "View Statistics" icon from API Key Manager that 404'd on a never-implemented endpoint and surfaced a misleading "Enterprise-only" error.

### [v1.6.5.3-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5.3-ce) — 2026-05-19
**Install-day blockers + PDF preview.**

- RocksDB per-tenant lock race fixed (the install-day blocker that surfaced in fresh-Mac smoke testing of v1.6.5.2).
- `install-ce.sh` drops the MCP bridge at `~/.synapcores/`.
- New `GET /v1/mcp/bridge` endpoint serves the bridge offline.
- PDF preview in image gallery: `<iframe blob:>` → `<object blob:>` bypasses the Chrome PDF-plugin blob-URL restriction.

### [v1.6.5.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5.2-ce) — 2026-05-18
**OLTP overhaul + gateway wire-fixes.** Massive perf jumps.

- **Insert throughput 42 TPS → 18,300 TPS (430×)** at 16 threads. Root cause: per-insert `reqwest::Client::new()` was consuming ~96% of CPU on libcrypto X.509 root-CA bundle parsing. Fix: license-aware HTTP client pool (CE singleton, EE per-tenant).
- **Read-only range queries 1.9 TPS → 443 TPS (230×)** at 16 threads. Root cause: `IndexRangeScan` optimizer rule wasn't recognizing Sort/Aggregate/Filter-wrapped plans. Fix: recursive plan walker.
- Gateway wire-format: explicit 400 with remediation hint replaces silent-field-drop bugs for `MatchRequest.params`, named graphs, `AutoML config.inline_rows`.

### [v1.6.5.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5.1-ce) — 2026-05-17
**MCP+14, AUTOML.PREDICT planner fixes, SortExec fix, recipe-builder.** Manual + prompt + tier + UX gate.

### [v1.6.5-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.5-ce) — 2026-05-16
**AutoML out-of-the-box.**

- Default Auto-mode AutoML returns calibrated, polarity-correct probabilities end-to-end on realistic data with no operator hand-tuning.
- 100K rows train in 11s on CPU; predictions span [0.0001, 0.9989] with 444/500 distinct values on a Gold cohort.
- MCP-in-gateway, Gemini provider, Cypher fixes. 20 commits across AutoML perf + correctness, infrastructure, docs.

---

## v1.6.0–v1.6.4

### [v1.6.4-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.4-ce) — 2026-05-12
**Vector / RAG path correctness + install portability.** Closes 9 vector-path bugs.

- **VECTOR columns no longer read back as NULL** — four sibling JSON-to-Value converters all coerce numeric JSON arrays to `Value::Vector`.
- **`CREATE TABLE … VECTOR(N)` preserves the declared dimension for every N** — the Custom-type token parser previously fell back to 384 for any shape other than the strict three-token form.
- INSERT and UPDATE accept literal vector arrays; bulk `UPDATE` with `EMBED(col)` evaluates per-row and persists.
- Dimension validation at write time catches embedding-provider swaps before they silently degrade `COSINE_SIMILARITY` to zero.
- `COSINE_SIMILARITY` returns a real float in SELECT projections; `IS NULL`/`IS NOT NULL` agree across WHERE and projection contexts.
- `SELECT … WHERE pk = lit` no longer errors with `Index '<t>_pkey' not found` when the PK index isn't materialized — falls back to TableScan.
- `provider = "native"` config actually registers the bundled MiniLM as an embedding provider (was silently skipped).
- **`data_dir` sweep:** every active storage path routed through a published `data_dir` module — gateway is now drop-in portable on any install layout. Schema cache files, schemas RocksDB, columnar storage, partition catalog, statistics DB, procedure/trigger registry, users repo, AutoML store, ai_chat schema lookup all consume it.

### [v1.6.3.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.3.1-ce) — 2026-05-11
**Docker/config hotfix** on top of v1.6.3.

### [v1.6.3-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.3-ce) — 2026-05-11
**Cypher dispatch on `/v1/query/execute` dual-parser path** + `/v1/graph/match/profile` fixes + `/v2/graph/graphs` + `SHOW PROPERTY GRAPHS` dispatch + `COSINE_SIMILARITY` null-in-projections + gateway defensive 404 JSON for `/multimedia/*` without `/v1/` + `CALL db.labels()` / `db.relationshipTypes()` syntactic sugar.

### [v1.6.2.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.2.2-ce) — 2026-05-11
**Chat agent fix #2.**

### [v1.6.2.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.2.1-ce) — 2026-05-11
**Chat agent stop-token hotfix** — agent responses were running into delimiter tokens.

### [v1.6.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.2-ce) — 2026-05-10
**Frontend fixes + Graph Schema panel.**

- **Image gallery preview silently blank:** `getMultimediaUrl` emitted `/multimedia/...` (no `/v1/` prefix), which the gateway's SPA fallback returned as `200 OK + index.html`. `PreviewModal` wrapped the HTML in a Blob; the `<img>` rendered nothing. Both preview AND download URL builders fixed.
- **Cypher block with scalar projection rendered "no graph data":** the Cypher branch now runs the same SQL→graph transformer the SQL branch uses. Graph-traversal Cypher renders as cytoscape graph; aggregation/projection Cypher renders as table.
- **New:** Graph Schema panel on the Graph Explorer page. Auto-fetches distinct labels + relationship types, renders them as clickable pills.
- **Build:** `linux-x86_64-cuda` removed from the release matrix until the CUDA inference path is stable end-to-end.

### [v1.6.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.1-ce) — 2026-05-08
**Hardens against the v1.6.0 silent breakage.** Every bug here passed the v1.6.0 recipe smoke harness because the harness only checked HTTP 200, not data state.

- **DEFAULT clauses dropped on INSERT** (still fixed; gated here): `CREATE TABLE … created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP` then `INSERT` stored NULL. Root cause: `transactional_executors.rs` + `indexed_transactional_executors.rs` skipped default-fill; only legacy `executor.rs` called it.
- **`WHERE col = literal` returns 0 rows** (still fixed; gated here): top-level equality on non-PK columns was misrouted by a Wave-8 `IndexScan` planner rule to a non-existent index. `NOT(…)`, `<>`, `IN`, `LIKE`, `AND id > 0 AND name='Alice'` all worked.
- **CUDA tarball aborts at boot** (still fixed; gated here): `ggml-cuda`'s `CUDA_CHECK` calls `abort()` (real SIGABRT) on driver/toolkit mismatch. Fix: child-process canary; on probe failure the gateway boots and serves SQL with only native LLM endpoints disabled.
- **CSV import silently no-ops:** `POST /v1/data/import/csv` returned `{success: true, rows_imported: 0}` then `SELECT` errored "table does not exist". Tenant-context mismatch between import handler (no tenant scope) and SELECT path (JWT-scoped). Fixed.
- **Image gallery preview blocked by CSP:** `default()` + `development()` CSP presets were missing `blob:` in `img-src`/`media-src`/`worker-src` and Google Fonts hosts in `style-src`/`font-src`. The `strict()` preset already had them — wrong preset selected at runtime.
- **Cypher MERGE returns nodes only, no relationships:** executor populated `columns/rows` only with named identifiers (nodes); auto-created relationships, while stored, were absent from the response. Graph Explorer couldn't draw edges it didn't have.
- **SQL editor renders nothing for Cypher MERGE responses:** frontend now detects node-shaped cells in a SQL response and transforms them to `{nodes, edges}`.
- **New release gate:** `scripts/qa/feature_validator.py` — comprehensive state-asserting test runner. Replaces HTTP-200 smoke as the release ship gate. **No release tag until 0 failures.**

### [v1.6.0.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.0.1-ce) — 2026-05-08
**Three P0 engine hotfixes.** Same three bugs gated into v1.6.1 — see above. Both v1.6.0.1 and v1.6.0 are superseded; upgrade to v1.6.1+ directly.

### [v1.6.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.6.0-ce) — 2026-05-08
**Recipe gallery 95.9% pass + license-aware CE.** Shipped with three P0 regressions (DEFAULT clause silently dropped, WHERE col = literal returns 0 rows, CUDA tarball aborts at boot). **Do not deploy v1.6.0 — upgrade to v1.6.1+ immediately.** v1.6.0 was the trigger for adding the state-asserting `feature_validator.py` gate.

---

## v1.5 series

### [v1.5.3-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.5.3-ce) — 2026-05-07
**License acceptance + SDK 0.2.0 merge.**

- `LICENSE.txt` embedded in the binary (`include_str!`). `synapcores --license` prints full text and exits.
- First-run click-through prompts on TTY; non-interactive (Docker, systemd) requires `--accept-license` or `AIDB_ACCEPT_LICENSE=1`.
- `<data_dir>/license-accepted.json` mode 0600 with sha256 anchor; re-prompts only when license text changes.
- `@synapcores/sdk@0.2.0` (Node, scoped public) + `synapcores 0.2.0` (Python, first PyPI publish) source in tree.

### [v1.5.2-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.5.2-ce) — 2026-05-07
**Cypher variable-length relationship binding fix.** Single fix: var-length relationship variable now binds to a `List<Edge>` instead of a single `Edge`, so Cypher quantifiers (`ALL`/`ANY`/`NONE`/`SINGLE`) and list functions (`relationships`, `reduce`) work over var-length patterns. `fraud_ring_detection` 5/5 (was failing block 2).

### [v1.5.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.5.1-ce) — 2026-05-06
**Hotfix.**

- `execute_saved_recipe` stub returned fake success without running anything.
- CSP `img-src`/`media-src` missing `blob:` blocked upload preview thumbnails.
- Multimedia GET responses missing `Cross-Origin-Resource-Policy` blocked gallery render under `COEP: require-corp`.
- `execute_cypher_for_tenant` only accepted `MATCH`; `MERGE`/`UNWIND`/`UNION` recipes failed.
- Multi-clause `MERGE` in one block lost variable scope (parser + executor + splitter chain).
- Default `--log-level` changed from `info` to `error`. 17 misclassified `tracing::error!` calls downgraded to `debug!`.
- `tracing-subscriber` switched to `EnvFilter` so `RUST_LOG` works for per-target overrides.

### [v1.5.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.5.0-ce) — 2026-05-06
**Perf defaults + multimodal vision + UI capability disclosure.**

- **Performance defaults flipped:** `AIDB_ENABLE_RESULT_CACHE=true` (was `false`), `AIDB_OPTIMIZER_VERSION="unified"` (was `"legacy"`). 102× `oltp_point_select` gain available zero-config.
- **Multimodal vision (NEW):** `CompletionProvider::complete_multimodal()` with OpenAI / Anthropic / Ollama adapters. `ImageProcessor::vision_analyze` dispatched by `AIDB_VISION_PROVIDER` env var. `/v1/system/vision` admin routes for config; API key encrypted at rest with AES-256-GCM (HKDF-SHA256 from `AIDB_JWT_SECRET`).
- **Licensing:** Tenant-isolation license-locked in CE binaries via `#[cfg(not(feature = "multi-tenant"))]`. Config-load forces `enabled=false` with a tracing warn — EE feature can't leak into a CE install regardless of TOML contents.
- **Routing:** V2 API routes mounted by default (decoupled from `tenant_isolation.enabled`). V2 gains `/multimodal` + `/vector-algebra` parity backports. Removed duplicate `/query/prepare` handler that caused gateway panic on every startup since perf-wave-8 P2.
- **Frontend:** `PlatformCapabilitiesBanner` on Dashboard + Filesystem Collections pages — 7 capability rows showing what's ready vs. what needs setup. Settings → Vision Provider tab.

---

## v1.3 series

### [v1.3.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.3.1-ce) — 2026-04-30
**Debian 12 install fix.**

- `install-ce.sh` was mapping `ubuntu:22.04` and `debian:12` to the same FFmpeg 4 tarball variant. They are different ABIs: Ubuntu 22.04 → FFmpeg 4.4 (`libavutil.so.56`), Debian 12 → FFmpeg 5.1 (`libavutil.so.57`), Ubuntu 24.04 → FFmpeg 6.x (`libavutil.so.58`). CE has builds for first + third only.
- `check_distro` now splits these. Debian 12 gets a clear "use Docker" message with the exact `docker run` command and exits early.
- Bootstrap's edition-check (`synapcores --version | grep Community`) was running BEFORE deps were installed → always emitted "binary did not report 'Community' on --version" on a fresh box. Removed; check moved post-deps with an `ldd` hint.

### [v1.3.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.3.0-ce) — 2026-04-30
**Intel Mac dropped from release matrix.** `macos-13` runner allocation on personal-account repos was unreliable (3+ hour waits). `darwin-aarch64` (Apple Silicon) is ~80%+ of the 2026 Mac install base; Intel Mac users have a working Docker fallback documented at [Run on macOS](../macos.md). Bootstrap's `detect_platform()` now exits early on `darwin/x86_64` with a clear Docker hint instead of 404'ing on a missing tarball.

---

## v1.2 series

### [v1.2.1-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.2.1-ce) — 2026-04-29
**Docker tag-vs-binary version grep dropped.** The Dockerfile's pre-ship verification was comparing `SYNAPCORES_VERSION` (release tag, e.g. `v1.2.1-ce`) against the binary's `--version` output (Cargo workspace version, e.g. `0.1.0`). These are independent by design — every release tag was failing the docker build with `binary version mismatch: expected v1.2.1-ce` even though the binary was correct. The `Community` edition grep is kept.

### [v1.2.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.2.0-ce) — 2026-04-29
**FileSystemRAG GA.** See the [Filesystem Collections](../filesystem-rag.md) page.

---

## v1.0

### [v1.0.0-ce](https://github.com/SynapCores/synapcores-releases/releases/tag/v1.0.0-ce) — 2026-04-28
**Community Edition GA release.** First public CE tag.

---

## How to upgrade

For any version older than two releases back, the safest path is:

1. Stop the running engine.
2. Back up `<data_dir>` (the RocksDB volume).
3. Run the [installer one-liner](../quickstart.md) — it always grabs the current `:latest`.
4. Restart with your existing config. The new binary will read the old `data_dir` in place; no migration step exists between any two CE tags.

If you're on **v1.6.0** specifically, jump straight to v1.6.1+ — three P0 regressions shipped in v1.6.0 (DEFAULT dropped, equality returns 0 rows, CUDA SIGABRT).

If you're on **anything before v1.6.6.3** and running on a non-AVX-512 x86_64 CPU (i5-10400F, older Xeons), upgrade — v1.6.6.3 is the first release with the AVX2-baseline guarantee (no SIGILL on boot).

If you upgraded from **v1.6.5.5 or earlier with split-mode Ollama embedding configured**, re-embed your vector collections after upgrading to v1.6.5.6+ — earlier versions silently fell back to the chat model's hidden state for `EMBED()`.
