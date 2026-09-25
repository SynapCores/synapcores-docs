# SQL Reference

!!! note "Auto-synced from source"
    This page is **auto-synced** from the SynapCores AIDB engine
    repository on every release tag. The canonical source is
    `AIDB_SQL_MANUAL.md` in the engine repo — **do not edit this
    page directly**; your change will be overwritten on the next release.

    **Last synced from**: `v1.16.0-ce` on 2026-09-25


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
| Persona as a database object | `CREATE PERSONA name WITH (system_prompt = '...')` (v1.14.2+)          |
| Persona inspection           | `SHOW PERSONAS [LIKE '...']`, `DESCRIBE PERSONA name` (v1.14.2+)       |
| Fire a durable agent async   | `WAKE AGENT name` (v1.14.2+)                                          |
| Agent-to-agent chaining      | `... ON INSERT INTO t ALLOW AGENT ORIGIN` (v1.14.2+)                   |
| Persistent agent memory      | `CREATE MEMORY name IDENTITY user_id` (v1.14.3+)                       |
| Write a memory               | `REMEMBER mem FOR user_id = 123 '<text>'` (v1.14.3+)                   |
| Read assembled context       | `RECALL mem FOR user_id = 123 ABOUT '<query>'` (v1.14.3+)              |
| Authoritative present value  | `CURRENT mem FOR user_id = 123 ATTRIBUTE attr` (v1.14.3+)              |
| Explain a remembered value   | `TRACE mem FOR user_id = 123 ATTRIBUTE attr` (v1.14.3+)                |
| Authorized removal           | `FORGET mem FOR user_id = 123 ABOUT '<scope>'` (v1.14.3+)              |
| Query Parquet on object storage | `CREATE EXTERNAL TABLE t STORED AS PARQUET LOCATION 's3://...'` *(v1.16.0-ce+)* |
| Re-read a lake schema        | `REFRESH EXTERNAL TABLE t` *(v1.16.0-ce+)*                             |
| List lake tables             | `SHOW EXTERNAL TABLES` *(v1.16.0-ce+)*                                 |

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

**Casts.** `CAST(expr AS type)` and the `::type` shorthand both work.
`SIGNED`, `UNSIGNED`, `SIGNED INTEGER` and `UNSIGNED INTEGER` are accepted as
cast targets for MySQL compatibility (v1.16.0-ce) and map to `BIGINT`, so a
query copied from MySQL — or emitted by a BI tool against the MySQL wire
port — parses unchanged. `UNSIGNED` is a spelling only; the result is a signed
64-bit integer.

Casting **to** `DECIMAL` is accepted from any numeric or text source, but the
precision and scale you write are **not applied**:

```sql
SELECT CAST(12.5 AS DECIMAL(10,2));     -- 12.5, exact, still scale 1
SELECT CAST('12.5' AS DECIMAL(10,2));   -- 12.5, but APPROXIMATE (a DOUBLE)
SELECT CAST(12 AS DECIMAL(10,2));       -- 12.0, approximate
```

A `DECIMAL` source passes through unchanged at its own scale; a `TEXT` or
integer source becomes an approximate `DOUBLE`. So a cast is not the way to
rescale money. Declare the column at the scale you want and let `INSERT` or
`UPDATE` coerce to it — as of v1.16.0-ce both do, and the declared scale is
honoured:

```sql
CREATE TABLE invoices (id INT PRIMARY KEY, amount DECIMAL(10,2));
INSERT INTO invoices VALUES (1, 29.99);
SELECT amount FROM invoices;            -- 29.99
UPDATE invoices SET amount = 7.7 WHERE id = 1;
SELECT amount FROM invoices;            -- 7.70, padded to the declared scale
```

**Column constraints:** `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, `CHECK (expr)`, `DEFAULT expr`, `REFERENCES other_table(other_col)`.

**Table constraints:** `PRIMARY KEY (a, b)` — including the composite form — is
honoured as of v1.15.0-ce, and creates both the primary key and its index.
Out-of-line `UNIQUE (...)` and `FOREIGN KEY (...)` are parsed but not yet
enforced; declare those at the column level.

!!! warning "Before v1.15.0-ce"
    A table-level `PRIMARY KEY (a, b)` was accepted and then silently discarded:
    `SHOW INDEXES` returned nothing and the table had no primary key at all. If
    you have tables declared that way, re-running their `CREATE TABLE ... IF NOT
    EXISTS` on this version registers and backfills the missing index.

**Constraint enforcement.** `PRIMARY KEY`, `UNIQUE` and `CHECK (expr)` are enforced on write —
a violating statement errors and no row is persisted. Since **v1.14.3**:

* `CHECK` is enforced on `UPDATE` as well as `INSERT` and `INSERT ... SELECT`
  (before v1.14.3, `UPDATE` skipped it, so a row could be inserted legally and
  then updated into a state the constraint forbids);
* a **table-level** `CHECK`, written after the column list, is enforced too:

  ```sql
  CREATE TABLE scores (
      score INTEGER,
      CONSTRAINT score_non_negative CHECK (score >= 0)
  );

  INSERT INTO scores (score) VALUES (-5);
  -- ERROR: Table CHECK constraint 'score_non_negative' violated (row 1)
  ```

  (before v1.14.3 a table-level `CHECK` parsed, was accepted, and then enforced
  nothing — only inline column `CHECK`s did anything);
* a `CHECK` survives a restart; it used to be dropped when the schema was
  reloaded from the on-disk cache.

`NULL` follows SQL three-valued logic: a `CHECK` rejects a row only when the
predicate evaluates to `FALSE`. `CHECK (grade IN ('A','B'))` therefore **accepts**
a `NULL` grade, because the predicate is `UNKNOWN`. Add `NOT NULL` if you want to
forbid that.

**Out-of-line `PRIMARY KEY`, including the composite form, is honoured since
v1.15.0-ce.**

```sql
CREATE TABLE zip_fips_crosswalk (
    zip_code VARCHAR NOT NULL,
    tract    VARCHAR NOT NULL,
    v        VARCHAR,
    PRIMARY KEY (zip_code, tract)
);

SHOW INDEXES FROM zip_fips_crosswalk;
-- PRIMARY                (zip_code, tract)
-- zip_fips_crosswalk_pkey (zip_code, tract)

INSERT INTO zip_fips_crosswalk VALUES ('10001', 'A', 'x');
INSERT INTO zip_fips_crosswalk VALUES ('10001', 'A', 'y');
-- ERROR: Duplicate key: row violates a UNIQUE or PRIMARY KEY constraint
```

Before v1.15.0-ce that `CREATE TABLE` returned **ok** and then produced no primary
key, no index and no error — `SHOW INDEXES` came back empty. A single-column
out-of-line `PRIMARY KEY (a)` behaves exactly like the inline `a ... PRIMARY KEY`
spelling.

Storage note: a table with a **single-column** primary key stores each row under
its primary-key value, so a lookup by that key is a direct fetch. A table with a
**composite** primary key keeps the document store's own row id, and the
composite key is enforced through the primary key's own index instead.

Out-of-line `UNIQUE (...)` and `FOREIGN KEY (...)` are still not enforced —
declare those inline on the column.

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

### Columnar tables — `ENGINE = SYNAPCORES_COLUMNAR` *(v1.15.0-ce+)*

Stores the table as Parquet parts with RocksDB metadata instead of row storage.
Built for analytical scans over large tables: a query reads only the columns it
names, and row-group statistics skip parts that cannot match.

```sql
CREATE TABLE events (
    id      INTEGER PRIMARY KEY,
    sensor  TEXT,
    reading DOUBLE,
    ts      TIMESTAMP
) ENGINE = SYNAPCORES_COLUMNAR;
```

`ENGINE = SYNAPCORES_COLUMNAR` is the whole syntax — it takes no further
options. Adding a trailing option (`, COMPRESSION = 'zstd'`, `, CHECKSUM = …`,
`, BLOCK_SIZE = …`) is a **parse error**, and a `WITH ( … )` clause on
`CREATE TABLE` must not be used at all: it silently retypes every column to
`TEXT` and discards the primary key.

!!! warning "`PRIMARY KEY` and `UNIQUE` are NOT enforced on columnar tables"
    On a columnar table a `PRIMARY KEY` or `UNIQUE` declaration is a
    clustering and metadata hint. **Duplicate keys are accepted** — the insert
    succeeds and both rows are stored.

    This follows the analytical-store convention: Snowflake, BigQuery, Amazon
    Redshift, ClickHouse, Apache Iceberg and Delta Lake all allow the
    declaration without enforcing it, because a uniqueness probe on every
    insert removes the bulk-ingest advantage that is the reason to use
    columnar storage at all. BigQuery makes you write `NOT ENFORCED`
    explicitly.

    **A row-storage table in this engine DOES enforce it.** The same
    statement therefore behaves differently depending on `ENGINE`, so
    `CREATE TABLE` emits a warning when a columnar table declares one.

    If you need uniqueness: de-duplicate at ingest, or use a row table for
    the keyed data and a columnar table for the analytical copy.



Columnar tables support the full `INSERT` / `UPDATE` / `DELETE` surface. Writes
are copy-on-write — an affected part is rewritten rather than edited in place,
which is why an `UPDATE` or `DELETE` touching a large part costs more than the
same statement against a row table. Prefer batched writes and analytical reads.

!!! note "Before v1.15.0-ce"
    `ENGINE = SYNAPCORES_COLUMNAR` parsed and was then normalised away, so the
    table was silently created as a row table. Tables created that way are row
    tables and stay row tables; recreate them to move to columnar storage.

### ALTER TABLE / DROP TABLE / CREATE INDEX / DROP INDEX

```sql
ALTER TABLE t ADD COLUMN c data_type [constraint];
ALTER TABLE t DROP COLUMN c;
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t ALTER COLUMN c TYPE new_type;

DROP TABLE [IF EXISTS] t [CASCADE];

-- Both clause orders are accepted (v1.15.0-ce): the MySQL trailing order and
-- the PostgreSQL leading order.
CREATE [UNIQUE] INDEX [IF NOT EXISTS] idx_name ON t (col [ASC|DESC], ...) [USING BTREE|HASH];
CREATE [UNIQUE] INDEX [IF NOT EXISTS] idx_name ON t USING BTREE|HASH (col [ASC|DESC], ...);
DROP   INDEX [IF EXISTS] idx_name;
```

**Index types.** `BTREE` (the default) and `HASH` are the only implemented
types. Both clause orders are accepted since v1.15.0-ce — `USING` before the
column list (PostgreSQL style) and after it (MySQL style). Earlier versions
rejected the trailing form with a parse error.

`USING HNSW` and `USING IVFFLAT` are **rejected**. There is no SQL-level vector
index. For similarity search use `COSINE_SIMILARITY`, which scans, or the
dedicated vector-collection REST surface (`POST /v1/vectors/collections`).

An index on a `VECTOR(N)` column with no `USING` clause is also **rejected**: a
B-tree over an embedding accelerates nothing, and creating one silently would
leave a table that looks indexed and behaves as though it is not.

**`CREATE INDEX` builds the index from the rows already in the table.** It
therefore takes time proportional to table size and reports how much it did:

```
Index idx_zip created on table zip_crosswalk (B-tree); 189361 rows indexed
```

Rows holding `NULL` in an indexed column are skipped and counted separately —
they can never satisfy an equality lookup. If the build fails, the statement
fails and **the index is not created**; a half-populated index would be worse
than none, because a lookup that misses it would silently return no row.

`CREATE UNIQUE INDEX` over data that already contains a duplicate fails, naming
the column and the repeated value.

!!! warning "Before v1.15.0-ce: two ways an index ended up empty"
    `CREATE INDEX` did not read the table — it relied entirely on INSERT-time
    maintenance. So an index created over a table that **already held rows** was
    left empty, and a lookup on it silently fell back to a full table scan
    (correct answers, no acceleration, no error).

    Separately, INSERT-time index maintenance was gated on the table's
    *constraint* list rather than its index catalog, so on a table with **no
    `PRIMARY KEY` and no `UNIQUE` column** even rows inserted *after* the index
    existed never reached it.

    An index on a table that had a `PRIMARY KEY` or `UNIQUE` column and was
    created **before** the rows were loaded worked correctly on earlier
    versions. Measured on v1.14.5-ce, 20k rows, same box:

    | scenario | v1.14.5-ce |
    |---|---|
    | PK table, index created before the load | 1.0 ms |
    | PK table, index created after the load | 66.6 ms |
    | no-PK table, index created before the load | 66.3 ms |
    | unindexed column (control) | 72.1 ms |

    Indexes created on an older build are registered but empty. Upgrading
    repopulates them at startup -- measured: a lookup that cost 66.6 ms on
    v1.14.5-ce serves in 0.9 ms after opening the same data directory with
    v1.15.0-ce, with no `CREATE INDEX` re-run.

**Indexes belong to the database that created them.** An index created in
database `analytics` is used only by queries running against `analytics`.

**Supported index types: `BTREE` (default) and `HASH`.**

**`CREATE INDEX` builds the index from the rows already in the table (v1.15.0-ce).**
The statement scans the table, populates the index and reports how many rows it
indexed:

```
Index idx_zip created on table zip_crosswalk (B-tree); 189361 rows indexed
```

It therefore takes time proportional to the table size — a large table is not
instantaneous, and that is the point. Before v1.15.0-ce the statement returned in
milliseconds no matter how big the table was and left the index **empty**: the
planner then fell back to a full table scan on every lookup while
`SHOW INDEXES` listed the index and the definition survived restarts. Nothing
reported an error.

Rows whose indexed column is NULL are counted separately as skipped — such a row
can never satisfy `col = <value>`, so leaving it out changes no answer. If the
build fails, the statement fails and the index is **not** created: a partially
built index would make lookups return fewer rows than the table actually holds.

**`CREATE UNIQUE INDEX` over data that already contains duplicates fails
(v1.15.0-ce)**, naming the column and the repeated value:

```
CREATE UNIQUE INDEX idx_email ON users (email);
-- ERROR: cannot create UNIQUE index 'idx_email' on table 'users': the table
--        already contains a DUPLICATE VALUE in the indexed column (email).
--        The row with email = 'dupe@x.com' repeats a value an earlier row
--        already has. De-duplicate the table first, or create a non-unique
--        index. The index was NOT created.
```

The `UNIQUE` flag is also now recorded on every code path; one lowering
discarded it and quietly produced a plain index, so the declared guarantee was
present on one route and absent on the other.

**`CREATE UNIQUE INDEX` also rejects duplicates written AFTER it exists
(v1.15.0-ce).** A `UNIQUE` index is a constraint, not a hint: an INSERT whose value
already appears in the index is rejected on every path.

```
CREATE UNIQUE INDEX idx_email ON users (email);
INSERT INTO users (id, email) VALUES (2, 'a@x.com');   -- value already present
-- ERROR: Duplicate key: row violates a UNIQUE or PRIMARY KEY constraint on
--        table 'users'
```

Before v1.15.0-ce the write-time check consulted only the constraints declared in
`CREATE TABLE`, so an index created by `CREATE UNIQUE INDEX` — which the table
schema never learns about — enforced nothing at all. The duplicate was written,
the index layer's own rejection was logged and discarded, and the only symptom
was duplicated data.

**Indexes are rebuilt on first use after a restart (v1.15.0-ce).** Index
definitions are durable; the index data structures live in memory and are
rebuilt from the table the first time a statement touches it, not at startup —
so startup time does not grow with the size of the database, and the first
statement against a large table pays a one-off scan (about half a second per
10,000 rows per table in a debug build, less in a release build). Until an index
is rebuilt it is never treated as authoritative: lookups fall back to a table
scan and uniqueness is decided by a direct check, so results are correct
throughout.

This is the fix for a defect present through v1.14.5: after a restart every
index was registered but empty, and the emptiness was read as an answer. A
duplicate `INSERT` was accepted — and because a single-column `PRIMARY KEY` is
the row's storage id, it **overwrote** the existing row with no error. The first
write after a restart also made the empty index non-empty, after which an
indexed lookup returned only the rows written since the restart while
`COUNT(*)` still counted them all.

If an index cannot be rebuilt — most often a `UNIQUE` index over rows that
already contain duplicates — the engine logs the table, the index, the column
and the offending value, leaves the index empty and unused, and keeps answering
from table scans. `REINDEX <index>` is the recovery once the data is fixed.

**An index belongs to the database that created it (v1.15.0-ce).** Each database
keeps its own index catalog. `CREATE INDEX` issued inside `USE demo` — or with
`"database": "demo"` on the REST call — creates an index that only `demo`'s
queries use and that only `demo`'s rows go into.

Through v1.14.5 the planner looked indexes up under the bare table name while
every writer filed them under the database, so outside `main` no index was ever
chosen: `CREATE INDEX` reported success, `SHOW INDEXES` listed the index, and
every lookup still paid for a full table scan. Where `main` happened to hold a
table of the same name, the planner picked `main`'s index for the other
database's query. Both are fixed: an index is matched to its own database, and a
plan naming an index from a different database is discarded in favour of a table
scan rather than used. A 60,000-row lookup on an indexed non-key column in a
non-`main` database went from 214 ms to 0.4 ms, matching `main`.

**A composite index answers a leftmost-prefix lookup (v1.15.0-ce).** With
`CREATE INDEX idx ON t (a, b)`, `WHERE a = ?` uses the index. Before v1.15.0-ce
only the full key `WHERE a = ? AND b = ?` was served and any partial match
returned **zero rows** — creating a composite index silently removed rows from
query results. A predicate the index cannot serve (a non-leading column, or a
partial key on a `HASH` index) falls back to a table scan and returns correct
rows.

**Vector indexes are NOT supported at SQL level.** `USING HNSW` and
`USING IVFFLAT` parse and plan, but execution fails with an explicit error —
there is no HNSW structure behind the SQL index layer in this release.
`CREATE INDEX` on a `VECTOR(N)` column *without* a vector index type is also
rejected: a B-tree over an embedding accelerates nothing, and reporting
"(B-tree) created" for it was a silent failure that made operators believe the
column was indexed.

```sql
-- Rejected: no HNSW behind the SQL index layer.
CREATE INDEX i ON products(description_vec) USING HNSW;

-- Also rejected: decoy B-tree over a VECTOR column.
CREATE INDEX i ON products(description_vec);
```

For a dedicated vector store with metadata-filtered k-NN, use the vector
collection REST subsystem:

```
POST /v1/vectors/collections            {"name": "products", "dimensions": 384, "distance_metric": "cosine"}
POST /v1/vectors/collections/products/vectors      # insert or upsert by id
POST /v1/vectors/collections/products/search       # supports "filter" and "threshold"
```

Note: that subsystem answers **exact** k-NN (brute force). As of v1.15.0-ce an
`index_type` other than `"flat"` is **rejected with HTTP 400** rather than
accepted and ignored — earlier releases echoed the requested type back in the
response while always building a flat index. Exact search is correct, just not
sublinear. Scope for the SQL-level fix: `docs/proposals/sql_vector_indexes.md`.

Similarity SQL over a `VECTOR` column needs no index and keeps working — it is
a full scan:

```sql
SELECT id, COSINE_SIMILARITY(description_vec, EMBED('audio gear')) AS score
FROM products
ORDER BY score DESC
LIMIT 10;
```

### Immutable (append-only) tables and chain attestation

```sql
CREATE IMMUTABLE TABLE audit_log (
    id         BIGINT PRIMARY KEY,
    actor      TEXT NOT NULL,
    action     TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO audit_log (id, actor, action) VALUES (1, 'alice', 'LOGIN');

-- UPDATE and DELETE are refused on an immutable table.

-- -> is_valid BOOL, message TEXT, status TEXT,
--    records_verified INT, blocks_verified INT   (v1.12.0+; status/counts v1.14.2.1+)
VERIFY TABLE audit_log;
VERIFY RECORD 1 IN audit_log;    -- -> record_id, is_valid, message    (v1.12.0+)
```

`CREATE IMMUTABLE TABLE` writes every row into a hash-chained, append-only block store
in addition to the table itself.

`VERIFY TABLE <t>` attests that chain: it walks every block, recomputes each record
checksum, each sealed block's checksum and merkle root, and each block-to-block link,
then cross-checks the number of chained records against the rows actually stored.
`message` reports the number of records/blocks verified, or the first break found.
`VERIFY TABLE` on a table that is **not** immutable returns an error rather than a pass.

Read the counts from `records_verified` / `blocks_verified` rather than parsing `message`,
and branch on `status`, not on `is_valid` alone:

| `status` | `is_valid` | meaning |
|---|---|---|
| `verified` | `true` | the chain was walked and every record in it is attested (`records_verified` > 0) |
| `empty` | `true` | the chain holds **zero** records — the check is **vacuous**, nothing was attested. An empty chain is internally consistent, so `is_valid` stays `true`; it is *not* evidence of tampering, it usually means nothing was ever written |
| `failed` | `false` | at least one integrity problem was found; `message` names the first one |

A pipeline that claims "N/N runs verified" must therefore require
`status = 'verified'` **and** `records_verified > 0`.

`VERIFY RECORD <id> IN <t>` attests a single record: it recomputes that record's
checksum and proves its membership in its block's merkle tree. `<id>` is the row's SQL
id (matched against the chained payload) or the chain's internal record id.

Immutable tables must be created empty with an explicit column list —
`CREATE IMMUTABLE TABLE ... AS SELECT` is not supported; populate with
`INSERT` / `INSERT ... SELECT`.

**Encryption at rest (v1.12.0+):**

```sql
CREATE IMMUTABLE TABLE ssn_ledger (
    id      BIGINT PRIMARY KEY,
    subject TEXT NOT NULL,
    secret  TEXT NOT NULL
) WITH (ENCRYPTION='AES256GCM');
```

`WITH (ENCRYPTION='AES256GCM')` encrypts **both** the hash-chain and the row-engine copy
on disk. Requirements and behavior:

- Requires the `AIDB_IMMUTABLE_MASTER_KEY` environment variable (a hex-encoded key). If it
  is missing or invalid, the `CREATE` is **rejected** — never downgraded to a plaintext
  table (fail-closed).
- `AES256GCM` is the only supported algorithm. An unsupported or missing algorithm
  (e.g. bare `WITH (ENCRYPTION)` or `ENCRYPTION='AES128'`) is rejected.
- `ENCRYPTION_KEY='name'` (a named-key registry) is **not** supported in this release; use
  `ENCRYPTION='AES256GCM'` alone — the table gets an auto-generated per-table key wrapped
  by the master key.
- Encrypted immutable tables are supported only in the **default database**; creating one
  under another database is rejected.

---


### Columnar tables — `ENGINE = SYNAPCORES_COLUMNAR` *(v1.15.0-ce+)*

```sql
CREATE TABLE events (
    id      INTEGER PRIMARY KEY,
    sensor  TEXT,
    reading DOUBLE,
    ts      TIMESTAMP
) ENGINE = SYNAPCORES_COLUMNAR;
```

Stores the table as Parquet parts with RocksDB metadata instead of row storage.
A query reads only the columns it names, and row-group statistics skip parts
that cannot match the predicate.

`ENGINE = SYNAPCORES_COLUMNAR` is the entire syntax and takes no further
options — a trailing `, COMPRESSION = …` / `, CHECKSUM = …` / `, BLOCK_SIZE = …`
is a parse error. Do not use a `WITH ( … )` clause on `CREATE TABLE`: it
silently retypes every column to `TEXT` and drops the primary key.

Full `INSERT` / `UPDATE` / `DELETE` are supported, but writes are
**copy-on-write**: an affected part is rewritten rather than edited in place, so
a statement touching a large part costs more than the equivalent on a row table.
Batch the writes; read analytically. Columnar tables have no secondary indexes —
scanning is the access path, which is the point of the format.


**`PRIMARY KEY` and `UNIQUE` are NOT enforced on columnar tables.** The
declaration is a clustering/metadata hint; duplicate keys are accepted and both
rows are stored. This follows the analytical-store convention (Snowflake,
BigQuery, Redshift, ClickHouse, Iceberg, Delta all declare without enforcing) —
a uniqueness probe per insert would remove the bulk-ingest advantage columnar
exists for. **A row-storage table in this engine DOES enforce it**, so the same
statement behaves differently depending on `ENGINE`; `CREATE TABLE` warns when a
columnar table declares one. If you need uniqueness, de-duplicate at ingest or
keep the keyed copy in a row table.

**Before v1.15.0-ce** the `ENGINE` clause parsed and was then normalised away, so
the statement returned ok and silently created an ordinary **row** table. A table
created on an older build is a row table and stays one; recreate it to move to
columnar storage.

### External tables — Parquet on object storage *(v1.16.0-ce+)*

```sql
CREATE EXTERNAL TABLE [IF NOT EXISTS] [db.]name [(col type [NOT NULL], ...)]
  STORED AS PARQUET
  LOCATION '<uri>'
  [PARTITIONED BY (col type, ...)]
  [WITH (CONNECTION = '<destination name>')];

DROP EXTERNAL TABLE [IF EXISTS] [db.]name;
REFRESH EXTERNAL TABLE [db.]name;
SHOW EXTERNAL TABLES [FROM <database> | IN <database>] [LIKE '<pattern>'];
```

An **external table** is a name in the catalog pointing at a prefix in object
storage. Nothing is copied into the database and no storage is allocated — the
data stays where the export job wrote it, and `CREATE EXTERNAL TABLE` costs one
Parquet footer read.

What makes it worth having is that the planner treats it as an ordinary
relation. A lake table **joins, filters and aggregates against RocksDB tables in
one statement**, so a year of history in cheap object storage answers a question
alongside today's live rows — with no ETL step back into the database and no
second query engine whose results you join by hand in application code.

**External tables are read-only.** `INSERT`, `UPDATE` and `DELETE` are refused:

```
INSERT is not allowed on 'pings': it is an external table and is read-only.
Its data is PARQUET at s3://acme-lake/raw/pings/. Write to the lake, then run
REFRESH EXTERNAL TABLE pings.
```

**Worked example — register a lake table and join it to a live one:**

```sql
CREATE EXTERNAL TABLE pings
  STORED AS PARQUET
  LOCATION 's3://acme-lake/raw/pings/'
  PARTITIONED BY (dt DATE)
  WITH (CONNECTION = 'lake_s3');
-- External table 'pings' registered over PARQUET at s3://acme-lake/raw/pings/
-- (25 column(s), 1 partition column(s))

SELECT c.name, COUNT(*) AS calls
  FROM pings p
  JOIN customers c ON c.id = p.customer_id
 WHERE p.dt BETWEEN '2026-09-01' AND '2026-09-07'
 GROUP BY c.name
 ORDER BY calls DESC;
```

`customers` is an ordinary table; `pings` is Parquet on S3. One statement.

#### Partition layout

`PARTITIONED BY` declares the columns carried by the **directory name** rather
than by the files — the Hive-style layout the export job writes:

```text
s3://acme-lake/raw/pings/dt=2026-09-18/part-0000.parquet
s3://acme-lake/raw/pings/snapshot_dt=2026-09-18/part-0000.parquet
s3://acme-lake/raw/_manifests/pings/dt=2026-09-18.json
s3://acme-lake/raw/_state/pings.json
```

Only `dt=YYYY-MM-DD` and `snapshot_dt=YYYY-MM-DD` are recognised as partition
directories. Anything else is not a partition and is skipped, so a hand-made
`year=2026/` directory is invisible rather than half-read.

A partition column is **never repeated in the column list**. Declaring it in
both is a parse error, because the relation would then have two columns of the
same name and no predicate could say which one it meant.

Pruning partitions from their directory names, before any object is opened, is
what makes `WHERE dt BETWEEN …` on a year-wide table cheap.

!!! warning "Only partitions with a verified manifest are visible"
    The export job writes a partition's manifest **last**, after every data file
    is durable. Object stores have no rename, so that write order *is* the
    atomicity: a partition with no manifest, or one whose manifest is not marked
    verified, does not exist as far as a reader is concerned.

    A half-written export is therefore **invisible**, not partially readable.
    That is the behaviour you want: a partial partition read as if it were
    complete is a wrong answer that looks exactly like a right one.

    If a query returns fewer rows than you expect, the partition is usually
    still exporting or its export failed — check the job's run history before
    suspecting the query.

#### Schema: inferred, or declared and checked

Omit the column list and the schema is read from the Parquet footer at `CREATE`
time. Declare one and it is **checked against the data**, and a disagreement
fails the `CREATE`:

```
CREATE EXTERNAL TABLE orders: the declared schema does not match the data:
column 'amount' is declared DOUBLE PRECISION but the data holds BIGINT;
column 'region' is present in the data but not declared.
Omit the column list to take the schema from the data.
```

Failing here rather than at query time is deliberate — a table that registers
cleanly and breaks on first use usually breaks in front of a BI tool that cannot
show the error. Names match case-insensitively and types by meaning rather than
spelling; column **order** is not compared.

#### `WITH (CONNECTION = '…')`

Names the lake destination supplying credentials and path confinement.

**The grammar marks it optional; in practice it is required.** With no
`CONNECTION` the read path has no destination, and therefore no credentials, so
`CREATE` fails:

```
external table 'pings' has no CONNECTION; the lake cannot know which
destination — and therefore which credentials — to read it with
```

Always write it. It is the only supported `WITH` option, and an unknown option
is **rejected rather than ignored** — silently dropping `CONNECTION` would build
a table reading through credentials the operator did not choose.

`LOCATION` is resolved **relative to the destination's prefix** and checked
against that destination's read allowlist. A location outside it is refused,
naming what was compared:

```
external table 'pings' reads raw/pings/ through destination 'lake_s3', which is
outside its read allowlist (raw/approved/). Add the prefix to the destination
before querying it.
```

Anyone who can write SQL could otherwise read every object those credentials
reach, so the allowlist on the destination is the security boundary — not the
SQL statement.

`STORED AS PARQUET` is the only format (`STORED AS ORC` errors naming PARQUET),
`LOCATION` must be a non-empty quoted URI, and `PARTITIONED BY` / `WITH` may
appear in either order.

#### `REFRESH EXTERNAL TABLE`

An external table stores the column list it last resolved, so `DESCRIBE` and
BI-tool introspection answer without a network round trip. That stored schema is
what goes stale, and `REFRESH` is what updates it — run it after an export job
adds or retypes a column:

```sql
REFRESH EXTERNAL TABLE pings;
-- External table 'pings' refreshed: 26 column(s) (was 25)
```

**New partitions do not need it.** Partitions are discovered per scan and cached
for `partition_cache_ttl_secs` (default 300 seconds; set it to `0` to discover
on every scan), so yesterday's export becomes visible on its own inside that
window. `REFRESH` re-resolves the schema, not the partition cache — if a day is
missing, wait out the TTL or check that the export finished, rather than
refreshing.

Refreshing moves no data and still shows only verified partitions: an incomplete
export stays invisible afterwards, because it is still incomplete.

#### `DROP EXTERNAL TABLE` never deletes your data

```sql
DROP EXTERNAL TABLE pings;
-- External table 'pings' unregistered. No objects were deleted from storage.
```

The registration goes; **not one object under `LOCATION` is touched**. Re-running
`CREATE EXTERNAL TABLE` over the same prefix brings the table straight back.
This is the opposite of `DROP TABLE` on an ordinary table, which does destroy
the rows — worth being certain which one you are typing.

Deleting lake objects is a lifecycle-policy job for the object store, and is
deliberately not something SQL can do here: a `DROP` that silently emptied an S3
prefix would be an unrecoverable mistake one autocomplete away.

#### `SHOW EXTERNAL TABLES`

One row per registered table: `table_name`, `format`, `location`, `connection`,
`partition_columns` (rendered as `dt DATE`), `columns` (the **count** of
resolved columns — use `DESCRIBE` for their names) and `created_at`.

```sql
SHOW EXTERNAL TABLES;
SHOW EXTERNAL TABLES FROM lake LIKE 'ping%';
```

This is how you answer "what is this database reading out of object storage, and
with whose credentials" — an external table's reach is the reach of its
connection, so it is worth reading before granting anyone access. Ordinary
tables are not listed here; `SHOW TABLES` covers both.

#### Other refusals worth knowing

* The statement fails if the lake provider is not installed in the build
  ("external tables are not available in this build"). A missing provider is an
  **error**, never an empty result — an empty answer that should have been an
  error is the worst thing a database can return.
* A name already used by a **regular** table is refused even under
  `IF NOT EXISTS`: that flag means "leave the external table I already
  registered alone", not "accept that this name belongs to something else".

---

## Data Manipulation Language (DML)

```sql
INSERT INTO t [(c1, c2, ...)] VALUES (v1, v2, ...), ...;
INSERT INTO t [(c1, c2, ...)] SELECT ...;

UPDATE t SET c1 = v1, c2 = v2 [WHERE condition];

DELETE FROM t [WHERE condition];
```

**`rows_affected` is the number of rows actually written.** For
`INSERT ... SELECT` whose `SELECT` matches nothing, `rows_affected` is `0` — so a
client can detect a no-op. (Before **v1.14.3** one entry point reported `1` for
that case, because it counted the result envelope instead of the write; the
count is now consistent across every entry point.) The same rule holds for
`UPDATE` and `DELETE`: a statement whose `WHERE` matches no row reports `0`,
which is what makes a conditional `UPDATE` usable as an atomic claim.

**Worked example — populate a vector column from text using `EMBED`:**

```sql
UPDATE products SET description_vec = EMBED(description)
 WHERE description_vec IS NULL;
```

---

## MySQL compatibility (v1.10.0-ce)

The MySQL dialect that applications actually depend on runs unchanged, and `UNIQUE` / `PRIMARY KEY`
constraints are enforced (index-backed), not silently accepted.

**Upserts (F1)** — `ON DUPLICATE KEY UPDATE`, `INSERT IGNORE`, and `REPLACE`:

```sql
INSERT INTO inventory (sku, qty) VALUES ('A1', 5)
ON DUPLICATE KEY UPDATE qty = qty + 5;

INSERT IGNORE INTO inventory (sku, qty) VALUES ('A1', 1);   -- no-op on conflict
REPLACE INTO inventory (sku, qty) VALUES ('A1', 9);         -- delete-then-insert
```

**JSON (F2)** — path operators `->` (JSON) / `->>` (text) plus `JSON_EXTRACT`, `JSON_SET`,
`JSON_ARRAY`, `JSON_OBJECT`, `JSON_CONTAINS`:

```sql
SELECT data->>'$.email'                     AS email,
       JSON_EXTRACT(data, '$.plan')         AS plan
FROM   users
WHERE  JSON_CONTAINS(tags, '"vip"');
```

**ENUM (F3), auto-timestamp (F4), full-text (F5), type aliases (F6):**

```sql
CREATE TABLE tickets (
  id         INT PRIMARY KEY,
  status     ENUM('open','pending','closed'),          -- enforced as CHECK (status IN (...))
  note       VARCHAR(255),                              -- MySQL type aliases accepted
  updated_at TIMESTAMP ON UPDATE CURRENT_TIMESTAMP      -- refreshes on every UPDATE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE docs (id INT, body TEXT, FULLTEXT(body));
SELECT id FROM docs WHERE MATCH(body) AGAINST('quick fox');
```

Also mapped to the native format: `DATETIME`, `LONGBLOB`, `DECIMAL`, `DOUBLE`, `BIGINT`,
`VARCHAR(n)`, `utf8mb4` charset clauses, `ENGINE=InnoDB`.

**Constraint enforcement (F7)** — a duplicate on a `UNIQUE`/`PRIMARY KEY` column is rejected; a
`DELETE` frees the key so a re-insert succeeds (MySQL semantics). Enforcement is index-backed
(O(log n)), so it does not slow inserts at scale.

**mysqldump import (S1)** — the CLI can ingest a `mysqldump` file directly; oversized `INSERT`s are
chunked and MySQL escape / `DELIMITER` / `LOCK` / `SET` syntax is translated or skipped.

**MySQL wire-protocol front-end (S3)** — existing MySQL clients (DBeaver, Metabase, Connector/J,
mysql CLI, Node/PHP drivers) can connect directly over the MySQL wire protocol. It is **off by
default**; enable it under `[mysql_wire]` in the gateway config:

```toml
[mysql_wire]
enabled     = true
bind        = "127.0.0.1"
port        = 3307          # default is 3307 (NOT MySQL's 3306) so it can run alongside
                            # an existing MySQL/MariaDB on 3306; set port = 3306 for the classic port
require_tls = false         # when true, non-TLS clients are refused; the listener reuses the
                            # gateway HTTPS cert ([server] tls_cert_path / tls_key_path) and refuses
                            # to start if require_tls is true but no server cert is configured
```

Authenticate with your SynapCores **username** and **API key** (used as the MySQL password).
When `require_tls = false` (default) TLS is still *offered* whenever a server cert exists, so
clients may opt into encryption; `require_tls = true` makes it mandatory.

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

#### Qualified wildcard — `t.*` *(v1.15.0+)*

`SELECT t.*` selects every column of one relation. The qualifier may be a
table alias, a bare table name, or schema-qualified:

```sql
SELECT emp.* FROM emp;                       -- identical to SELECT * FROM emp
SELECT e.*   FROM emp e;                     -- by alias
SELECT app.emp.* FROM app.emp;               -- schema-qualified

-- what it exists for: one relation's columns out of a join
SELECT e.*
  FROM emp e
  JOIN dept d ON e.dept_id = d.id;

-- mixes with anything else
SELECT e.*, d.dname, 1 AS lit
  FROM emp e
  JOIN dept d ON e.dept_id = d.id;
```

Notes:

* In a join, `e.*` returns exactly the columns of `e` that `SELECT *` would
  return for that join, with the same names. A join qualifies its right-hand
  columns (`d.dname`), so `SELECT d.*` returns `d.id`, `d.dname` while
  `SELECT * FROM dept` returns `id`, `dname`.
* A qualifier that names no table or alias in the FROM clause is an error that
  names it (`Unknown qualifier 'x' in 'x.*'`), never a silent expansion. In a
  self-join, the shared table name is ambiguous — use the aliases
  (`SELECT a.* FROM t a JOIN t b ...`).
* Not supported with aggregates / `GROUP BY` (list the columns instead), and
  `COUNT(t.*)` is rejected — use `COUNT(*)` or `COUNT(t.<column>)`.

**Qualified wildcards** — `SELECT t.*` — are supported as of v1.15.0-ce, and are
the way to take one relation's columns out of a join:

```sql
SELECT o.*            FROM orders o JOIN customers c ON o.cust_id = c.id;
SELECT o.*, c.name    FROM orders o JOIN customers c ON o.cust_id = c.id;
```

The qualifier may be an alias, a table name, or a database-qualified name
(`analytics.orders.*`). Where a relation has an alias, either spelling resolves
to it. `t.*` returns exactly the subset of `SELECT *` belonging to `t`, in the
same order.

An unknown qualifier is an error naming what is in scope, and a qualifier
matching two relations (a self-join) is an ambiguity error asking you to use the
alias — neither silently guesses. `COUNT(t.*)` is rejected: it is not the same
aggregate as `COUNT(*)` once an outer join is involved, so it is refused rather
than answered with a different number.

Earlier versions rejected `t.*` outright with `Complex projections not supported`.

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

### `TRANSCRIBE(path [, language])` — *Enterprise only*

```sql
SELECT TRANSCRIBE('/data/media/call.wav')        AS text;
SELECT TRANSCRIBE('/data/media/call.wav', 'en')  AS text;
```

Speech-to-text over an audio file on the server, returning `TEXT`.

!!! warning "Not available in Community Edition"
    Community builds ship no speech-to-text engine. `TRANSCRIBE()` there returns
    an error stating that the Enterprise transcription feature is absent or that
    no Whisper model is configured under `[transcription]`. It fails loudly and
    never returns a partial or empty transcript.

    Before v1.15.0-ce the function was missing from the parser's function
    allowlist, so it answered `Unknown function: transcribe` on every edition.

### `MEMORY_STORE(namespace, content [, metadata [, options]])` — *v1.8.5+ (`options`: v1.8.9+)*

Stores text in an agent-memory namespace. On first call for a namespace the engine auto-creates the backing table `_memory_<namespace>` `(id TEXT PK, content TEXT NOT NULL, embedding VECTOR(384) NOT NULL, metadata TEXT, created_at TIMESTAMP, accessed_at TIMESTAMP)`. The content is embedded via the configured embedding model (default `all-minilm:latest`, dim 384) and a row is INSERTed under a sortable id (`mem_<base32 ts>_<6 alphanum>`).

* `namespace` MUST match `^[A-Za-z_][A-Za-z0-9_]*$`.
* `metadata` is optional and stored verbatim — convention: JSON-encoded with fields like `importance`, `kind`, `source`.
* `options` is an optional JSON object (build one with `json_object(...)`). The only key today is `dedup`.
* Returns: `TEXT` — the generated memory id.
* Per-tenant scoped via the standard storage path.

> **This is an unconditional insert.** Storing the same content twice creates two rows. Pass
> `json_object('dedup', true)` as the fourth argument to route the write through
> [`MEMORY_UPSERT`](#memory_upsertnamespace-content--options--natural_key--v189)'s `noop_if_equal`
> policy, which returns the existing row's id instead of duplicating it.

```sql
-- Store a user preference
SELECT MEMORY_STORE('default', 'I prefer Python over Java') AS memory_id;

-- Store with importance metadata
SELECT MEMORY_STORE('default', 'Customer renewed annual plan',
                    '{"importance": 0.9, "kind": "fact"}') AS memory_id;

-- Store without creating a duplicate row (v1.8.9+)
SELECT MEMORY_STORE('default', 'I prefer Python over Java',
                    NULL, json_object('dedup', true)) AS memory_id;
```

### `MEMORY_UPSERT(namespace, content [, options | natural_key])` — *v1.8.9+*

Idempotent, policy-driven write. Where `MEMORY_STORE` always inserts, `MEMORY_UPSERT` first looks
for the row this content should supersede, then applies a conflict-resolution policy.

* Returns: `TEXT` — the action taken, one of `'ADD'`, `'UPDATE'`, `'DELETE'`, `'NOOP'`.
* **Matching.** With a `key`, the row whose metadata carries the same `_key`. Without one, the row
  whose embedding is within `similarity_threshold` (default `0.95`) cosine of the new content —
  *semantic keying*, so an agent can revise a belief without having invented a key for it.
* **Third argument.** Either a bare `TEXT` natural key, or a JSON object built with `json_object(...)`.

| Option | Type | Default | Meaning |
|---|---|---|---|
| `key` | TEXT | — | Natural key. Absent ⇒ semantic keying. |
| `policy` | TEXT | `replace` | See below. |
| `confidence` | REAL `[0,1]` | `1.0` | Confidence of the incoming assertion. |
| `similarity_threshold` | REAL `[0,1]` | `0.95` | Cosine bar for semantic keying. |
| `metadata` | TEXT \| JSON | — | Merged into the stored metadata object. |
| `deleted` | BOOLEAN | `false` | Retract: delete the matched row. |
| `audit` | BOOLEAN | `true` | Emit a row into `_system_agent_memory_audit`. |

**Policies**

| Policy | Behavior |
|---|---|
| `replace` | Overwrite verbatim. `ADD` when absent, `UPDATE` when present. |
| `replace_higher_confidence` | Write only if the new confidence ≥ the stored one; otherwise `NOOP`. A low-confidence guess never clobbers a high-confidence fact. |
| `merge_max_confidence` | Always write, keeping `MAX(new, stored)` confidence. |
| `append_history` | Never overwrite — insert a superseding row with `_version = prev + 1`. The belief timeline is retained. |
| `noop_if_equal` | `NOOP` when the content is byte-identical; otherwise behaves as `replace`. |

**Retraction.** `deleted: true`, or `confidence: 0`, deletes the matched row and returns `'DELETE'`.
Retracting a belief that was never held is a `NOOP`, not an error.

**Reserved metadata keys.** Confidence, key and version live inside the `metadata` JSON object under
`_confidence`, `_key` and `_version` — so namespaces created by v1.8.5 `MEMORY_STORE` keep working
with no migration and no schema divergence.

**Audit.** Every non-`NOOP` call (and every confidence-declined `NOOP`) appends a row to
`_system_agent_memory_audit` `(audit_id, namespace, memory_id, natural_key, action, policy,
old_value, new_value, caller, timestamp)`. Audit writes are best-effort: an audit failure never
fails the write you asked for. Disable per-call with `audit: false`.

```sql
-- Revise a belief, keeping the higher-confidence one
SELECT MEMORY_UPSERT('default', 'User is pescatarian',
                     json_object('key', 'dietary',
                                 'policy', 'replace_higher_confidence',
                                 'confidence', 0.95)) AS action;
-- 'UPDATE'

-- Semantic dedup — no key needed
SELECT MEMORY_UPSERT('default', 'I prefer Python over Java') AS action;
-- 'NOOP' if a near-identical memory already exists, else 'ADD'

-- Natural key as a bare string
SELECT MEMORY_UPSERT('conv_42', 'Prefers email over phone', 'contact_pref') AS action;

-- Retract a fact the agent no longer holds
SELECT MEMORY_UPSERT('default', 'User is vegetarian',
                     json_object('key', 'dietary', 'deleted', true)) AS action;
-- 'DELETE'

-- Keep the full belief timeline instead of overwriting
SELECT MEMORY_UPSERT('default', 'Now on the enterprise plan',
                     json_object('key', 'plan', 'policy', 'append_history')) AS action;
-- 'ADD' (with _version bumped)

-- Audit what the agent changed its mind about
SELECT action, old_value, new_value, timestamp
  FROM _system_agent_memory_audit
 WHERE namespace = 'default'
 ORDER BY timestamp DESC;
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

### `AGENT_RUN(persona, task [, options])` — *v1.6.6.9+ (`options`: v1.8.9+)*

Runs a complete in-database agent loop — the ReAct pattern: reason → call tool → observe → repeat —
and returns the agent's final answer as `TEXT`. The agent sees the same per-tenant database as the
calling session, with the tools `execute_query`, `list_tables`, `describe_table` and `rag_search`.

Requires a tool-capable LLM configured in `[query.ai_service]` (e.g. `qwen2.5-coder:7b` via the
embedded `local` provider — the v1.8 default — or Ollama / OpenAI / Anthropic / Gemini).

* `persona` — the agent persona; `'aidb-assistant'` is the built-in default.
* `task` — a natural-language instruction.
* `options` — optional JSON object, built with `json_object(...)`:

| Option | Type | Default | Meaning |
|---|---|---|---|
| `model` | TEXT | persona / config | Override the model for this call (v1.8.10+). Outranks the persona and config default; an uninstalled name errors. |
| `max_iterations` | INT | `5` | ReAct tool-call cap. Clamped to `1..10`. |
| `timeout_ms` | INT | `600000` | Wall-clock budget. Clamped to `1000..600000`. |
| `allow_unknown_persona` | BOOL | `FALSE` | v1.14.3+. Run on the engine's `default` persona when `persona` does not exist, instead of erroring. |

Out-of-range numbers are **clamped**, not rejected — failing a long agentic query on a knob is worse
than capping it. Unknown option keys **are** an error, so typos surface immediately.

> **The persona must exist — v1.14.3 breaking change.** `AGENT_RUN('does-not-exist', …)` now fails
> with `AGENT_RUN: unknown persona '<name>'. Available: …`, matching what `CREATE AGENT` has done
> since v1.14.2. Earlier builds logged a warning and silently ran the request on the built-in
> `default` persona — which is *conversational*, so an unattended agent answered in prose instead of
> calling its tools and returned plausible-looking wrong output. Create the persona
> (`CREATE PERSONA`, below), name a built-in, or pass
> `json_object('allow_unknown_persona', true)` to opt back into the fallback deliberately.

> **`allow_writes` is rejected — the agent's database tools are always read-only from SQL.** A
> per-query flag would let any authenticated caller escalate a read-only agent into a writing one.
> Write capability is declared once by an operator on a durable agent (`CREATE AGENT`, v1.9.0), never
> per query. `allow_unknown_persona` is different: it grants no capability, so SQL may set it.

Returns `NULL` only when no agent service is wired (i.e. the AI service is not configured); a *run*
that fails (the ReAct loop errors, the model times out) returns a `TEXT` value prefixed
`AGENT_RUN error:` so it survives `COALESCE` in a procedure body while remaining greppable. A bad
**argument** — a non-string persona or task, an unknown option key, or (v1.14.3+) a persona that does
not exist — fails the statement instead: it is a mistake in the SQL, and burying it in a result cell
is exactly the silent degrade v1.14.3 removed.

```sql
-- Aggregation + reasoning
SELECT AGENT_RUN('aidb-assistant',
  'Execute SELECT category, SUM(price*stock) FROM products GROUP BY category,
   then tell me which category has the highest total') AS reply;

-- Bound a long agentic query
SELECT AGENT_RUN('aidb-assistant', 'Investigate the slowest query',
                 json_object('max_iterations', 3, 'timeout_ms', 60000)) AS reply;

-- Composed in a CTE. 'returns-triage' is NOT a built-in: create it first
-- (CREATE PERSONA, below), or this statement errors with
-- "AGENT_RUN: unknown persona 'returns-triage'" (v1.14.3+).
WITH triaged AS (
  SELECT order_id,
         AGENT_RUN('returns-triage',
                   'Process return for order_id ' || CAST(order_id AS TEXT)) AS recommendation
    FROM pending_returns
)
SELECT * FROM triaged;
```

---


### `TRANSCRIBE(path [, language])` — *Enterprise only*

```sql
SELECT TRANSCRIBE('/data/media/call.wav')       AS text;
SELECT TRANSCRIBE('/data/media/call.wav', 'en') AS text;
```

Speech-to-text over an audio file on the server, returning `TEXT`.

**Community builds have no speech-to-text engine.** The call returns an explicit
error saying the Enterprise transcription feature is absent, or that no Whisper
model is configured under `[transcription]`. It fails loudly and never returns an
empty or partial transcript.

Before v1.15.0-ce the function was missing from the parser's function allowlist,
so it answered `Unknown function: transcribe` on every edition — including builds
that had the engine.

## Personas (v1.14.2+)

A persona is the reasoning profile — system prompt, optional pinned model, output shape — that
`AGENT_RUN(persona, task)` and `CREATE AGENT ... PERSONA '<name>'` bind to. Since v1.14.2 a persona is
a first-class database object with its own DDL instead of a config-file entry: a persona created at
runtime is usable immediately and survives restart. Personas are install-wide (not tenant-scoped).

Eight personas are built in and seeded on first boot — `default`, `sql_developer`, `data_scientist`,
`rust_developer`, `devops_engineer`, `marketing_expert`, `ui_ux_designer`, and `aidb-assistant` (the
one the AI-chat UI and the `AGENT_RUN` examples above use). A built-in may be edited, but it stays
built-in and cannot be dropped. `SHOW PERSONAS` lists exactly what this engine has.

Any other name — including every persona in an example you copy from a blog post or a recipe — must
be created first. Both surfaces **reject** an unknown persona with `unknown persona '<name>'`:
`CREATE AGENT` since v1.14.2, and `AGENT_RUN` since v1.14.3. Each offers the same deliberate escape
hatch — `WITH (allow_unknown_persona = TRUE)` on `CREATE AGENT`,
`json_object('allow_unknown_persona', true)` in `AGENT_RUN`'s options — which restores the old
behaviour of running on the built-in `default` persona.

> **Upgrading from v1.14.2 or earlier:** an `AGENT_RUN` call naming a persona nobody created used to
> succeed with a warning in the log. It now errors. That is the point: the `default` persona is
> conversational, and substituting it for a task-shaped one produced answers that read fine and did
> not do the work.

### `CREATE PERSONA` / `CREATE OR REPLACE PERSONA`

```sql
CREATE [OR REPLACE] PERSONA <name> WITH ( key = value [, ...] );
```

| Key | Type | Default | Meaning |
|---|---|---|---|
| `system_prompt` | TEXT | *(required)* | The persona's instructions. |
| `display_name` | TEXT | the persona name | Human-facing label. |
| `description` | TEXT | `''` | Free-text note. |
| `model` | TEXT | *(none)* | Pin a model. Must already be installed. Omit to inherit `[query.ai_service]`. |
| `response_type` | TEXT | `'text'` | `'text'`, `'markdown'`, `'json'` or `'code'`. |
| `tool_enabled` | BOOL | `TRUE` | Whether runs on this persona may call tools. |

`<name>` is a SQL identifier. Use a **delimited** (double-quoted) identifier when the name carries a
character a bare identifier cannot — the quotes are not part of the stored name, and a doubled `""`
inside means one literal `"`:

```sql
CREATE PERSONA "aidb-assistant" WITH ( system_prompt = 'You are a database task executor.' );
DESCRIBE PERSONA "aidb-assistant";      -- resolves to the persona named aidb-assistant
```

> **Fixed in v1.14.2.1.** Earlier builds stored the quotes verbatim, so `CREATE PERSONA "x-y"` created
> a persona literally named `"x-y"` that no later statement could address.

```sql
-- Minimal
CREATE PERSONA retention_analyst
  WITH ( system_prompt = 'You are a retention analyst. Be concise and cite numbers.' );

-- Full, with a pinned model
CREATE PERSONA compliance_reviewer WITH (
  display_name  = 'Compliance Reviewer',
  description   = 'Reviews transactions against policy',
  system_prompt = 'You are a compliance reviewer. Quote the policy clause you rely on.',
  model         = 'qwen2.5-coder:7b',
  response_type = 'json',
  tool_enabled  = TRUE
);
```

An unknown option key is rejected at parse time, and `system_prompt` is required — `CREATE PERSONA p
WITH ( display_name = 'P' )` errors with `CREATE PERSONA requires system_prompt = '<string>'`.
Creating a name that already exists errors unless `OR REPLACE` is given; `OR REPLACE` is a **full
rewrite** (keys you omit revert to their defaults) that preserves `created_at` and the built-in flag.

Commas inside a quoted value are safe, and a literal single quote is escaped by doubling it:

```sql
CREATE OR REPLACE PERSONA docs_writer WITH (
  system_prompt = 'First, do no harm. Use the user''s schema.',
  description   = 'Writes SQL, only SQL'
);
```

> **A pinned model must be installed (v1.14.2).** `model = 'typo-not-installed:7b'` is rejected at DDL
> time with the list of installed models and a pointer to `PULL_MODEL('...')`, instead of failing
> later inside an agent loop. The check is skipped only when no model registry can be resolved.

### `ALTER PERSONA`

```sql
ALTER PERSONA <name> SET ( key = value [, ...] );
```

Edits only the keys named (the same six keys as `CREATE PERSONA`; at least one is required). Setting
`model` to the empty string clears the pin and restores "inherit the configured default".

```sql
ALTER PERSONA retention_analyst SET ( system_prompt = 'Answer in at most three sentences.' );
ALTER PERSONA retention_analyst SET ( model = 'qwen2.5-coder:7b', response_type = 'json' );
ALTER PERSONA retention_analyst SET ( model = '' );          -- back to the config default
ALTER PERSONA retention_analyst SET ( tool_enabled = false );
```

### `DROP PERSONA`

```sql
DROP PERSONA [IF EXISTS] <name> [CASCADE];
```

`IF EXISTS` makes the drop idempotent (`Persona 'x' does not exist, skipping (IF EXISTS)`). A built-in
persona cannot be dropped (`cannot drop built-in persona 'sql_developer'`).

Since **v1.14.2.1** the drop is also refused while a durable agent still binds to the persona, so a
`DROP` can no longer silently orphan an agent:

```
Cannot drop persona 'retention_analyst' because 2 agent(s) depend on it:
acme/churn_watcher, acme/renewal_bot. Repoint or drop those agents first, or use
DROP PERSONA retention_analyst CASCADE to drop it anyway ...
```

Dependents are listed as `tenant/agent`, and the scan covers **every** tenant (personas are
install-wide). Repoint or drop those agents first — a new `CREATE AGENT` naming an unknown persona is
refused too. If the agent store cannot be read at all, the check is skipped rather than blocking the
drop.

`CASCADE` forces the drop. It does **not** delete the dependent agents; it only waives the refusal,
and they are left with a dangling binding that `DESCRIBE AGENT` reports as `persona_resolved = false`:

```sql
DROP PERSONA retention_analyst CASCADE;
-- Persona 'retention_analyst' dropped (CASCADE) — 2 agent(s) now have a dangling
-- persona binding: acme/churn_watcher, acme/renewal_bot
```

### `SHOW PERSONAS` / `DESCRIBE PERSONA`

```sql
SHOW PERSONAS;                         -- persona_name, display_name, model, tool_enabled, is_builtin
SHOW PERSONAS LIKE 'retention%';
SHOW PERSONAS LIKE '%gate%';           -- substring match needs wildcards on both sides
DESCRIBE PERSONA retention_analyst;    -- (property, value) rows, including the full system_prompt
DESCRIBE PERSONA "aidb-assistant";     -- delimited identifier for a hyphenated name
```

`model` comes back `NULL` in `SHOW PERSONAS` (and `(config default)` in `DESCRIBE PERSONA`) when no
model is pinned.

The `LIKE` pattern is standard, **anchored** SQL `LIKE` (v1.14.2.1): `%` matches any run of
characters, `_` matches exactly one, every other character is a literal, and the whole
`persona_name` must match. `LIKE 'gate'` therefore matches only a persona named exactly `gate` —
use `'%gate%'` to find `docgate_analyst`. Matching is case-sensitive.

> **Changed in v1.14.2.1.** In v1.14.2 the pattern was compiled into an *unanchored* regex, so
> `LIKE 'gate'` matched `docgate_analyst` and a pattern containing regex metacharacters (`.`, `(`,
> `[`) was silently treated as a regex.

---

## Persistent agent memory (v1.14.3+)

`CREATE MEMORY` gives an agent memory that survives the conversation: episodes, durable facts,
temporal validity, relationships, provenance, and an authoritative current state — as **one**
logical object.

The rule that shapes everything below: **the engine owns the lifecycle.** An agent expresses
intent (`REMEMBER`, `RECALL`); classification, extraction, deduplication, temporal reconciliation,
relationship creation, provenance, indexing, consolidation, retention and cleanup are the
database's job. You never call `CONSOLIDATE` to keep memory correct.

### Creating a memory object

```sql
CREATE MEMORY customer_memory
FOR AGENT support_agent
IDENTITY user_id
WITH (
    episodic = true,
    semantic = true,
    temporal = true,
    relationships = true,
    provenance = true,
    consolidation = 'AUTO',
    consolidate_on_session_end = true,
    consolidate_after_episodes = 50,
    retain_raw_episodes = true
);
```

Every option has a default, so `CREATE MEMORY customer_memory IDENTITY user_id;` is a complete,
working object. `IDENTITY` is required: it names the field that scopes every read and write, and
memory never crosses identities.

The object materializes five ordinary tables, so nothing is hidden from you:

| Table | Holds |
|---|---|
| `_mem_<name>_episodes` | raw statements — the source everything else derives from |
| `_mem_<name>_facts`    | durable facts and attribute claims, with validity intervals |
| `_mem_<name>_state`    | the authoritative current value per (identity, attribute) |
| `_mem_<name>_prov`     | derivation lineage |
| `_mem_<name>_rels`     | typed relationships (also mirrored into the tenant graph) |

```sql
SELECT attribute, value, valid_from FROM _mem_customer_memory_state WHERE identity = '123';
```

### The five verbs you actually use

| Verb | Purpose |
|---|---|
| `REMEMBER` | persist an observation; the engine derives everything from it |
| `RECALL`   | assemble the context relevant to a question |
| `CURRENT`  | read the authoritative present value of one attribute |
| `FORGET`   | explicit, authorized removal |
| `TRACE`    | explain why a value is believed |

Everything else — `CONSOLIDATE`, `RECONCILE`, `SUPERSEDE`, `HISTORY`, `SEARCH`, `RELATE`,
`EXPLAIN MEMORY` — is optional control, never a prerequisite.

### REMEMBER

```sql
REMEMBER customer_memory
FOR user_id = 123
'We completed the production migration from Azure to AWS yesterday because the engineering team wants Bedrock integration.';
```

One statement persists the raw episode and then derives, without any further call: the semantic
facts, the current-state change, the closed validity interval on the old value, the relationships,
and the provenance linking each derived record back to this sentence.

The response is a JSON envelope in the `envelope` column:

```json
{
  "status": "accepted",
  "operation": "REMEMBER",
  "memory": "customer_memory",
  "identity": {"field": "user_id", "value": "123"},
  "request_id": "req_91ac",
  "data": {
    "episode_id": "ep_89232",
    "memory_ids": ["mem_1042", "mem_1043"],
    "changes": {"episodes_created": 1, "facts_created": 1, "state_changes": 1,
                "relationships_created": 1, "records_superseded": 1},
    "consolidation": {"required": true, "status": "pending"}
  },
  "warnings": [],
  "error": null
}
```

`consolidation.status = "pending"` never means the episode was lost — the raw episode is committed
*before* any derivation runs.

**Works with no model configured.** A deterministic extractor always runs; the LLM pass is
additive and degrades to the rule result on any failure. When you want certainty, state the change
outright:

```sql
REMEMBER customer_memory FOR user_id = 123 'Now on AWS'
  WITH (attribute = 'cloud_provider', value = 'AWS');
```

Full option list: `source_type`, `source_id`, `confidence`, `event_time`,
`session_id`, `session_end`, `attribute`, `value`, `extract`.

`source_type` sets the claim's authority, which is the first dimension of conflict resolution:

```
system_of_record > verified_user_statement > user_statement > tool_result
                 > prior_user_statement > agent_derivation > agent_inference
```

### RECALL

```sql
RECALL customer_memory FOR user_id = 123 ABOUT 'customer deployment environment';
```

Returns current state **separately** from facts, episodes, relationships, temporal transitions and
provenance, so a model never has to guess which similar-looking string is authoritative:

```json
{
  "current_state": {"cloud_provider": {"value": "AWS", "confidence": 0.98,
                                       "valid_from": "2026-08-15 14:32:00"}},
  "semantic":  [{"memory_id": "mem_1043", "fact": "Customer prefers managed cloud services",
                 "confidence": 0.91, "relevance": 0.77}],
  "episodic":  [{"episode_id": "ep_89232", "event": "...", "relevance": 0.95}],
  "relationships": [{"subject": "Customer", "predicate": "USES", "object": "AWS"}],
  "temporal":  [{"attribute": "cloud_provider", "previous_value": "Azure",
                 "current_value": "AWS", "changed_at": "2026-08-15 14:32:00"}],
  "context":   {"prompt_ready": "Cloud provider is AWS. ...",
                "estimated_tokens": 31, "truncated": false}
}
```

Two properties worth relying on:

* **Authority beats similarity.** "What cloud do we use now?" returns AWS ahead of the
  semantically similar older Azure episode. "What did we use before AWS?" routes to temporal
  history instead.
* **`confidence` and `relevance` are different fields.** Confidence is evidence strength;
  relevance is retrieval score. A highly relevant historical statement can be non-authoritative.

Options: `current`, `history`, `episodes`, `facts`, `relationships`, `provenance`, `temporal`,
`token_budget`, `prompt_ready`, `as_of`, `min_confidence`, `explain`.

```sql
RECALL customer_memory FOR user_id = 123 ABOUT 'production infrastructure'
  WITH (current = true, history = 3, provenance = true);
```

When the token budget is exceeded, lower-priority context is truncated and `context.truncated`
becomes `true` — but current state is preserved.

### CURRENT

```sql
CURRENT customer_memory FOR user_id = 123 ATTRIBUTE subscription_plan;
```

Deterministic state lookup, never semantic search. Three outcomes:

* `status = "ok"` — one accepted value, with `valid_from`, `confidence`, `authority`, and the
  source record.
* `status = "not_found"` — nothing recorded; `data.value` is `null`.
* `status = "conflict"` — competing authoritative claims that authority, temporal validity and
  confidence all failed to separate. Both claims are returned in `data.competing_claims`. **The
  engine does not pick.** Resolve with `RECONCILE` or a higher-authority `REMEMBER`.

### TRACE

```sql
TRACE customer_memory FOR user_id = 123 ATTRIBUTE cloud_provider;
```

Returns the source episode and its verbatim content, when it was recorded versus when the event
happened, the extraction method and version, the confidence, the derivation steps, and the record
this value superseded.

### FORGET

```sql
FORGET customer_memory FOR user_id = 123 ABOUT 'home address';
```

Explicit removal only — for a user request, a retention obligation, or an administrative
correction. **It is not the cleanup mechanism**: expiry, deduplication, compaction and
consolidation all happen automatically without it. Removal propagates to derived facts,
embeddings, relationships, current-state values and provenance, and the response reports the count
per category. `WITH (mode = 'suppress')` makes records inaccessible instead of deleting them.

### Consolidation is the engine's job

The engine runs consolidation on its own schedule — on an episode-count threshold, a session
boundary, elapsed age, or an idle window — with no LLM prompt, agent tool call, or application
callback involved. A consolidation cycle retries failed extractions, collapses duplicate facts,
re-checks contradictions, applies retention, and advances a per-identity watermark. It is
idempotent: running it twice over an unchanged set of episodes produces no new facts.

Watch it work:

```sql
SELECT memory_name, identity, pending_episodes, last_trigger, next_eligible, last_run_at
FROM _system_memory_consolidation;
```

You *may* force a cycle — before a report, in a test, after a bulk import:

```sql
CONSOLIDATE customer_memory FOR user_id = 123;
```

but you never *have to*. If you find yourself calling it to make an answer correct, that is a bug
worth reporting.

### Giving a durable agent memory

```sql
CREATE AGENT support_bot
PERSONA 'support_agent'
TASK 'Answer the customer''s question.'
WITH (
    memory_object   = 'customer_memory',
    memory_identity = '123',
    memory_auto_recall   = true,   -- default
    memory_auto_remember = true    -- default
);
```

With this binding the runtime performs a `RECALL` **before** inference — so memory is available
while the model reasons, not after it has answered — and a `REMEMBER` after the turn. Neither is a
tool the model can forget to call, and the identity is fixed in the definition, so the model
cannot address another user's memory. Exposing `REMEMBER`/`RECALL` as tools is still supported
when an agent needs discretionary control; the two paths reach the same engine.

`memory_object` and `memory_identity` must be given together — a bound object with no identity has
no scope to operate in.

### Conflicts

Two contradictory claims coexist as claims until policy resolves them; evidence is never
overwritten. Resolution considers, in order: explicit correction language, source authority,
temporal validity, then a meaningful confidence margin. When none of them separates the claims the
attribute is marked conflicted and both are exposed.

```sql
-- Both statements are user statements, both dated the same, and they disagree:
CURRENT customer_memory FOR user_id = 123 ATTRIBUTE cloud_provider;
-- => status "conflict", data.competing_claims = [{"value": "Azure", ...}, {"value": "AWS", ...}]

RECONCILE customer_memory FOR user_id = 123 ATTRIBUTE cloud_provider WITH (prefer = 'AWS');
```

Set `conflict_policy = 'ALWAYS_FLAG'` to require a human for every contradiction, or
`'LATEST_WINS'` / `'HIGHEST_CONFIDENCE'` for simpler deployments.

### Identity isolation

Memory reads and writes never cross identities. Over REST, a credential whose token carries a
`memory_identity` claim is *pinned*: a request naming any other identity is refused with
`MEMORY_ACCESS_DENIED` before any SQL is composed.

**An ordinary interactive login is pinned to its own user automatically.** A client can then send
no identity at all and have it resolved from the token — it cannot address another identity
because it never names one.

On a default CE install `/v1/auth/register` also gives each user its own tenant, and memory
objects are tenant-scoped, so that is a second boundary already. The pin matters for the
configuration the licensing model describes — one tenant with several users
(`MAX_TENANTS = 1`) — where the identity is the only boundary left.

Admins and API keys stay **unpinned**, deliberately: an admin needs to inspect or `FORGET` on a
user's behalf, and a service credential serving many end users supplies the identity per request
(the application-server pattern). If you mint your own JWTs, set `memory_identity` yourself.

### REST and MCP

The same contract is available over HTTP at `/v1/memory/*` and as the MCP tools
`memory_remember`, `memory_recall`, `memory_current`, `memory_forget`, `memory_trace`. All three
surfaces return the identical envelope, so a client never has to parse prose to learn what
happened. The maintenance verbs are deliberately absent from MCP: memory correctness must not
depend on an agent remembering to invoke them.

```bash
curl -sS -X POST localhost:8080/v1/memory/customer_memory/recall \
  -H 'Authorization: Bearer <token>' -H 'Content-Type: application/json' \
  -d '{"identity":"123","about":"production cloud"}'
```

---

## Durable-agent activation control (v1.14.2+)

Durable agents (`CREATE AGENT`, v1.9.0+) fire on a cron schedule and/or on committed DML events.
v1.14.2 adds two ways to drive them from a pipeline.

### `WAKE AGENT` — fire now, asynchronously

```sql
WAKE AGENT <name>;
```

Enqueues an immediate run on the background dispatcher and returns at once
(`Agent 'stage2_summarizer' woken (queued)`) — the LLM loop never runs inside your request. This is
the async counterpart to `EXECUTE AGENT`, which runs the loop synchronously and returns its output.
Use it to hand off between pipeline stages without waiting for a cron tick.

The agent must exist and be **enabled**; waking a disabled agent errors with
`Agent '<name>' is disabled (ALTER AGENT <name> ENABLE to wake it)`. The queued run shows up in
`_system_agent_runs` with `activation = schedule 'manual-wake'` once the dispatcher drains it:

```sql
WAKE AGENT stage2_summarizer;

SELECT started_at, status, output
FROM _system_agent_runs
WHERE agent_name = 'stage2_summarizer'
LIMIT 5;
```

A user-issued wake is enqueued at cascade hop 0 — it starts a fresh chain rather than continuing one.

### `ALLOW AGENT ORIGIN` — let one agent's write fire another

```sql
CREATE AGENT <name> PERSONA '<persona>' TASK '<task>'
  ON {INSERT|UPDATE|DELETE} {INTO|ON|FROM} <table> [WHERE <expr>] ALLOW AGENT ORIGIN
  [WITH ( ... )];
```

This is a **suffix on an event binding**, not a statement of its own. By default a write made inside
an agent's run never re-fires any event binding — the cascade suppression that stops an
`allow_writes` agent from looping on its own `INSERT`. `ALLOW AGENT ORIGIN` opts that one binding in,
so agents can be chained into a pipeline:

```sql
-- Stage 2 fires when stage 1's agent marks a row VALIDATED
CREATE AGENT stage2_summarizer
  PERSONA 'retention_analyst'
  TASK 'Summarize the validated row.'
  ON UPDATE ON doc_queue WHERE stage = 'VALIDATED' ALLOW AGENT ORIGIN
  WITH ( max_iterations = 2, timeout_seconds = 120 );
```

Two guards always remain in force: a binding never fires on **its own** agent's write (no self-loop),
and a chain is capped at **5 hops**, so even a mis-declared `A → B → A` cycle halts. Ordinary user
writes fire the binding at hop 0 whether or not the suffix is present.

Place the suffix after the optional `WHERE`; the predicate is cut at the suffix, so the binding above
stores its predicate as `stage = 'VALIDATED'`.

**Reading the opt-in back (v1.14.2.1+).** The flag is auditable, not write-only: both `DESCRIBE AGENT`
and `SHOW AGENTS` print the suffix on every binding that carries it, spelled exactly as it is
declared, so the readback can be pasted straight back into DDL. A binding without the opt-in prints
no suffix.

```sql
DESCRIBE AGENT stage2_summarizer;
-- property   | value
-- Bindings   | UPDATE ON doc_queue WHERE stage = 'VALIDATED' ALLOW AGENT ORIGIN

SHOW AGENTS;
-- name               | persona            | bindings                              | state   | tokens_today
-- stage2_summarizer  | retention_analyst  | UPDATE ON doc_queue ALLOW AGENT ORIGIN | enabled | 0
```

(The `SHOW AGENTS` summary stays compact — it omits the `WHERE` predicate but still shows the
cascade opt-in. Use `DESCRIBE AGENT` for the full binding.)

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

## Operator configuration

These are gateway (`community.toml`) settings and CLI, not SQL — included here because they
feed docs.synapcores.com and affect how the engine runs.

### Timeouts for in-process LLM operations (v1.12.0+)

The shipped config defaults were raised so first-token cold starts on in-process (native GGUF)
models don't trip a request timeout:

```toml
[server]
request_timeout = 300        # seconds (was 30)

[query]
default_timeout_ms = 300000  # was 30000
```

### Anonymous product telemetry (v1.12.0+, opt-out)

SynapCores sends a small amount of **anonymous** usage telemetry to measure the install
footprint. What is sent: a random installation id (a UUID with no link to you), the product
version + edition, and the deployment / OS / architecture. What is **never** sent: SQL, database
or table names, schemas, prompts, embeddings, credentials, hostnames, usernames, IP addresses,
or any row / customer data. It runs on a detached background task and can never slow down or
affect the database.

```toml
[telemetry]
enabled  = true
endpoint = "https://telemetry.synapcores.com"
# heartbeat_hours = 24   # optional cadence override (default 24h)
```

Turn it off in any of these ways:

- set `[telemetry] enabled = false`, or
- run `synapcores telemetry disable`, or
- set the environment variable `DO_NOT_TRACK=1`, or
- set the environment variable `SYNAPCORES_TELEMETRY=off`.

Operator CLI (`synapcores telemetry <sub>`):

```bash
synapcores telemetry status                  # enabled?, installation id, endpoint, effective state + reason
synapcores telemetry preview                 # print the EXACT outbound heartbeat JSON (no send)
synapcores telemetry test --send             # send one event now and report success/failure
synapcores telemetry enable                  # persist an explicit choice (overrides the config flag)
synapcores telemetry disable
synapcores telemetry reset-installation-id   # mint a fresh installation id on next boot
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
