# SQL Reference

!!! note "Auto-synced from source"
    This page is **auto-synced** from the SynapCores AIDB engine
    repository on every release tag. The canonical source is
    `AIDB_SQL_MANUAL.md` in the engine repo — **do not edit this
    page directly**; your change will be overwritten on the next release.

    **Last synced from**: `v1.8.7-ce` on 2026-06-19


AIDB is an AI-native SQL database with first-class support for vector embeddings, AutoML, Cypher graph queries, and LLM functions. This manual is the authoritative reference for AIDB SQL features (v1.6.0 through v1.6.5.1). Use ONLY features documented here.

---

## Quick reference — AIDB extensions at a glance

The features below are AIDB-specific extensions that distinguish AIDB SQL from generic SQL. Recipes for AI / analytics intents should prefer these over hand-rolled equivalents.

| Feature                      | Form                                                                  |
|------------------------------|-----------------------------------------------------------------------|
| Vector column type           | `col_name VECTOR(N)` where N is the embedding dimension               |
| Text embedding               | `EMBED(text_expr)` -> VECTOR                                          |
| Cosine similarity            | `COSINE_SIMILARITY(vec_a, vec_b)` -> float in [-1, 1]                 |
| Euclidean distance           | `EUCLIDEAN_DISTANCE(vec_a, vec_b)` -> float >= 0                      |
| Train AutoML model           | `CREATE EXPERIMENT name AS SELECT ... WITH (task_type=..., ...)`      |
| Predict with AutoML model    | `SELECT AUTOML.PREDICT('model', col1, col2, ...) AS risk FROM t`      |
| List/describe models         | `SHOW MODELS`, `DESCRIBE MODEL name`                                  |
| LLM text generation          | `GENERATE(prompt_text [, options_json])` -> TEXT                       |
| JSON object literal builder  | `json_object(key, val, ...)` -> JSON  *(v1.8.7+)*                       |
| Native-inference model pull  | `PULL_MODEL('qwen2.5-coder:7b')` -> TEXT (v1.8.0+)                    |
| Native-inference model list  | `LIST_MODELS()` -> table (v1.8.0+)                                    |
| Native-inference model drop  | `DELETE_MODEL('model_name')` -> TEXT (v1.8.0+)                        |
| Cypher graph pattern         | `MATCH (n:Label) RETURN n` (per-tenant graph)                         |
| Cypher graph write           | `CREATE (n:Label {prop: value})`, `MERGE`, `DETACH DELETE n`          |

---

## Data Definition Language (DDL)

### CREATE DATABASE / DROP DATABASE / USE / SHOW DATABASES

```sql
CREATE DATABASE [IF NOT EXISTS] db_name;
DROP DATABASE [IF EXISTS] db_name [CASCADE];
USE db_name;
SHOW DATABASES [LIKE 'pattern'];
```

### CREATE TABLE

```sql
CREATE TABLE [IF NOT EXISTS] table_name (
    column_name data_type [column_constraint],
    ...
    [table_constraint]
);
```

**Data Types (full list):**

Scalar: `BOOLEAN`, `SMALLINT`, `INTEGER`, `BIGINT`, `REAL`, `DOUBLE`, `DECIMAL(p, s)`,
`TEXT`, `VARCHAR(n)`, `CHAR(n)`, `BYTEA`, `JSON`, `JSONB`, `UUID`,
`TIMESTAMP`, `DATE`, `TIME`.

**AI-native:** `VECTOR(N)` where `N` is the embedding dimension (must match the configured embedding model — default MiniLM is 384).
Multimedia: `AUDIO`, `VIDEO`, `IMAGE`, `PDF`.

**Column constraints:** `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, `CHECK (expr)`, `DEFAULT expr`, `REFERENCES other_table(other_col)`.

**Worked example — table with a vector column for semantic search:**

```sql
CREATE TABLE products (
    id        BIGINT PRIMARY KEY,
    name      TEXT NOT NULL,
    category  TEXT,
    price     DECIMAL(10, 2),
    description     TEXT,
    description_vec VECTOR(384)
);
```

### ALTER TABLE / DROP TABLE / CREATE INDEX / DROP INDEX

```sql
ALTER TABLE t ADD COLUMN c data_type [constraint];
ALTER TABLE t DROP COLUMN c;
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t ALTER COLUMN c TYPE new_type;

DROP TABLE [IF EXISTS] t [CASCADE];

CREATE [UNIQUE] INDEX [IF NOT EXISTS] idx_name ON t (col [ASC|DESC], ...);
DROP   INDEX [IF EXISTS] idx_name;
```

---

## Data Manipulation Language (DML)

```sql
INSERT INTO t [(c1, c2, ...)] VALUES (v1, v2, ...), ...;

UPDATE t SET c1 = v1, c2 = v2 [WHERE condition];

DELETE FROM t [WHERE condition];
```

**Worked example — populate a vector column from text using `EMBED`:**

```sql
UPDATE products SET description_vec = EMBED(description)
 WHERE description_vec IS NULL;
```

---

## Query Language

### SELECT

```sql
SELECT [ALL | DISTINCT] expr [AS alias], ...
  FROM table_name
 [WHERE condition]
 [GROUP BY expr, ...]
 [HAVING condition]
 [ORDER BY expr [ASC | DESC], ...]
 [LIMIT n] [OFFSET k];
```

`ORDER BY` may reference projection aliases directly:

```sql
SELECT id,
       COSINE_SIMILARITY(description_vec, EMBED('wireless headphones')) AS similarity
  FROM products
 ORDER BY similarity DESC
 LIMIT 10;
```

(If the alias is misspelled, the query returns a clear `unknown column` error rather than silently returning empty — fixed in v1.6.5.1.)

### Joins, CTEs, subqueries

```sql
SELECT o.id, o.total, c.name
  FROM orders o
  JOIN customers c ON o.customer_id = c.id
 WHERE o.created_at >= NOW() - INTERVAL '30 days';

WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '30 days'
)
SELECT customer_id, COUNT(*) AS n_orders
  FROM recent_orders
 GROUP BY customer_id;
```

### Transaction Control

```sql
BEGIN [TRANSACTION];
COMMIT;
ROLLBACK;
```

---

## Built-in Functions

### Math
`ABS`, `CEIL`/`CEILING`, `FLOOR`, `ROUND`, `MOD`, `POWER`/`POW`, `SQRT`, `EXP`,
`LOG`/`LN`, `LOG10`, `SIGN`, `TRUNCATE`/`TRUNC`, `PI`, `RAND`/`RANDOM`,
`SIN`, `COS`, `TAN`, `ASIN`, `ACOS`, `ATAN`, `DEGREES`, `RADIANS`.

### String
`UPPER`, `LOWER`, `LENGTH`, `SUBSTRING`, `CONCAT`, `TRIM`, `LTRIM`, `RTRIM`,
`REPLACE`, `LEFT`, `RIGHT`, `LPAD`, `RPAD`, `REPEAT`, `REVERSE`,
`INSTR`/`POSITION`, `ASCII`, `CHAR`/`CHR`, `INITCAP`, `MD5`, `SHA1`, `SHA256`.

### Date / Time
`NOW`/`CURRENT_TIMESTAMP`, `CURRENT_DATE`, `CURRENT_TIME`,
`YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `UNIX_TIMESTAMP`,
`DATE_FORMAT(date, fmt)`, `STR_TO_DATE(s, fmt)`,
`DATE_ADD(date, n, unit)`, `DATE_SUB(date, n, unit)`, `DATEDIFF(d1, d2)`,
`LAST_DAY`, `DAYNAME`, `MONTHNAME`, `QUARTER`, `WEEK`/`WEEKOFYEAR`,
`DAYOFWEEK`, `DAYOFYEAR`.

Format examples:

```sql
DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s')   -- '2026-05-17 14:30:45'
DATE_ADD('2026-01-15', 30, 'DAY')          -- 2026-02-14
DATEDIFF('2026-12-31', NOW())              -- days until year end
```

### Conditional / null
`GREATEST(a, b, ...)`, `LEAST(a, b, ...)`, `IF(cond, then, else)`/`IIF`,
`IFNULL(expr, alt)`/`ISNULL`, `COALESCE(...)`, `NULLIF(a, b)`,
`CASE WHEN ... THEN ... ELSE ... END`.

---

## Vector & AI Functions

AIDB exposes vector and LLM operations as first-class SQL functions. They compose normally with `WHERE`, `ORDER BY`, joins, and CTEs.

### `EMBED(text)`

Computes an embedding for the given text using the configured embedding model.

* Argument: any `TEXT` expression.
* Returns: `VECTOR(N)` matching the configured model dimension (default 384 for MiniLM).
* The column you store the result in must use the matching dimension.

```sql
SELECT EMBED('wireless noise cancelling headphones');

UPDATE products SET description_vec = EMBED(description);
```

### `COSINE_SIMILARITY(vec_a, vec_b)`

Returns a `DOUBLE` in `[-1, 1]`. `1` = identical direction, `0` = orthogonal, `-1` = opposite.

```sql
SELECT id, name,
       COSINE_SIMILARITY(description_vec, EMBED('running shoes')) AS similarity
  FROM products
 ORDER BY similarity DESC
 LIMIT 10;
```

### `EUCLIDEAN_DISTANCE(vec_a, vec_b)`

Returns a `DOUBLE` >= 0. Smaller = more similar.

```sql
SELECT id, EUCLIDEAN_DISTANCE(description_vec, EMBED('hiking boots')) AS dist
  FROM products
 ORDER BY dist ASC
 LIMIT 5;
```

### `GENERATE(prompt [, options])`

Calls the configured completion model and returns the generated text.

* Arguments:
  * `prompt` (`TEXT`, required) — the prompt sent to the model.
  * `options` (`JSON` object, optional, *v1.8.7+*) — sampling + output-shape knobs. Build with `json_object()`. Recognized keys:
    * `max_tokens` (INT, default 4096 in *v1.8.7+*, was 200 prior).
    * `temperature` (FLOAT)
    * `top_p` (FLOAT 0–1)
    * `top_k` (INT)
    * `repeat_penalty` (FLOAT)
    * `seed` (INT) — same seed + same prompt → reproducible output
    * `system` (TEXT) — system prompt override
    * `grammar` (TEXT) — GBNF grammar to constrain sampling
    * `grammar_triggers` (JSON array) — when set with `grammar`, the grammar activates lazily once a trigger string appears
    * `response_format` (`"json"`) — engine applies a built-in lazy JSON grammar so output is a valid JSON value
* Returns: `TEXT`.
* Cached on identical (prompt, options) tuple within a session.
* Local LLMs are slow per-row — use `GENERATE` for small result sets, not full-table scans.

```sql
-- Pre-v1.8.7 form — still works.
SELECT id,
       GENERATE('Summarize this customer review in one sentence: ' || review_text) AS summary
  FROM reviews
 WHERE rating <= 2
 LIMIT 50;

-- v1.8.7+ — deterministic JSON output with full sampling control.
SELECT GENERATE(
  'Extract product, sentiment, reason as JSON: ' || review_text,
  json_object(
    'max_tokens', 1024,
    'temperature', 0.2,
    'top_p', 0.95,
    'seed', 42,
    'response_format', 'json'
  )
) AS analysis
  FROM reviews LIMIT 10;
```

### `json_object(key1, value1, key2, value2, ...)` *(v1.8.7+)*

Builds a JSON object literal from an even-length alternating list of `(key, value)` arguments. Designed for the options bag passed to `GENERATE` and other AI functions, but usable anywhere a `JSON` value is accepted (column default, expression, INSERT). Keys must be `TEXT`; values can be any scalar SQL type — they're serialized to their JSON form. Errors on odd-length argument lists or non-text keys.

* Returns: `JSON`.

```sql
-- Inline options for GENERATE.
SELECT GENERATE('Answer in one word: capital of France?',
                json_object('max_tokens', 8, 'temperature', 0.0)) AS answer;

-- Use as a JSON column default.
CREATE TABLE prefs (
  user_id INTEGER PRIMARY KEY,
  settings JSON DEFAULT json_object('theme', 'dark', 'notifications', true)
);
```

### `SEMANTIC_MATCH`, `MULTI_MODAL_SIMILARITY`, `CROSS_MODAL_SEARCH`

Higher-level helpers used inside `SEMANTIC JOIN` and multi-modal queries. Prefer `EMBED` + `COSINE_SIMILARITY` for explicit similarity, and use `SEMANTIC_MATCH` only in `SEMANTIC JOIN` clauses.

### Other built-in AI functions

`CLASSIFY(text, categories)`, `EXTRACT_ENTITIES(text)`, `SENTIMENT(text)`, `SUMMARIZE(text)`, `TRANSLATE(text, target_lang)`.

### `MEMORY_STORE(namespace, content [, metadata])` — *v1.8.5+*

Stores text in an agent-memory namespace. On first call for a namespace the engine auto-creates the backing table `_memory_<namespace>` `(id TEXT PK, content TEXT NOT NULL, embedding VECTOR(384) NOT NULL, metadata TEXT, created_at TIMESTAMP, accessed_at TIMESTAMP)`. The content is embedded via the configured embedding model (default `all-minilm:latest`, dim 384) and a row is INSERTed under a sortable id (`mem_<base32 ts>_<6 alphanum>`).

* `namespace` MUST match `^[A-Za-z_][A-Za-z0-9_]*$`.
* `metadata` is optional and stored verbatim — convention: JSON-encoded with fields like `importance`, `kind`, `source`.
* Returns: `TEXT` — the generated memory id.
* Per-tenant scoped via the standard storage path.

```sql
-- Store a user preference
SELECT MEMORY_STORE('default', 'I prefer Python over Java') AS memory_id;

-- Store with importance metadata
SELECT MEMORY_STORE('default', 'Customer renewed annual plan',
                    '{"importance": 0.9, "kind": "fact"}') AS memory_id;
```

### `MEMORY_RECALL(namespace, query [, top_k])` — *v1.8.5+, table-valued*

Semantically retrieves the most-similar stored memories for a free-text query. Used in a `FROM` clause. Embeds the query via the same provider as `MEMORY_STORE`, scans the namespace's backing table, computes cosine similarity, and returns the top `top_k` rows ordered by similarity DESC. Columns: `(id TEXT, content TEXT, similarity REAL in [0,1], metadata TEXT, created_at TIMESTAMP)`. `top_k` defaults to 10, capped at 100. If `_memory_<namespace>` doesn't exist yet, returns an empty result set (not an error). Best-effort bumps `accessed_at` on the rows returned.

```sql
-- Retrieve top 5 similar memories
SELECT id, content, similarity
  FROM MEMORY_RECALL('default', 'what languages do I like', 5);

-- Join recalled memories with a downstream prompt
WITH r AS (
  SELECT content, similarity
    FROM MEMORY_RECALL('default', 'return policy', 3)
)
SELECT GENERATE('Given context: ' || STRING_AGG(content, ' | ') ||
                '. Answer: what is the return policy?') AS answer
  FROM r;
```

### `MEMORY_FORGET(namespace, id)` — *v1.8.5+*

Hard-deletes the memory identified by `id`. Returns `BOOLEAN` — `true` if a row was deleted, `false` if the id didn't exist. Same namespace validation as `MEMORY_STORE`. If `_memory_<namespace>` doesn't exist yet, returns `false` (no error).

```sql
-- Forget a single memory by id
SELECT MEMORY_FORGET('default', 'mem_abc123') AS deleted;

-- Age-out cold memories
SELECT MEMORY_FORGET('default', id) AS deleted
  FROM (
    SELECT id FROM MEMORY_RECALL('default', 'irrelevant', 100)
     WHERE similarity < 0.2
  ) AS cold;
```

---

## Native-inference model lifecycle (v1.8.0+)

v1.8.0-ce ships an in-process OCI v2 model registry: the gateway can
pull GGUF models from `registry.ollama.ai` (or any Docker Distribution
v2 registry) and serve them via the embedded `local` provider — no
external Ollama daemon, no separate process. The three functions below
expose that registry as SQL, alongside the equivalent `synapcores pull`
/ `synapcores models list` CLI commands.

These functions are active when the gateway is running with
`[query.ai_service].provider = "local"` (the v1.8 default — set
automatically when `[query.ai_service]` is omitted from `gateway.toml`).
The model store lives under `data_dir/models/` with sha256-addressed
blobs and JSON manifest sidecars.

### `PULL_MODEL(name)` — fetch a model into the local store

* Argument: `TEXT` — model reference. Accepts `name`, `name:tag`,
  `namespace/name[:tag]`, or `registry/namespace/name[:tag]`.
  Defaults: registry=`registry.ollama.ai`, namespace=`library`,
  tag=`latest`.
* Returns: `TEXT` — the resolved manifest digest.
* Idempotent: a second pull with the same name short-circuits when
  the local manifest digest matches the registry's current digest;
  no blob bytes are re-fetched.
* Resume: interrupted pulls leave a `.partial` file; re-running
  `PULL_MODEL` resumes from the byte offset already on disk.
* Min engine version: **1.8.0**.

```sql
-- Pull the default 7B chat model (the v1.8 install-script default).
SELECT PULL_MODEL('qwen2.5-coder:7b');

-- Pull an embedding model (used by EMBED + AGENT_RUN's memory layer).
SELECT PULL_MODEL('library/all-minilm:latest');

-- Pull from a third-party namespace.
SELECT PULL_MODEL('bartowski/Llama-3.2-3B-Instruct-GGUF:Q4_K_M');
```

### `LIST_MODELS()` — inventory the local model store

* Arguments: none.
* Returns: a table with columns
  `(name TEXT, architecture TEXT, size_bytes BIGINT, digest TEXT, pulled_at TIMESTAMP, last_used_at TIMESTAMP)`.
* Reads on-disk manifest sidecars only — never touches the network.
* Min engine version: **1.8.0**.

```sql
-- Inventory installed models.
SELECT * FROM LIST_MODELS();

-- Total disk used by the model store.
SELECT SUM(size_bytes) AS total_bytes FROM LIST_MODELS();
```

### `DELETE_MODEL(name)` — remove a model from the local store

* Argument: `TEXT` — model reference (same name forms as `PULL_MODEL`).
* Returns: `TEXT` — the digest of the removed manifest.
* Reference-counts content-addressed blobs: a blob shared with another
  tag is kept on disk until the last reference is dropped.
* Errors if the model is currently loaded in the LRU; unload it first
  by pointing `[query.ai_service].model` elsewhere, or restart the
  gateway.
* Min engine version: **1.8.0**.

```sql
-- Remove a model that's no longer needed.
SELECT DELETE_MODEL('library/qwen2.5-coder:0.5b');
```

---

## AutoML

AutoML trains real models from a SQL `SELECT` and exposes the trained model as an in-SQL function `AUTOML.PREDICT`. Training and prediction are first-class SQL — you do not call out to Python.

### `CREATE EXPERIMENT` — train a model

**Syntax (current — v1.6.5):**

```sql
CREATE EXPERIMENT model_name AS
  SELECT feature_1, feature_2, ..., label_column AS target
    FROM training_table
   [WHERE ...]
WITH (
    task_type        = 'binary_classification' | 'multi_classification' | 'regression'
                       | 'clustering' | 'time_series',
                       -- 'anomaly_detection' task_type — coming in v1.8 (Algorithm::IsolationForest + ANOMALY_SCORE())
    target_column    = 'target',
    [optimization_metric = 'auc' | 'accuracy' | 'f1' | 'rmse' | 'mae' | ...,]
    [max_trials      = 50,]
    [algorithms      = ['logistic_regression', 'random_forest', 'gradient_boosting']]
);
```

* The `target` column from the SELECT becomes the label. By convention, for binary classification `target = 1` is the positive class (e.g. fraud, churn).
* Without `algorithms`, AutoML runs in Auto mode and explores a sensible default set.
* Validation predictions are calibrated with isotonic regression for binary tasks, so `AUTOML.PREDICT` returns a well-calibrated `P(class=1)`.
* `CREATE EXPERIMENT ASYNC name AS ...` schedules training in the background; poll with `SHOW MODELS` / `DESCRIBE MODEL`.

**Algorithm options:**

| Algorithm           | Best for                              | Speed     | Accuracy        |
|---------------------|---------------------------------------|-----------|-----------------|
| logistic_regression | Binary classification, interpretable  | Fast      | Good            |
| linear_regression   | Simple regression, interpretable      | Fast      | Good for linear |
| random_forest       | General purpose, robust               | Medium    | High            |
| gradient_boosting   | High accuracy, competitions           | Slow      | Very High       |
| neural_network      | Complex patterns, large data          | Slow      | High            |
| knn                 | Simple, local patterns                | Fast      | Medium          |
| svm                 | Binary classification, kernels        | Medium    | High            |
| naive_bayes         | Text classification, simple           | Very fast | Medium          |

**Worked example — train a churn model:**

```sql
CREATE EXPERIMENT churn_model_v1 AS
  SELECT tenure_months,
         monthly_charges,
         total_charges,
         visits_30d,
         churned AS target
    FROM customers
WITH (
    task_type        = 'binary_classification',
    target_column    = 'target',
    optimization_metric = 'auc',
    max_trials       = 30,
    algorithms       = ['logistic_regression', 'random_forest', 'gradient_boosting']
);
```

### `AUTOML.PREDICT(...)` — predict with a trained model

**Syntax:**

```sql
SELECT pass_through_col_1, pass_through_col_2, ...,
       AUTOML.PREDICT('model_name', feature_1, feature_2, ...) [AS alias]
  FROM scoring_table
 [WHERE ...]
 [ORDER BY alias DESC|ASC]
 [LIMIT n];
```

* First argument is the **model name as a quoted string**.
* Remaining arguments are the **feature columns**, in any order — they are matched by name to the model's feature schema.
* Returns:
  * Binary classification: calibrated `P(target = 1)` as `DOUBLE`.
  * Multiclass: probability of the predicted (top) class.
  * Regression: the raw numeric prediction.
* Default alias is `prediction` if `AS alias` is omitted.
* You may sort or filter on the alias (`ORDER BY alias DESC`, `WHERE alias > 0.8`) — the planner pushes the prediction down so the alias is in scope.

**Worked example — rank customers by churn risk:**

```sql
SELECT id, name, tier,
       AUTOML.PREDICT('churn_model_v1',
                      tenure_months, monthly_charges, total_charges, visits_30d) AS risk
  FROM customers
 WHERE tier = 'Gold'
 ORDER BY risk DESC
 LIMIT 50;
```

**Important — feature columns also in the projection (v1.6.5.1):**
If a feature column is also a pass-through column, the planner dedupes it automatically; you do NOT need to list it twice:

```sql
-- OK: tenure_months in BOTH pass-through and features — dedupe handles it.
SELECT id, tenure_months,
       AUTOML.PREDICT('churn_model_v1', tenure_months, monthly_charges) AS risk
  FROM customers
 ORDER BY risk DESC;
```

### Model lifecycle

```sql
SHOW MODELS;                       -- list all models in current tenant
DESCRIBE MODEL churn_model_v1;     -- schema, algorithm, metrics, training time
DROP MODEL churn_model_v1;         -- delete model artifacts

SHOW EXPERIMENTS;                  -- list (legacy) experiments
DESCRIBE EXPERIMENT name;
```

**Known limitations:**

* Model artifacts live under `<data_dir>/models/` and are not portable across binaries (the in-memory representation may change with version upgrades).
* Models are tenant-prefixed; a tenant cannot use another tenant's models.
* `AUTOML.PREDICT` is supported in the SELECT projection. Wrapping it inside a `FROM (...) AS sub` subquery requires the v1.6.5.1 outer-subquery-wrap fix and works for `ORDER BY alias` / `LIMIT` but not for arbitrary outer projection rewrites.

---

## Cypher Graph Queries

AIDB ships a per-tenant property graph engine with a Cypher subset. Cypher statements are routed automatically by `/v1/query/execute` (and by the SQL `/v2/query/execute` path) — you do not need a separate endpoint.

### Read patterns

```cypher
-- Find all nodes with a given label
MATCH (n:Person) RETURN n LIMIT 100;

-- Filter on properties
MATCH (n:Person) WHERE n.age >= 18 RETURN n.name, n.age;

-- Traverse a relationship
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->(b:Person)
RETURN b.name;

-- Variable-length pattern + filter on the path
MATCH (a:Account)-[:TRANSFERRED*1..3]->(b:Account)
 WHERE a.owner = 'alice@example.com'
RETURN a.id, b.id;
```

### Write patterns

```cypher
-- Create a node
CREATE (p:Person {name: 'Bob', age: 30});

-- Create a relationship between two existing nodes
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CREATE (a)-[:KNOWS {since: 2026}]->(b);

-- MERGE = match-or-create (good for ingest pipelines)
MERGE (p:Patient {mrn: 'MRN-101'})
MERGE (drug:Drug {name: 'Warfarin'})
MERGE (p)-[:PRESCRIBED]->(drug);

-- Delete a node and ALL its relationships
MATCH (n:Person {name: 'Charlie'}) DETACH DELETE n;

-- UNWIND a list to bulk-create
UNWIND [{name: 'Alice', age: 30}, {name: 'Bob', age: 25}] AS row
CREATE (:Person {name: row.name, age: row.age});
```

### When to use graph vs SQL JOIN

* Use a SQL JOIN for tabular, fixed-depth relationships you already model in tables.
* Use Cypher when you need multi-hop traversals, variable-length paths, or to express "find me everyone reachable from X via these edge types" succinctly. Cypher beats N-way SQL self-joins on graph-shaped data.

### Discovery

```sql
SHOW PROPERTY GRAPHS;     -- list graphs in this tenant
CALL db.labels();         -- list all node labels in the active graph
```

---

## Multi-modal SQL

For images, audio, video, and PDF stored in `IMAGE`/`AUDIO`/`VIDEO`/`PDF` columns, AIDB exposes:

```sql
-- Embed any modality and search across modalities
SELECT id,
       MULTI_MODAL_SIMILARITY(text  := description,
                              image := cover_image,
                              weights := '{"text":0.6,"image":0.4}') AS score
  FROM products
 ORDER BY score DESC LIMIT 10;

-- Semantic JOIN: match rows by semantic similarity instead of equality
SELECT a.id, b.id
  FROM articles a
SEMANTIC JOIN reference_docs b
    ON SEMANTIC_MATCH(a.body, b.text, threshold := 0.75);
```

`MULTI_MODAL_SIMILARITY(...)`, `CROSS_MODAL_SEARCH(...)`, and `SEMANTIC_MATCH(...)` accept named arguments using the `name := value` syntax.

---

## Triggers and Procedures

```sql
CREATE [OR REPLACE] TRIGGER trg_name
  {BEFORE | AFTER} {INSERT | UPDATE | DELETE} ON table_name
  [FOR EACH ROW]
  [WHEN (condition)]
  EXECUTE PROCEDURE proc_name(args);

DROP TRIGGER [IF EXISTS] trg_name ON table_name;

CREATE [OR REPLACE] PROCEDURE proc_name(args) AS $$
BEGIN
    -- procedure body
END;
$$ LANGUAGE plpgsql;

DROP PROCEDURE [IF EXISTS] proc_name;
CALL proc_name(args);

SHOW PROCEDURES [LIKE 'pattern'];
SHOW TRIGGERS [FROM table_name] [LIKE 'pattern'];
```

---

## Natural Language

```sql
ASK '<natural language question>';            -- run a natural language query
EXPLAIN NATURAL '<natural language question>'; -- show the SQL plan
```

---

## Critical "do's and don'ts" for AIDB recipes

**DO** prefer the AIDB-native extension when the intent matches:

* Need text/image similarity?              -> `EMBED` + `COSINE_SIMILARITY` (or `MULTI_MODAL_SIMILARITY`).
* Need a trained model?                    -> `CREATE EXPERIMENT ... WITH (...)` then `AUTOML.PREDICT(...)`.
* Need risk-ranked output?                 -> `ORDER BY <prediction_alias> DESC LIMIT N` directly.
* Need multi-hop relationships?            -> Cypher `MATCH`, not N-way self-joins.
* Need LLM-generated text per row?         -> `GENERATE(prompt)` in the SELECT.

**DON'T**:

* Don't invent functions or syntax not in this manual.
* Don't list a feature column twice when it's also in the pass-through projection — `AUTOML.PREDICT` dedupes.
* Don't use placeholder comments like `-- your SQL here` or `[bracket placeholders]` — every recipe must have actual, runnable SQL.
* Don't include `DROP TABLE`/`DROP INDEX` cleanup steps in recipes (they delete user data).
* Don't store an embedding in a column whose declared dimension differs from the model's output dim — that is a runtime error.
