# Releases

This page tracks SynapCores Community Edition releases. The latest release is always installed by the [one-liner installer](../quickstart.md):

```bash
curl -fsSL https://get.synapcores.com | sh
```

To pin a specific version:

```bash
curl -fsSL https://get.synapcores.com | SYNAPCORES_VERSION=v1.8.5-ce sh
```

Or via Docker:

```bash
docker pull synapcores/community:v1.8.5-ce
```

Release artifacts (binaries + SHA-256 sidecars + the install script) live on the [SynapCores/synapcores-releases](https://github.com/SynapCores/synapcores-releases/releases) GitHub repository.

## Release history

| Version | Date | Headline |
|---|---|---|
| [**v1.8.5-ce**](v1.8.5-ce.md) | 2026-06-15 | Agent-memory primitives — `MEMORY_STORE` / `MEMORY_RECALL` / `MEMORY_FORGET` |

For every prior release (v1.0 → v1.8.4) with short user-facing summaries — what was fixed, what to watch out for if you're on that exact tag — see the [**Full changelog**](changelog.md). Per-release deep-dive pages are added going forward starting with v1.8.5.
