# First-run AI setup

The CE binary ships with the **inference engine** ([llama.cpp](https://github.com/ggerganov/llama.cpp)
embedded) built in. There is no separate AI service to install — you point the gateway at a
model and the AI SQL functions (`EMBED`, `GENERATE`, `AGENT_RUN`, …), the REST AI endpoints,
and AI Chat all light up.

This page wires up the **generative + embedding provider**. For a capability-by-capability
"what do I switch on to use X" map, see **[Capabilities & how to enable each](capabilities.md)**.

## What works on a fresh install

| Capability | Works out of the box? |
| --- | --- |
| SQL, vector **storage**, recipes, dashboards, REST API, JWT auth, Web UI | ✅ always |
| Document ingest, [Filesystem Collections](filesystem-rag.md) | ✅ always |
| Embeddings (`EMBED()`, vector search by text) | ✅ — `all-minilm:latest` (384-dim) pulls on first use |
| **AI Chat / `GENERATE` / NL2SQL / `AGENT_RUN`** | ✅ when a chat model is pulled (the installer offers to do this; otherwise `auto_pull` fetches it on first use) |
| [Claim verification (NLI)](nli.md) | ⚙️ one-time `--with-nli` opt-in (or `AIDB_NLI_ENABLED=1`) + restart |
| [One-call decisions (d1)](decisions.md) | ⚙️ one-time `synapcores pull d1:latest` |

!!! info "The default provider is in-process — no cloud account, no sidecar"
    Since v1.8.0 the default provider is **`local`**: it runs quantized GGUF models
    **in-process** and pulls them on demand from the model registry with `synapcores pull`
    (the same `name:tag` scheme as Ollama, but no separate Ollama daemon). Your data never
    leaves the box. You can also point at an Ollama server or a hosted API — see the three
    options below.

## The default config

The installer writes this `[query.ai_service]` block to your `gateway.toml`
(`/etc/synapcores/gateway.toml` on Linux, `~/.synapcores/gateway.toml` on macOS):

```toml
[query.ai_service]
provider        = "local"            # in-process llama.cpp; no external service
model           = "qwen2.5:7b"       # chat / GENERATE / AGENT_RUN / NL2SQL
embedding_model = "all-minilm:latest"  # EMBED() -> 384-dim vectors
auto_pull       = true               # fetch a named model on first use if it is missing
```

With `auto_pull = true`, the first call that needs a model pulls it automatically. To pull
ahead of time (recommended on a fresh box, so the first request isn't slow):

```bash
synapcores --config /etc/synapcores/gateway.toml pull qwen2.5:7b      # ~4.4 GiB chat model
synapcores --config /etc/synapcores/gateway.toml pull all-minilm:latest  # ~44 MiB embeddings
synapcores --config /etc/synapcores/gateway.toml models list
```

!!! tip "Hardware-adaptive default"
    On a host with **≥ 8 GiB RAM**, `qwen2.5:7b` is the default chat model. On tighter boxes,
    pick a smaller GGUF (below) and set it as `model`. Embeddings (`all-minilm:latest`, 384-dim)
    run in tens of milliseconds even on a Raspberry Pi.

## Generative LLM (Chat / NL2SQL / AGENT_RUN) — three options

### Option 1 — In-process model (default, fully offline)

This is what the default config above does. To change or add a model, pull it and point
`model` at it:

```bash
synapcores --config /etc/synapcores/gateway.toml pull llama3.2:3b
```

```toml
[query.ai_service]
provider  = "local"
model     = "llama3.2:3b"
auto_pull = true
```

Restart to pick up a config change (`sudo systemctl restart synapcores`). No restart is needed
to *use* a model that is already pulled.

Good in-process choices:

| Model (`pull` name) | Size | Notes |
| --- | --- | --- |
| `qwen2.5:7b` | ~4.4 GiB | **default** — strong tool-use + reasoning on CPU |
| `llama3.2:3b` | ~2 GiB | lighter, good general chat |
| `phi3.5` | ~2 GiB | strong reasoning for its size |
| `llama3.2:1b` | ~0.7 GiB | smallest; for very constrained hosts |

CPU-only is the default for the published binaries. GPU acceleration (`cuda` / `metal` /
`vulkan`) requires a source build — see [Hardware requirements](requirements.md). Throughput
bands on a 7B Q4\_K\_M model:

| Hardware | tokens/sec |
| --- | --- |
| Apple Silicon M1–M4 | 30–80 |
| Modern x86 with AVX2 (Zen 3+, Intel 11th gen+) | 5–15 |
| Older x86 without AVX2 | 1–3 |

!!! success "Tool-using agents work on the in-process provider"
    The `local` provider implements **native, grammar-constrained tool calling**, so
    `AGENT_RUN` and the tool-using NL2SQL / agent flows run entirely in-process on a
    tool-capable model (the default `qwen2.5:7b` qualifies). The older guidance that said
    "switch to Ollama for tool calls" is obsolete. If you point `model` at a model that
    cannot emit tool calls, `AGENT_RUN` returns a clear error naming the config key to
    change — it does not silently degrade.

### Option 2 — Ollama (a separate model server)

Run Ollama on the host and have the gateway call it over HTTP. Useful if you already run
Ollama or want its model menu.

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2
```

```toml
[query.ai_service]
provider = "ollama"
base_url = "http://localhost:11434"
model    = "llama3.2"
```

```bash
sudo systemctl restart synapcores
```

### Option 3 — External provider (OpenAI / Anthropic / Gemini)

Cheapest to run, but sends data to a third party, and counts against the CE quota
(`MaxExternalLlmCallsPerDayPerUser = 300` — see [Limits](limits.md)).

```toml
[query.ai_service]
provider        = "openai"                   # or "anthropic" / "gemini"
model           = "gpt-4o-mini"              # a model the provider serves
embedding_model = "text-embedding-3-small"   # EMBED() -> OpenAI embeddings (1536-dim)
# api_key supplied via systemd env (below), or inline: api_key = "sk-..."
```

Keep the secret out of the file with a systemd override:

```bash
sudo systemctl edit synapcores
# [Service]
# Environment="OPENAI_API_KEY=sk-..."
sudo systemctl restart synapcores
```

Valid `provider` values: `local` (default, in-process), `ollama`, `openai`, `anthropic`
(alias `claude`), `gemini`. (`native` is a legacy filesystem-GGUF provider; prefer `local`.)

## Changing configuration in Docker

A container has no systemd and no file to edit "inside" it. The config is a **host file you
bind-mount read-only**; you edit it on the host and **restart the container** (config is read
once at process start — there is no hot-reload).

```bash
docker run -d --name synapcores -p 8080:8080 \
  -v "$PWD/gateway.toml":/etc/synapcores/gateway.toml:ro \
  -v synapcores-data:/var/lib/synapcores \
  -e AIDB_ACCEPT_LICENSE=1 \
  -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
  synapcores/community:latest --config /etc/synapcores/gateway.toml
```

Point a RAG deployment at OpenAI (generation *and* embeddings) by setting the host
`gateway.toml`:

```toml
data_dir = "/var/lib/synapcores"
[server]
listen_addr = "0.0.0.0:8080"
[auth]
enabled = true

[query.ai_service]
provider        = "openai"
model           = "gpt-4o"
embedding_model = "text-embedding-3-small"
```

To stay fully in-process instead, use `provider = "local"`, `model = "qwen2.5:7b"`,
`embedding_model = "all-minilm:latest"`. For Ollama on the host, `provider = "ollama"` with
`base_url = "http://host.docker.internal:11434"` (from inside a container the host is
`host.docker.internal`, not `localhost`). Pass secrets with `-e OPENAI_API_KEY=sk-...`, then
`docker restart synapcores`.

!!! note "Embedding dimensions must match your VECTOR columns"
    Changing the embedding model changes the vector width: native `all-minilm` = **384**,
    OpenAI `text-embedding-3-small` = **1536**, `-3-large` = **3072**. Declare your column to
    match (e.g. `embedding VECTOR(1536)`), and never query vectors embedded under one model
    with another — they are different spaces. This also bites on **upgrade**: a data dir
    embedded with one model mismatches if a new config sets a different `embedding_model`.
    Keep `embedding_model` the same as the data you already have, or re-embed.

## Verify AI is wired up

Get a token (`POST /v1/auth/login`), then exercise the two core paths through
`/v1/query/execute` — unambiguous and provider-independent:

```bash
TOKEN="<JWT from /v1/auth/login>"

# Embeddings (no chat model needed):
curl -sS -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"sql":"SELECT EMBED('\''hello world'\'') AS v"}' \
  http://localhost:8080/v1/query/execute

# Generation (needs a chat model from Option 1, 2, or 3):
curl -sS -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"sql":"SELECT GENERATE('\''Say hello in one word.'\'') AS reply"}' \
  http://localhost:8080/v1/query/execute
```

AI Chat has its own endpoints, `POST /v1/ai/chat` and `POST /v1/ai/chat/stream` (SSE). If a
call returns `no AI provider configured`, the `[query.ai_service]` block is missing — pick an
option above and restart.
