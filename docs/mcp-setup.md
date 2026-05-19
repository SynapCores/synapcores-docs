# Using SynapCores AIDB from MCP-compatible clients

**Released**: v1.6.5-ce (2026-05-16); 14 tools as of v1.6.5.1-ce; gateway wire-fixes in v1.6.5.2-ce
**Verified against**: Claude Desktop 1.x, Claude Code 1.x, Cursor 0.45+, Continue.dev 0.9+, Windsurf, Zed, LangChain MCP adapter — both macOS and Linux

<video controls width="100%" style="max-width: 800px" preload="metadata">
  <source src="assets/mcp-demo.mp4" type="video/mp4">
  Your browser doesn't support inline video. <a href="assets/mcp-demo.mp4">Download the demo</a>.
</video>

*45-second demo: setting up the bridge and asking Claude Code to find at-risk loyalty members + draft a personalized retention message via the synapcores MCP. Six MCP tools, seven seconds end-to-end.*

SynapCores AIDB ships a built-in MCP (Model Context Protocol) server inside
the gateway. Once configured, any MCP-compatible client can:

- Discover your schema (`list_tables`, `describe_table`)
- Read data (`query` — any SELECT)
- Modify data (`execute` — INSERT/UPDATE/DELETE/DDL)
- Validate SQL without running it (`validate_query`)
- Look up SynapCores SQL syntax (`sql_manual`)
- **Train AutoML models** (`train_model`) and **score** them (`predict`)
- **Inspect models** (`list_models`, `describe_model`)
- **Semantic search** vector collections without writing `EMBED()` / `COSINE_SIMILARITY()` SQL (`semantic_search`)
- **Generate embeddings** for arbitrary text (`embed_text`)
- **Query the property graph** in Cypher (`graph_query`)
- **Generate text** from the configured LLM (`generate_text`)

All tool calls go through the gateway's normal auth + tenant-isolation
layer, so the client only sees the data — and only operates on the models,
vectors, and graph — that the connecting user is allowed to see.

---

## How it works

The gateway exposes MCP as plain JSON-RPC 2.0 over HTTP:

```
POST /v1/mcp          — single JSON-RPC request
POST /v1/mcp/batch    — batch of requests
GET  /v1/mcp/info     — server info (also requires auth)
```

Most MCP clients (Claude Desktop, Claude Code, Cursor, Continue.dev, Windsurf, Zed) expect MCP over **stdio** (or SSE / streamable HTTP — but not raw JSON-RPC POST). So we ship a small **stdio→HTTP bridge** at `scripts/integrations/synapcores-mcp-bridge.js` that translates one to the other. ~100 lines of Node, no npm deps.

> **Note (transitional):** in v1.6.5.x the gateway exposes only plain
> JSON-RPC POST. The bridge wallpapers over the missing SSE /
> Streamable HTTP transports. **v1.6.6 will add both transports** so
> clients can connect to the gateway URL directly without any local
> process. The bridge will remain in the repo as a fallback for
> older clients.

```
  MCP client (stdio JSON-RPC)
        ▼
  synapcores-mcp-bridge.js  (node, on your machine)
        ▼ POST /v1/mcp + JWT
  SynapCores gateway
        ▼
  AIDB tools (query / execute / schema / …)
```

---

## Prerequisites

1. **SynapCores CE running** — `curl -fsSL https://get.synapcores.com/install.sh | sh`,
   or you already have a gateway up.
2. **Reachable endpoint** — note the URL (e.g. `http://127.0.0.1:8080`
   for a local install, `https://aidb.example.com` for a hosted one).
3. **An account** — username + password OR a long-lived JWT / API key.
   The default admin password is printed once at first boot; copy it,
   or pass `AIDB_ADMIN_PASSWORD=...` to the install for a known value.
4. **Node.js 18+** on the machine running the MCP client (the client
   bundles nothing of its own; the bridge is a regular Node script).

---

## Step 1 — Get the bridge

```bash
mkdir -p ~/.synapcores
curl -fsSL \
  https://raw.githubusercontent.com/mataluis2k/aidb/feature/v1.5.0-ce/scripts/integrations/synapcores-mcp-bridge.js \
  -o ~/.synapcores/synapcores-mcp-bridge.js
chmod +x ~/.synapcores/synapcores-mcp-bridge.js
```

(Or copy `scripts/integrations/synapcores-mcp-bridge.js` from a clone
of the repo to anywhere you like; the absolute path is what your client
needs.)

---

## Step 2 — Configure your client

All clients share the same config shape: a stdio command + args + env. Pick yours.

### Option A: Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`
on macOS, `%APPDATA%\Claude\claude_desktop_config.json` on Windows, or
`~/.config/Claude/claude_desktop_config.json` on Linux. Add:

```json
{
  "mcpServers": {
    "synapcores": {
      "command": "node",
      "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
      "env": {
        "SYNAPCORES_URL": "http://127.0.0.1:8080",
        "SYNAPCORES_USERNAME": "admin",
        "SYNAPCORES_PASSWORD": "your-admin-password-here"
      }
    }
  }
}
```

Restart Claude Desktop. Open a new conversation; the MCP server icon
in the bottom-left of the chat composer should show "synapcores" as
connected, with **14 tools** available.

### Option B: Claude Code (CLI)

One command, no JSON editing:

```bash
claude mcp add synapcores \
  --transport stdio \
  --env SYNAPCORES_URL=http://127.0.0.1:8080 \
  --env SYNAPCORES_USERNAME=admin \
  --env SYNAPCORES_PASSWORD='your-admin-password-here' \
  -- node ~/.synapcores/synapcores-mcp-bridge.js
```

Verify it registered:

```bash
claude mcp list
# synapcores  - node ~/.synapcores/synapcores-mcp-bridge.js   ✓ connected
```

### Option C: Cursor

Edit `~/.cursor/mcp.json` (create the file if it doesn't exist):

```json
{
  "mcpServers": {
    "synapcores": {
      "command": "node",
      "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
      "env": {
        "SYNAPCORES_URL": "http://127.0.0.1:8080",
        "SYNAPCORES_USERNAME": "admin",
        "SYNAPCORES_PASSWORD": "your-admin-password-here"
      }
    }
  }
}
```

Restart Cursor. Open the **Composer** panel — the synapcores tools appear in the tool palette. (Cursor 0.45+ required for MCP support.)

### Option D: Continue.dev (VS Code extension)

Edit `~/.continue/config.json` and add the `mcpServers` block under the root object:

```json
{
  "models": [ ... your existing model config ... ],
  "mcpServers": {
    "synapcores": {
      "command": "node",
      "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
      "env": {
        "SYNAPCORES_URL": "http://127.0.0.1:8080",
        "SYNAPCORES_USERNAME": "admin",
        "SYNAPCORES_PASSWORD": "your-admin-password-here"
      }
    }
  }
}
```

Reload the VS Code window (Cmd+Shift+P → "Developer: Reload Window"). Continue picks up the new server automatically.

### Option E: Windsurf (Codeium)

Edit `~/.codeium/windsurf/mcp_config.json` (create if missing):

```json
{
  "mcpServers": {
    "synapcores": {
      "command": "node",
      "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
      "env": {
        "SYNAPCORES_URL": "http://127.0.0.1:8080",
        "SYNAPCORES_USERNAME": "admin",
        "SYNAPCORES_PASSWORD": "your-admin-password-here"
      }
    }
  }
}
```

Restart Windsurf. The synapcores server shows up under Cascade's MCP tool list.

### Option F: Zed editor

Edit `~/.config/zed/settings.json` and add under the `experimental` block:

```json
{
  "experimental": {
    "mcp_servers": {
      "synapcores": {
        "command": "node",
        "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
        "env": {
          "SYNAPCORES_URL": "http://127.0.0.1:8080",
          "SYNAPCORES_USERNAME": "admin",
          "SYNAPCORES_PASSWORD": "your-admin-password-here"
        }
      }
    }
  }
}
```

Restart Zed. The tools appear in the assistant panel's tool list.

### Option G: LangChain MCP adapter (Python)

```bash
pip install langchain-mcp-adapters
```

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

client = MultiServerMCPClient({
    "synapcores": {
        "command": "node",
        "args": ["/Users/YOU/.synapcores/synapcores-mcp-bridge.js"],
        "env": {
            "SYNAPCORES_URL": "http://127.0.0.1:8080",
            "SYNAPCORES_TOKEN": "aidb_your_long_lived_token_here",
        },
        "transport": "stdio",
    }
})

tools = await client.get_tools()

# Pass to any LangChain / LangGraph agent
agent = create_react_agent(ChatOpenAI(model="gpt-4o"), tools)
response = await agent.ainvoke({"messages": "List the tables in synapcores"})
```

For long-running services prefer the `SYNAPCORES_TOKEN` env var (a long-lived JWT or API key) over username+password — see Auth notes below.

### Option H: Anthropic API direct (no bridge)

If you're calling the Anthropic Messages API directly with tool use, you
don't need a bridge — call `POST /v1/mcp` from your application
code and forward the tool definitions / results into the Messages API
yourself. See the JSON-RPC examples at the bottom of this doc.

---

## Step 3 — Use it

In any of the configured clients, try:

> *"What tables are in the SynapCores database, and how many rows does each have?"*

The assistant should call `list_tables`, then call `query` for each one with
`SELECT COUNT(*)`, then summarize the result in plain English.

Or:

> *"Train an AutoML model on the loyalty_members table to predict churn,
> then show me the top 10 at-risk Gold members with their risk scores."*

The assistant should chain `describe_table` → `train_model` → `query` (SELECT … with `predict` for scoring).

---

## Auth notes

The bridge supports two auth modes:

- **`SYNAPCORES_TOKEN`** — a pre-minted JWT or API key. No login round-trip
  at startup. Use this for hosted / CI environments where you don't want
  the bridge handling credentials.
- **`SYNAPCORES_USERNAME` + `SYNAPCORES_PASSWORD`** — the bridge logs in
  at startup and re-logs in if a tool call returns 401 (token aged out).
  Use this for local dev where it's just easier.

Don't set both — `SYNAPCORES_TOKEN` wins if it's present.

The bridge **never** writes credentials to disk or stdout; they live in
the client config's `env` block and are passed to the bridge process as
environment variables when the client spawns it.

---

## Available tools

v1.6.5.1-ce expanded the surface from 6 SQL-mediated tools to 14
first-class tools so the LLM doesn't have to translate every intent
through SQL. The original 6 are still there; the 8 new ones wrap
AIDB's ML/Vector/Graph/LLM primitives so the LLM can call them
without knowing AIDB-specific SQL (`EMBED`, `COSINE_SIMILARITY`,
`CREATE EXPERIMENT`, etc.).

### SQL surface (6 — alphabetical)

| Tool | What it does | Example invocation the assistant makes |
|---|---|---|
| `describe_table` | Columns, types, constraints for one table | `{ "table": "loyalty_members" }` |
| `execute` | Run DDL/DML (CREATE / INSERT / UPDATE / DELETE / DROP) | `CREATE TABLE temp (id INT, ...)`, `INSERT INTO ...` |
| `list_tables` | List every table in the current database | (no args) |
| `query` | Run any SELECT, return rows + types | `SELECT id, name FROM users WHERE tier='Gold' LIMIT 50` |
| `sql_manual` | Look up SynapCores SQL syntax docs | `{ "topic": "AUTOML.PREDICT" }` |
| `validate_query` | Parse + plan check without executing | `{ "sql": "SELECT foo FROM bar" }` |

### AutoML (4 — v1.6.5.1)

| Tool | What it does | Example invocation the assistant makes |
|---|---|---|
| `train_model` | Train an AutoML model on a column. Wraps `CREATE EXPERIMENT`. | `{ "collection": "loyalty_members", "target": "churned", "task_type": "binary_classification", "max_trials": 3 }` |
| `predict` | Score a trained model against inline input rows. Returns calibrated probabilities for classification. | `{ "model_name": "churn_model", "inputs": [{ "tenure_months": 5, "visits_30d": 0, "spend_30d": 12.5 }] }` |
| `list_models` | List the tenant's trained models. | `{ "task_type": "classification" }` |
| `describe_model` | Full details: algorithm, metrics, feature importance. | `{ "model_name": "churn_model" }` |

### Vector (2 — v1.6.5.1)

| Tool | What it does | Example invocation the assistant makes |
|---|---|---|
| `semantic_search` | Top-K similarity search. Embeds query internally — no SQL needed. | `{ "collection": "support_docs", "query_text": "how do I reset my password", "k": 5 }` |
| `embed_text` | Get a raw embedding vector for a piece of text. | `{ "text": "wireless noise-cancelling headphones" }` |

### Graph (1 — v1.6.5.1)

| Tool | What it does | Example invocation the assistant makes |
|---|---|---|
| `graph_query` | Run Cypher (MATCH/MERGE/CREATE) against the tenant graph. **Note**: as of v1.6.5.2, `$param` bind variables are not yet supported by the gateway's Cypher engine — inline the values into the query string (e.g. `MATCH (n {id: 'abc'})` instead of `MATCH (n {id: $id})`). | `{ "cypher": "MATCH (n:Person)-[:KNOWS]->(m:Person) RETURN n, m LIMIT 25" }` |

### LLM (1 — v1.6.5.1)

| Tool | What it does | Example invocation the assistant makes |
|---|---|---|
| `generate_text` | Generate text from the gateway's configured LLM. | `{ "prompt": "Summarize the top-3 churn risk factors in plain English.", "max_tokens": 200, "temperature": 0.4 }` |

All 14 are exercised by the contract tests; treat them as a stable surface.

---

## Example agent flow: retention-offer workflow

A realistic multi-tool chain the LLM can run **without ever writing
AIDB-specific SQL**:

```
Goal: "Find the 50 highest-churn-risk Gold members and draft a
       personalized retention offer for each."

Step 1.  describe_table { "table": "loyalty_members" }
         → schema (tenure_months, visits_30d, spend_30d, tier, churned, ...)

Step 2.  train_model { "collection": "loyalty_members",
                       "target": "churned",
                       "task_type": "binary_classification",
                       "max_trials": 5,
                       "model_name": "gold_churn" }
         → { "best_score": 0.94, "algorithm": "AutoML (classification)", "training_time_ms": 11423 }

Step 3.  query { "sql": "SELECT id, tenure_months, visits_30d, spend_30d
                          FROM loyalty_members
                          WHERE tier = 'Gold'
                          LIMIT 200" }
         → 200 candidate rows

Step 4.  predict { "model_name": "gold_churn",
                   "inputs": [ ... the 200 rows ... ] }
         → 200 calibrated probabilities; LLM sorts DESC and keeps the top-50

Step 5.  generate_text { "prompt": "You are a CRM copywriter. Write a 60-word
                                    retention offer for a Gold-tier member with
                                    tenure_months=5, visits_30d=2, spend_30d=$12.
                                    Risk score: 0.91.",
                          "max_tokens": 150,
                          "temperature": 0.6 }
         → personalized offer text

Step 6.  execute { "sql": "INSERT INTO retention_offers (member_id, risk, copy)
                            VALUES (...)" }
         → persisted
```

The same flow before v1.6.5.1 would have required the LLM to author
`CREATE EXPERIMENT … WITH (task_type='binary_classification', target_column='churned', max_trials=5)`,
`SELECT id, AUTOML.PREDICT('gold_churn', tenure_months, visits_30d, spend_30d) AS risk FROM …`,
and `SELECT GENERATE(…)` — three AIDB-specific SQL surfaces the LLM had
to memorize from `sql_manual`. With v1.6.5.1, none of that is needed.

---

## Verification (without an MCP client)

Quickest way to confirm the bridge works on your machine, no MCP client
involved:

```bash
SYNAPCORES_URL=http://127.0.0.1:8080 \
SYNAPCORES_USERNAME=admin \
SYNAPCORES_PASSWORD=YOUR_PW \
  node ~/.synapcores/synapcores-mcp-bridge.js <<'EOF'
{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}},"id":1}
{"jsonrpc":"2.0","method":"tools/list","id":2}
EOF
```

Expected output: two JSON lines, the second containing 14 tools
(6 SQL + 4 AutoML + 2 Vector + 1 Graph + 1 LLM).

---

## Direct JSON-RPC (no bridge, if you're integrating from code)

For application integrations not using a stdio MCP client, hit
the gateway directly:

```bash
# Get a JWT
JWT=$(curl -s -X POST http://127.0.0.1:8080/v1/auth/login \
        -H 'Content-Type: application/json' \
        -d '{"username":"admin","password":"YOUR_PW"}' \
      | jq -r '.access_token')

# Initialize
curl -s -X POST http://127.0.0.1:8080/v1/mcp \
  -H "Authorization: Bearer $JWT" \
  -H 'Content-Type: application/json' \
  -d '{
        "jsonrpc": "2.0",
        "method": "initialize",
        "params": {
          "protocolVersion": "2024-11-05",
          "capabilities": {},
          "clientInfo": { "name": "my-app", "version": "1.0" }
        },
        "id": 1
      }'

# List tools
curl -s -X POST http://127.0.0.1:8080/v1/mcp \
  -H "Authorization: Bearer $JWT" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":2}'

# Run a tool
curl -s -X POST http://127.0.0.1:8080/v1/mcp \
  -H "Authorization: Bearer $JWT" \
  -H 'Content-Type: application/json' \
  -d '{
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
          "name": "query",
          "arguments": { "sql": "SELECT 1 + 1 AS x" }
        },
        "id": 3
      }'
```

---

## Troubleshooting

### MCP client shows "synapcores: failed to start" or no tools

- Check `node --version` — must be 18+.
- Run the bridge manually (the verification block above). If that fails,
  the issue is bridge↔gateway; if that succeeds, the issue is
  client↔bridge (usually a wrong absolute path in the config).
- On macOS, GUI MCP clients (Claude Desktop, Cursor) run the bridge under the GUI's PATH, not
  your shell's. Use the **absolute** path to `node` if `node` isn't in
  `/usr/local/bin/` or `/opt/homebrew/bin/`: `"command":
  "/Users/YOU/.nvm/versions/node/v22.0.0/bin/node"`.

### `bridge: login failed: 401`

- Wrong username/password. The first-boot admin password is printed
  ONCE in the gateway's stdout — capture it from `journalctl -u
  synapcores` or your install log. To set a known password,
  reinstall with `SYNAPCORES_ADMIN_PASSWORD=mypw ./install-ce.sh` or
  set the `AIDB_ADMIN_PASSWORD` env var on the systemd unit.

### `bridge: SYNAPCORES_TOKEN missing and SYNAPCORES_USERNAME / PASSWORD not set`

- The env block in your client config didn't get passed through.
  JSON syntax errors silently drop the whole block; validate the file
  at jsonlint.com first.

### Tool call hangs or times out

- The gateway's default query timeout is 5 min as of v1.6.5. If a SELECT
  is genuinely slow, that's an AIDB query plan issue — try
  `validate_query` first to see the plan.
- If a `tools/call` for `execute` returns nothing for >30 s but
  `query` works, check that you're not waiting on `CREATE EXPERIMENT`
  on a huge dataset (training is synchronous; 100k rows ≈ 11 s,
  but bigger datasets scale roughly linearly).

### `graph_query` returns "Parse error: expected expression at offset N"

- You're probably using `$param` bind variables. As of v1.6.5.2, the
  gateway's Cypher engine doesn't accept named parameters — inline the
  values directly into the query string. Will be supported in v1.6.6.

### "Tool calls are working but the assistant doesn't see the results"

- That's usually a client-side conversation length / context issue,
  not an MCP issue. Try shorter queries (`LIMIT 100`); summarize
  yourself instead of asking the assistant to summarize raw rows.

---

## Multi-tenant + remote deployments

Each MCP connection uses the JWT's tenant_id; one session
configured for tenant A cannot see tenant B's data. If you're running
the gateway on a remote box (`SYNAPCORES_URL=https://aidb.example.com`),
the bridge still runs locally on the user's machine — only the JSON-RPC
payloads cross the network, over HTTPS, with the JWT in the
Authorization header.

For team setups: mint per-developer API keys (gateway → `/v1/api-keys`),
each scoped to a single tenant. Distribute the keys + the bridge script;
no shared password.

---

## Related

- [SQL reference](sql-reference.md) — full AIDB SQL syntax
- [Quickstart](quickstart.md) — install + first connection
- [Community discussions](https://github.com/SynapCores/synapcores-docs/discussions) — questions, share what you built
