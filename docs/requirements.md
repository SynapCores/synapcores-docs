# Hardware requirements

## Supported platforms

CE binaries are built for Linux on every release. macOS and Windows are
out of scope for the v1.0 native release; both can run CE via the
[Docker image](quickstart.md#docker).

| Platform | Triple | Tarball |
| --- | --- | --- |
| Linux x86\_64 | `x86_64-unknown-linux-gnu` | `synapcores-ce-VER-linux-x86_64.tar.gz` |
| Linux aarch64 | `aarch64-unknown-linux-gnu` | `synapcores-ce-VER-linux-aarch64.tar.gz` |

The Linux binaries are built against **glibc 2.35** (Ubuntu 22.04), so
they run on:

- Ubuntu 22.04+
- Debian 12+
- RHEL 9 / Rocky 9 / Alma 9

They will **not** run on RHEL 7 / CentOS 7 (glibc too old).

## Sizing

| Resource | Minimum | Recommended |
| --- | --- | --- |
| CPU | 2 cores, x86\_64-v2 | 4+ cores, x86\_64-v3 (AVX2) |
| RAM | 4 GB | 8 GB+ (LLM inference needs headroom) |
| Disk | 10 GB free in the data dir | 50+ GB SSD (vector indexes are I/O-heavy) |
| File descriptors | `ulimit -n` ≥ 4096 | 65 536 (the systemd unit sets this) |
| Network | TCP `8080` (HTTP) + `8443` (TLS) free | Same |
| GPU | Not required | Optional — improves LLM inference throughput; requires source build |

The CE binary runs out of the box on a small VM (4 GB / 2 vCPU).
Throughput on big workloads is bounded by RAM (HNSW vector indexes are
RAM-resident) and AVX2 availability (llama-cpp inference is much slower
without it).

## What the installer checks

When you run `curl -fsSL https://get.synapcores.com/install.sh | sh`,
the bootstrap script verifies:

- OS is Linux (bails with a Docker hint on macOS, errors on anything else)
- Architecture is `x86_64` or `aarch64`
- Tarball SHA-256 matches the published checksum
- Downloaded binary self-reports as Community

It does **not** check (your responsibility before running in production):

- RAM headroom
- Disk space at the data dir
- glibc version (you'd get a runtime symbol error on older distros, not an install-time refusal)
- Whether `:8080` / `:8443` are already in use
- File descriptor limits (the systemd unit sets `LimitNOFILE=65536`, but a binary-only install or `docker run` won't get that for free)
- AVX2 availability (llama-cpp will work without it but is slower)

## GPU acceleration

The published CE binaries are **CPU-only** — no CUDA, Metal, Vulkan, or
OpenCL backends are linked in.

Performance bands you can expect on a 7B-parameter Q4\_K\_M GGUF model:

| Hardware | Tokens/sec |
| --- | --- |
| Apple Silicon M1/M2/M3/M4 (Docker) | 30–80 |
| Modern x86 with AVX2 (Zen 3+, Intel 11th gen+) | 5–15 |
| Older x86 without AVX2 | 1–3 (painful for chat) |
| Linux ARM (Graviton, Pi 5) | 3–8 |

For embedding-only workloads (semantic search, vector indexing) the
numbers are dramatically better — 384/768-dim models run in tens of
milliseconds even on a Pi.

If you need GPU-accelerated inference (e.g. a 70B model in real time),
build SynapCores from source with `--features llama-cpp/cuda` (or
`metal`, `vulkan`). Pre-built GPU binaries are not distributed because
the cross-product of CUDA versions and driver versions is unmanageable
across distros.
