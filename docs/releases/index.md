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

Older releases are documented on their [GitHub release pages](https://github.com/SynapCores/synapcores-releases/releases) until they are migrated here.
