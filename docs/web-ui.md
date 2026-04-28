# Web UI

SynapCores ships with a built-in **React Web UI**, embedded directly
into the gateway binary. It's served on the same port as the REST API,
so a fresh install gives you a working browser experience the moment
the service starts — no separate frontend deployment, no nginx, no
extra ports to expose.

## How to access it

The UI is at the **root URL** of whatever address the gateway is
listening on.

| Listener | URL |
| --- | --- |
| Default HTTP | `http://<host>:8080/` |
| With TLS enabled | `https://<host>:8443/` |

For a same-host install, that's `http://localhost:8080/`. Behind a
reverse proxy or a public DNS name, point your browser at that
hostname instead.

If you see the SynapCores landing page in dark mode, the UI is live.
If you get a 404, check that you typed `/` (root) and not `/index` or
similar — the SPA shell only serves at root, with React Router taking
over for client-side paths after that.

## First-time login

Every fresh install creates a single `admin` user on first boot. The
password is generated at runtime, **printed once** to the gateway
logs, and never shown again.

To capture it:

```bash
sudo journalctl -u synapcores -n 200 --no-pager | grep -A 7 "FIRST-BOOT"
```

You should see a banner like:

```
===============================================================
  FIRST-BOOT ADMIN CREDENTIAL
  username: admin
  password: xUdQX5JAk1OsA1ehfoWylyZL
  source: auto-generated
  Capture this now — it will not be shown again.
  Change it via the admin UI on first login.
===============================================================
```

If you missed it (e.g. logs already rotated) you have two options:

1. **Stop the service, delete the data dir, restart** — you'll lose
   all data, but you'll get a new admin password printed. Only viable
   on a brand-new install.
2. **Pin a known password via `AIDB_ADMIN_PASSWORD`** before first
   start (next install only):

   ```bash
   sudo systemctl edit synapcores
   # add: Environment="AIDB_ADMIN_PASSWORD=<your password, 12+ chars>"
   sudo systemctl restart synapcores
   ```

   This only takes effect when the `admin` user does not yet exist.

After logging in, **change the password immediately** under
`Settings → Account`.

## What's in the UI

### Dashboard
Real-time system metrics, query throughput, connection counts, recent
slow queries. Updates over WebSocket — no polling.

### Query Editor
Monaco-based SQL editor with syntax highlighting, autocomplete against
your live schema, query history, and result-set rendering with
sortable/filterable tables. Press **Ctrl+Enter** (or **Cmd+Enter** on
macOS) to execute.

### NL2SQL
Type a question in natural English (`"orders where total exceeded
$500 last quarter"`); the AI generates the SQL, shows you the plan,
and executes on confirmation. Counts against the per-day NL2SQL quota
(see [Limits & quotas](limits.md)).

### AI Chat
Conversational interface to your data. Ask questions, ask the model
to write SQL, ask for visualizations. Sessions are persisted per-user
and searchable. Counts against the per-day AI chat quota.

### Vector search
Browse vector collections, run similarity search by text or by an
example record, inspect distances and metadata filters. Embedding
generation triggers a one-time Hugging Face download on first use —
see [First-run AI setup](ai-setup.md).

### Recipes
Browse and execute the 120 SQL recipes that ship pre-loaded on first
boot — common patterns like creating tables, building image
galleries, setting up RAG pipelines. Each recipe is editable; you
can clone, modify, save your own.

### Collections
CRUD interface for document collections. Create, browse, edit, delete
documents; manage indexes; inspect schemas.

### Graph explorer
Interactive node/edge canvas for native graph data. MATCH-style
traversals visualized live with d3-force layout, full-screen mode,
pan/zoom.

### Media gallery
Upload PDFs, images, audio, and video. The gateway extracts text via
OCR (Tesseract), generates embeddings via CLIP for images, and
transcribes audio/video via the embedded inference engine. All
multimedia content is searchable from the same vector index.

### Settings
Account password change, API key management, user preferences,
license info (calls `/v1/license` under the hood).

## Browser support

Tested on:

- Chrome / Edge 120+
- Firefox 121+
- Safari 17+ (macOS, iPad)

The build targets modern evergreen browsers — IE11 is not supported,
and ES2020+ syntax is used unconditionally. WebSocket support is
required for the Dashboard and AI Chat features.

## Customizing the UI

The bundled UI is the React build that ships with the gateway binary.
It calls back to the same origin for all REST and WebSocket requests
(no `VITE_API_URL` override needed for self-hosted deployments).

If you need a custom frontend — different branding, embedded into
another app, or replaced entirely — point your own client at the REST
API at `/v1/...` and the WebSocket endpoints at `/ws/...`. The bundled
UI uses no private APIs; everything it does, you can do too.
