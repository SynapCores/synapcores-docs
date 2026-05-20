# SQL Reference

!!! note "Auto-synced from source"
    This page is **auto-synced** from the SynapCores AIDB engine
    repository on every release tag. The canonical source is
    `AIDB_SQL_MANUAL.md` in the engine repo — **do not edit this
    page directly**; your change will be overwritten on the next release.

    **Last synced from**: `v1.6.6-ce` on 2026-05-20


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
| LLM text generation          | `GENERATE(prompt_text)` -> TEXT                                       |
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

### `GENERATE(prompt)`

Calls the configured completion model and returns the generated text.

* Argument: `TEXT` prompt.
* Returns: `TEXT`.
* Cached on identical prompt within a session (call latency drops to ~0 after first call).
* Local LLMs are slow per-row — use `GENERATE` for small result sets, not full-table scans.

```sql
SELECT id,
       GENERATE('Summarize this customer review in one sentence: ' || review_text) AS summary
  FROM reviews
 WHERE rating <= 2
 LIMIT 50;
```

### `SEMANTIC_MATCH`, `MULTI_MODAL_SIMILARITY`, `CROSS_MODAL_SEARCH`

Higher-level helpers used inside `SEMANTIC JOIN` and multi-modal queries. Prefer `EMBED` + `COSINE_SIMILARITY` for explicit similarity, and use `SEMANTIC_MATCH` only in `SEMANTIC JOIN` clauses.

### Other built-in AI functions

`CLASSIFY(text, categories)`, `EXTRACT_ENTITIES(text)`, `SENTIMENT(text)`, `SUMMARIZE(text)`, `TRANSLATE(text, target_lang)`.

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
                       | 'clustering' | 'anomaly_detection' | 'time_series',
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
