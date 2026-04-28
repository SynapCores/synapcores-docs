# SynapCores Documentation

**SynapCores** is an AI-native database. SQL, vector search, ML, and an
embedded LLM inference engine in one self-hosted binary.

## Install in one line

```bash
curl -fsSL https://get.synapcores.com/install.sh | sh
```

Detects your platform, downloads the latest signed release, verifies its
checksum, and runs the system installer. See [Quickstart](quickstart.md)
for the full walkthrough.

## What's in the box

| Built-in | Out-of-the-box | Optional |
| --- | --- | --- |
| SQL engine | 120 seeded recipes on first boot | GPU acceleration (source build) |
| Vector storage + HNSW search | Random admin password, printed once | Ollama integration |
| Embedded React Web UI on the same port | Embeddings via Hugging Face auto-download | OpenAI / Anthropic providers |
| llama-cpp inference engine | Hardened systemd unit | TLS termination |
| Recipe + dashboard system | REST + WebSocket API | Docker container |

## Where to start

<div class="grid cards" markdown>

- :material-rocket-launch:{ .lg .middle } **[Quickstart](quickstart.md)**

    Install, configure, and verify a working CE deployment.

- :material-server:{ .lg .middle } **[Hardware requirements](requirements.md)**

    Minimum and recommended sizing per platform.

- :material-robot:{ .lg .middle } **[First-run AI setup](ai-setup.md)**

    Wire up a local GGUF, Ollama, or external provider for AI Chat
    and NL2SQL.

- :material-monitor-dashboard:{ .lg .middle } **[Web UI](web-ui.md)**

    The bundled React UI on `:8080` — dashboard, query editor,
    AI chat, vector search, recipes, graph explorer.

- :material-account-group:{ .lg .middle } **[Community vs Enterprise](enterprise.md)**

    What's in CE, what's gated to EE, and how to upgrade.

</div>

## Releases

Source is private; binary releases are public:
[github.com/SynapCores/synapcores-releases](https://github.com/SynapCores/synapcores-releases/releases).

## Need help?

See the [Support page](support.md) — issue tracker, security disclosure,
and how to contact the team.
