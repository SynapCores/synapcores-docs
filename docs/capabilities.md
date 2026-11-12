# Capabilities & how to enable each

SynapCores is one binary that bundles a lot: SQL, vectors, an in-process LLM, decisions,
agents, graph, ML, audit storage, and a MySQL wire. This page is the **switchboard** — for
each capability: what it is, the one step to turn it on, and the one call to use it. Most are
on out of the box; a few need a one-time pull or flag.

First time here? Install the engine with the [Quickstart](quickstart.md), wire a model with
[First-run AI setup](ai-setup.md), then come back and switch on what you need.

!!! info "How 'enable' works"
    Everything AI-shaped reads one config block, `[query.ai_service]` (see
    [AI setup](ai-setup.md)). "Enable" below means one of: *already on*, *pull a model once*
    (`synapcores pull <name>`), or *set one config/env flag and restart*. Authenticate every
    call with a JWT (`POST /v1/auth/login`) or an `aidb_…` API key as
    `Authorization: Bearer …`.

## At a glance

| Capability | Enable | Use it with |
| --- | --- | --- |
| [SQL + transactions](#sql-transactions) | ✅ on | `POST /v1/query/execute` |
| [Vector search](#vector-search) | ✅ on (embedding model) | `EMBED()`, `COSINE_SIMILARITY()`, `VECTOR(N)` |
| [Text generation](#text-generation) | chat model | `GENERATE()` |
| [AI Chat + NL2SQL](#ai-chat-nl2sql) | chat model | `POST /v1/ai/chat` |
| [In-database agents](#in-database-agents) | tool-capable chat model | `AGENT_RUN()`, `CREATE AGENT` |
| [Agent memory](#agent-memory) | embedding model | `MEMORY_STORE` / `MEMORY_RECALL` |
| [One-call decisions (d1)](#decisions-d1) | `pull d1:latest` | `POST /v1/systemone` |
| [Claim verification (NLI)](#claim-verification-nli) | `--with-nli` / `AIDB_NLI_ENABLED=1` | `NLI_VERIFY()`, `/v1/ai/nli/verify` |
| [AutoML / in-database ML](#automl-ml) | ✅ on | `CREATE EXPERIMENT` → `AUTOML.PREDICT()` |
| [Graph / Cypher](#graph-cypher) | ✅ on | `POST /v1/graph/match` |
| [Native vision](#native-vision) | vision-capable model | `POST /v1/multimodal/describe` |
| [Filesystem RAG](#filesystem-rag) | ✅ on | `CREATE COLLECTION` |
| [Immutable audit tables](#immutable-audit-tables) | ✅ on (key for encryption) | `CREATE IMMUTABLE TABLE` |
| [Data lake export](#data-lake-export) | lake destination + grant | `COPY … TO LAKE` |
| [MySQL wire protocol](#mysql-wire-protocol) | `[mysql_wire] enabled = true` | any MySQL client, port 3307 |
| [MCP (LLM clients)](#mcp-for-llm-clients) | bridge config | Claude / Cursor / Continue / … |

---

## SQL + transactions {#sql-transactions}

**What** — a full SQL engine with durable, isolated transactions (`BEGIN` / `COMMIT` /
`SAVEPOINT`), enforced `FOREIGN KEY` / `UNIQUE`, and MySQL-compatible syntax.
**Enable** — on by default.
**Use** — every statement runs through `POST /v1/query/execute` with `{"sql": "..."}`:

```bash
curl -sS -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"sql":"CREATE TABLE t (id INT PRIMARY KEY, name TEXT); INSERT INTO t VALUES (1,'\''a'\'')"}' \
  http://localhost:8080/v1/query/execute
```

See the [SQL reference](sql-reference.md) for the full dialect.

## Vector search {#vector-search}

**What** — semantic search in plain SQL: a `VECTOR(N)` column, `EMBED()` to turn text into a
vector, `COSINE_SIMILARITY()` to rank, and an HNSW index for scale.
**Enable** — on by default; `embedding_model` (default `all-minilm:latest`, **384-dim**) pulls
on first use.
**Use**:

```sql
CREATE TABLE docs (id INT PRIMARY KEY, body TEXT, embedding VECTOR(384));
INSERT INTO docs VALUES (1, 'Refunds take 30 days', EMBED('Refunds take 30 days'));
SELECT id, COSINE_SIMILARITY(embedding, EMBED('how long for a refund?')) AS sim
FROM docs ORDER BY sim DESC LIMIT 5;
-- at scale: CREATE INDEX docs_hnsw ON docs(embedding) USING HNSW;
```

Declare `VECTOR(N)` to match your embedding model's width (384 native, 1536 OpenAI small).

## Text generation {#text-generation}

**What** — call an LLM as a SQL value.
**Enable** — a chat model wired in [AI setup](ai-setup.md) (default `qwen2.5:7b`).
**Use** — `GENERATE('prompt')`, optionally `GENERATE(prompt, json_object('temperature', 0.2))`:

```sql
SELECT GENERATE('Summarize in one sentence: ' || body) AS summary FROM docs WHERE id = 1;
```

## AI Chat + NL2SQL {#ai-chat-nl2sql}

**What** — a grounded chat assistant over your schema, and natural-language → SQL.
**Enable** — a chat model (as above).
**Use** — `POST /v1/ai/chat` (and `POST /v1/ai/chat/stream` for SSE), or the **AI Chat** tab
in the [Web UI](web-ui.md). NL2SQL is also exposed at `POST /v1/nl2sql/query`.

## In-database agents {#in-database-agents}

**What** — a ReAct agent loop callable from SQL (`AGENT_RUN`), and **durable agents**
(`CREATE AGENT`) that fire on a schedule or on committed DML.
**Enable** — a **tool-capable** chat model (`qwen2.5:7b` qualifies). The in-process `local`
provider does native tool calling — no Ollama required.
**Use**:

```sql
SELECT AGENT_RUN('aidb-assistant',
  'Find the 3 tables with the most rows and explain what each is for') AS report;
```

`AGENT_RUN(persona, task [, json_object('model', …, 'max_iterations', …, 'timeout_ms', …)])`.
Durable agents and their tamper-evident run log (`_system_agent_runs`) arrived in v1.9.0+.

## Agent memory {#agent-memory}

**What** — persistent, semantic memory for chatbots and agents with no schema of your own.
**Enable** — an embedding model (as for vector search).
**Use** — `MEMORY_STORE(ns, content)` to write, and `MEMORY_RECALL` (table-valued, goes in
`FROM`) to retrieve:

```sql
SELECT MEMORY_STORE('conv_42', 'The user prefers email over phone');
SELECT content, similarity FROM MEMORY_RECALL('conv_42', 'how should I contact them?', 3);
```

Also `MEMORY_UPSERT` (revise/retract a belief) and `MEMORY_FORGET`.

## One-call decisions (d1) {#decisions-d1}

**What** — a **typed decision** (routing choice, yes/no probability, or urgency score) in one
call with **zero generated tokens**, backed by the purpose-built **d1** model.
**Enable** — pull d1 once:

```bash
synapcores --config /etc/synapcores/gateway.toml pull d1:latest
```

**Use** — `POST /v1/systemone`, naming `"model": "d1:latest"` per request. Full guide:
**[Decision model (d1)](decisions.md)**.

## Claim verification (NLI) {#claim-verification-nli}

**What** — an in-process cross-encoder that verifies a statement against trusted evidence and
returns `TRUTH` / `LIE` / `UNVERIFIED_PLAUSIBLE`. The go-to hallucination gate.
**Enable** — one-time opt-in: `./install-ce.sh --with-nli` (or `AIDB_NLI_ENABLED=1` in Docker),
which pulls `nli-default` (~576 MB) and enables it on the next start.
**Use** — `NLI_VERIFY()` / `NLI_CLASSIFY()` in SQL, or `POST /v1/ai/nli/verify`. Full guide:
**[Claim verification (NLI)](nli.md)**.

## AutoML / in-database ML {#automl-ml}

**What** — train, deploy, and predict with classical ML (classification, regression,
clustering, forecasting) directly in SQL — no external ML stack.
**Enable** — on by default (CE caps: 5 models/tenant, 1M training rows/run — see
[Limits](limits.md)).
**Use**:

```sql
CREATE EXPERIMENT churn_model AS
  SELECT tenure, monthly_charges, churned AS target FROM customers
WITH (task_type = 'classification', target_column = 'target');

SELECT AUTOML.PREDICT('churn_model', tenure, monthly_charges) AS churn_risk
FROM new_customers;
```

## Graph / Cypher {#graph-cypher}

**What** — a property graph co-located with your tables: `MATCH` / `MERGE` / `CALL <algo>`,
indexed, no ETL.
**Enable** — on by default.
**Use** — `POST /v1/graph/match` with the Cypher in a **`sql`** field:

```bash
curl -sS -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"sql":"MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name LIMIT 10"}' \
  http://localhost:8080/v1/graph/match
```

Graph queries need an authenticated tenant context (send a JWT or API key).

## Native vision {#native-vision}

**What** — describe/interpret an image with a vision LLM **entirely in-process** (native
LLaVA) — no cloud, no API key, the image never leaves the box.
**Enable** — a vision-capable model configured on the AI service.
**Use** — `POST /v1/multimodal/describe` with `{"image": "<base64>", "prompt": "..."}`. See the
[v1.14.0-ce notes](releases/v1.14.0-ce.md).

## Filesystem RAG {#filesystem-rag}

**What** — point a folder at the engine and get a watched, chunked, embedded collection; CSVs
also become queryable tables.
**Enable** — on by default.
**Use** — `CREATE COLLECTION` from SQL or the Web UI. Full guide:
**[Filesystem Collections](filesystem-rag.md)**.

## Immutable audit tables {#immutable-audit-tables}

**What** — append-only, hash-chained tables for tamper-evident audit trails; `UPDATE` /
`DELETE` are rejected, and `VERIFY TABLE` proves the chain.
**Enable** — on by default. Optional encryption-at-rest needs a key:
`export AIDB_IMMUTABLE_MASTER_KEY=$(openssl rand -hex 32)` (**hex, ≥ 32 bytes**) — required only
when a table requests `ENCRYPTION`.
**Use**:

```sql
CREATE IMMUTABLE TABLE audit_log (id BIGINT PRIMARY KEY, actor TEXT, action TEXT)
  WITH (ENCRYPTION = 'AES256GCM');   -- omit WITH for an unencrypted audit chain
VERIFY TABLE audit_log;              -- attests the hash chain end to end
```

## Data lake export {#data-lake-export}

**What** — materialize any `SELECT` / `WITH` result as Hive-partitioned Parquet in your object
store, in one statement.
**Enable** — a configured lake destination with a `write_allowlist` grant, run by a
full-data-write principal (see the authorization notes in the
[v2.4.0-ce release page](releases/v2.4.0-ce.md)).
**Use** — `COPY (<query>) TO LAKE '<path>' WITH (CONNECTION = '<dest>', PARTITION_BY = (col))`.

## MySQL wire protocol {#mysql-wire-protocol}

**What** — connect Metabase, DBeaver, DataGrip, or any MySQL client with just a connection
string.
**Enable** — add a `[mysql_wire]` block and restart (off by default):

```toml
[mysql_wire]
enabled = true
bind    = "0.0.0.0"   # default 127.0.0.1
port    = 3307        # default
```

**Use** — `mysql -h <host> -P 3307 -u admin -p`. See the
[v1.11.0-ce notes](releases/v1.11.0-ce.md).

## MCP (for LLM clients) {#mcp-for-llm-clients}

**What** — expose the engine's SQL/tools to Claude, Cursor, Continue, Windsurf, Zed, and
LangChain over the Model Context Protocol, so an LLM can query your database directly.
**Enable** — configure the MCP bridge in your client.
**Use** — full walkthrough: **[MCP setup](mcp-setup.md)**.

---

## Where next

- [First-run AI setup](ai-setup.md) — wire the provider the AI capabilities share.
- [SQL reference](sql-reference.md) — every function and statement, version-matched.
- [Recipes](https://synapcores.com/recipes/) — 160+ runnable end-to-end examples.
- [Community vs Enterprise](enterprise.md) · [Limits & quotas](limits.md)
