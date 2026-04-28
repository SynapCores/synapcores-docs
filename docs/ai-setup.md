# First-run AI setup


The CE binary ships with the **inference engine** ([llama-cpp](https://github.com/ggerganov/llama.cpp)
embedded), but **no model weights**. Out of the box this means:

| What works without any setup | What needs a model |
| --- | --- |
| SQL, vector storage, recipes, dashboards, REST API, JWT auth, Web UI, document ingest | Embeddings (`EMBED()`, vector search by text) |
|  | AI Chat |
|  | NL2SQL (`ASK ...`) |

### Embeddings — auto-downloaded on first use

The first call to `EMBED()` (or any vector search by text) triggers a one-
time download from Hugging Face into `~/.cache/huggingface/`. Default model
is **MiniLM** (~90 MB, 384-dim). After the first call, embeddings run
offline. No config required — it Just Works as long as the gateway has
outbound network access to `huggingface.co`.

### Generative LLM (Chat / NL2SQL) — pick one

You have three options. Pick whichever matches your deployment.

#### Option 1 — Local GGUF (fully offline, no external API)

1. Download a GGUF-quantized model. The community defaults to a 7B-parameter
   Q4\_K\_M quant for a good size/speed balance (~4.5 GB on disk):

   ```bash
   sudo -u synapcores mkdir -p /opt/synapcores/models/text
   sudo -u synapcores curl -fsSL -o /opt/synapcores/models/text/llama-3-8b.gguf \
        "https://huggingface.co/<repo>/<model>/resolve/main/llama-3-8b-instruct-q4_k_m.gguf"
   ```

2. Tell the gateway where to find it (the default is `./models/text/`,
   relative to `data_dir` — pinning to an absolute path is cleaner):

   ```bash
   sudo systemctl edit synapcores
   ```

   ```ini
   [Service]
   Environment="AIDB_MODELS_DIR=/opt/synapcores/models/text"
   # Optional: tune llama-cpp behavior
   Environment="AIDB_CONTEXT_SIZE=4096"
   Environment="AIDB_GPU_LAYERS=0"        # CPU-only; set higher if you built from source w/ GPU
   ```

3. Tell the AI service which model to use, in `/etc/synapcores/gateway.toml`:

   ```toml
   [query.ai_service]
   provider = "native-gguf"
   model    = "llama-3-8b"           # filename WITHOUT the .gguf extension
   ```

4. Restart and verify:

   ```bash
   sudo systemctl restart synapcores
   sudo journalctl -u synapcores -f | grep -i gguf
   # Expect: Loading GGUF model: llama-3-8b from "/opt/synapcores/models/text"
   ```

Performance bands on this 7B-Q4 model — see [the platforms README](https://github.com/SynapCores/synapcores-installer#local-llm-inference)
for details. CPU-only is the default; GPU acceleration requires a source build.

#### Option 2 — Wire up Ollama (recommended for evaluation)

Ollama runs as a separate service on the host and gives you a model menu
with one-line `ollama pull` commands. The gateway just hits Ollama over
HTTP — no model files in your data dir.

1. Install Ollama and pull a model:

   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ollama pull llama3.2          # ~2 GB, runs comfortably on 8 GB RAM
   ```

2. Wire the gateway to it in `/etc/synapcores/gateway.toml`:

   ```toml
   [query.ai_service]
   provider = "ollama"
   base_url = "http://localhost:11434"
   model    = "llama3.2"
   ```

3. Restart:

   ```bash
   sudo systemctl restart synapcores
   ```

#### Option 3 — External provider (OpenAI / Anthropic)

Cheapest setup, requires sending data to a third party. Counts against
the CE per-day quota (`MaxExternalLlmCallsPerDayPerUser = 300`).

```toml
# /etc/synapcores/gateway.toml
[query.ai_service]
provider = "openai"
api_key  = "${OPENAI_API_KEY}"     # or hardcode if not secret-sensitive
model    = "gpt-4o-mini"           # or any other model name the provider serves
```

Set the secret via systemd so it stays out of the file:

```bash
sudo systemctl edit synapcores
# Add: Environment="OPENAI_API_KEY=sk-..."
sudo systemctl restart synapcores
```

Same pattern for Anthropic (`provider = "anthropic"`, `ANTHROPIC_API_KEY`).

### Verifying AI is wired up

```bash
TOKEN="<JWT obtained from /v1/auth/login>"

# Embeddings (no LLM needed):
curl -H "Authorization: Bearer $TOKEN" -X POST \
     -H "Content-Type: application/json" \
     -d '{"sql":"SELECT EMBED(\"hello world\") AS v"}' \
     http://localhost:8080/v1/query/execute

# AI Chat (needs Option 1, 2, or 3 wired up):
curl -H "Authorization: Bearer $TOKEN" -X POST \
     -H "Content-Type: application/json" \
     -d '{"message":"What tables exist?","database":"default"}' \
     http://localhost:8080/v1/chat/message
```

If AI Chat returns `{"error":"no AI provider configured"}` or similar,
the `[query.ai_service]` block is missing from your config. Pick an
option above and restart.

