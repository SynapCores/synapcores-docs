# First-run AI setup


The CE binary ships with the **inference engine** ([llama-cpp](https://github.com/ggerganov/llama.cpp)
embedded). As of **v1.3.1-ce** the bootstrap installer also offers to
download a default GGUF model on first run, so **AI Chat works
end-to-end with zero post-install steps** when you accept the prompt.

| What works on a fresh install | Notes |
| --- | --- |
| SQL, vector storage, recipes, dashboards, REST API, JWT auth, Web UI | always |
| Document ingest, FilesystemRAG | always |
| NLI claim verification (`/v1/ai/nli/*`, `NLI_VERIFY`) | after a one-time `pull nli-default` (~576 MB) + restart with `AIDB_NLI_ENABLED=1` — see the [NLI manual](nli.md) |
| Embeddings (`EMBED()`, vector search by text) | first call triggers a one-time MiniLM download (~90 MB) into `~/.cache/huggingface/` |
| **AI Chat** | works if you accept the bootstrap's GGUF download prompt |
| NL2SQL (`ASK ...`) | works for queries the model can answer directly; tool-using paths require Ollama or a hosted provider — see "Tool-using agents" below |

### Default install (recommended)

```bash
curl -fsSL https://get.synapcores.com/install.sh | sh
# → drops the binary, installs runtime deps, then prompts:
# [get-synapcores] Download default LLM (Llama 3.2 1B Q4_K_M, ~700MB)? [Y/n]
# → answer Y and AI Chat works after `systemctl start synapcores`.
```

The default config (`/etc/synapcores/gateway.toml` on Linux,
`~/.synapcores/gateway.toml` on macOS) ships pre-wired:

```toml
[query.ai_service]
provider        = "native"
model           = "llama-3.2-1b-instruct-q4_k_m"
embedding_model = "minilm"
```

### Generative LLM (Chat / NL2SQL) — three options

#### Option 1 — In-process GGUF (default — fully offline)

What the bootstrap does for you on `Y`. To do it manually:

1. Create the models dir:

   ```bash
   # Linux
   sudo -u synapcores mkdir -p /opt/synapcores/models/text
   # macOS
   mkdir -p ~/.synapcores/models/text
   ```

2. Download a GGUF-quantized model:

   ```bash
   # Linux
   sudo -u synapcores curl -fL \
     -o /opt/synapcores/models/text/llama-3.2-1b-instruct-q4_k_m.gguf \
     "https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf"
   ```

3. Confirm the gateway's systemd unit (Linux) sets `AIDB_MODELS_DIR`:

   ```bash
   sudo systemctl cat synapcores | grep AIDB_MODELS_DIR
   # Expect: Environment=AIDB_MODELS_DIR=/opt/synapcores/models/text
   ```

   If missing (older install), add it:

   ```bash
   sudo systemctl edit synapcores
   ```

   ```ini
   [Service]
   Environment="AIDB_MODELS_DIR=/opt/synapcores/models/text"
   Environment="AIDB_CONTEXT_SIZE=4096"
   Environment="AIDB_GPU_LAYERS=0"  # CPU-only; raise if you built from source w/ GPU
   ```

4. Restart and verify:

   ```bash
   sudo systemctl restart synapcores
   sudo journalctl -u synapcores -f | grep -i gguf
   # Expect: Loading GGUF model: llama-3.2-1b-instruct-q4_k_m from ".../models/text"
   ```

CPU-only is the default for published binaries; GPU acceleration
(`--features llama-cpp/cuda` or `metal`) requires a source build.
Performance bands on the default Llama 3.2 1B Q4_K_M model:

| Hardware | tokens/sec |
| --- | --- |
| Apple Silicon M1/M2/M3/M4 (CPU) | 30–80 |
| Modern x86 with AVX2 (Zen 3+, Intel 11th gen+) | 5–15 |
| Older x86 without AVX2 | 1–3 |

To swap models, drop a different GGUF in the same dir and update the
`model` field in the config to its filename without `.gguf`.
Recommended bigger options:

* **Llama 3.2 3B Instruct Q4_K_M** (~2 GB, better quality)
* **Phi-3.5 Mini Q4_K_M** (~2 GB, strong reasoning for size)
* **Qwen 2.5 7B Instruct Q4_K_M** (~4.5 GB, near-frontier quality on CPU)

### Tool-using agents (NL2SQL with tool calls, FilesystemRAG agent search)

The default `native` provider runs in-process llama-cpp inference and
gives you plain text completions back. **It does not implement
native tool calling** the way Ollama's `/api/chat` endpoint does, so
agent flows that require the model to emit structured `tool_calls`
will fall back to direct text — fine for most chat, but agents that
chain SQL execution or filesystem search via tool invocation will be
degraded.

If you need tool-using agents today, swap to Ollama or a hosted
provider — both implement the tool-calling API. v1.3.2 is expected
to add prompt-template-based tool extraction so the native provider
can participate in tool flows too.

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

### Changing configuration in Docker

In a container there is **no systemd and no file to edit "inside" the
container**. The config is a **host file you bind-mount** read-only; you edit it
**on the host** and **restart the container** to apply (config is read once at
process start — there is no hot-reload).

**1. Keep `gateway.toml` on the host and mount it** (with a named volume for
data so it survives restarts):

```bash
docker run -d --name synapcores -p 8080:8080 \
  -v "$PWD/gateway.toml":/etc/synapcores/gateway.toml:ro \
  -v synapcores-data:/var/lib/synapcores \
  -e AIDB_ACCEPT_LICENSE=1 \
  -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
  synapcores/community:latest --config /etc/synapcores/gateway.toml
```

The container's copy is read-only and ephemeral — the file that matters is the
one on your host.

**2. Change the model: edit the host file.** For a RAG deployment pointed at
OpenAI (embeddings *and* generation), set the host `gateway.toml` to:

```toml
data_dir = "/var/lib/synapcores"
[server]
listen_addr = "0.0.0.0:8080"
[auth]
enabled = true

[query.ai_service]
provider        = "openai"
model           = "gpt-4o"                   # generation (GENERATE / AGENT_RUN)
embedding_model = "text-embedding-3-small"   # EMBED() -> OpenAI embeddings (split mode)
# api_key via -e OPENAI_API_KEY (below), or inline: api_key = "sk-..."
```

To use the in-process model instead, set `provider = "native"`,
`model = "llama-3.2-1b-instruct-q4_k_m"`, `embedding_model = "minilm"`. For
Ollama on the host, `provider = "ollama"` with
`base_url = "http://host.docker.internal:11434"` (note: from inside a container,
the host is `host.docker.internal`, not `localhost`).

**3. Secrets go through `-e`, not the file.** Add the key to `docker run`:

```bash
#   ... -e OPENAI_API_KEY=sk-... ...
```

**4. Restart the container to apply:**

```bash
docker restart synapcores
```

**5. Verify it took effect** — the boot log prints the resolved provider/model:

```bash
docker logs synapcores 2>&1 | grep -iE "Configured OpenAI provider|SPLIT mode|Local AI default"
# e.g. "OpenAI provider configured for SPLIT mode: completion=gpt-4o, embedding=text-embedding-3-small"
```

!!! note "Embedding dimensions must match your VECTOR columns"
    Changing the embedding model changes the vector width:
    OpenAI `text-embedding-3-small` = **1536**, `-3-large` = **3072**, native
    `minilm` = **384**. Declare your column to match (e.g. `embedding VECTOR(1536)`),
    and do not query vectors embedded under one model with another — they are
    different spaces, so re-embed if you switch models.

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

