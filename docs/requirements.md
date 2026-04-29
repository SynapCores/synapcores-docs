# Hardware requirements

## Supported architectures

Starting with v1.3.0-ce, CE ships native binaries for Linux **and**
macOS on every release. Windows users run CE via the
[Docker image](quickstart.md#docker) or WSL 2.

| Architecture | Triple | Tarball |
| --- | --- | --- |
| Linux x86\_64 | `x86_64-unknown-linux-gnu` | `synapcores-ce-VER-linux-x86_64.tar.gz` |
| Linux aarch64 | `aarch64-unknown-linux-gnu` | `synapcores-ce-VER-linux-aarch64.tar.gz` |
| macOS x86\_64 (Intel) | `x86_64-apple-darwin` | `synapcores-ce-VER-darwin-x86_64.tar.gz` |
| macOS aarch64 (Apple Silicon) | `aarch64-apple-darwin` | `synapcores-ce-VER-darwin-aarch64.tar.gz` |

## Supported Linux distributions

The CE binary is dynamically linked against system FFmpeg + Tesseract +
Leptonica + FreeType + Fontconfig. The bootstrap installer
(`curl -fsSL https://get.synapcores.com | sh`) detects your distro and
selects the matching tarball variant (FFmpeg 4 SONAMEs for 22.04 /
Debian 12; FFmpeg 6 SONAMEs for 24.04 / Debian 13). On macOS, the
`install-ce.sh` equivalent is the user's responsibility — the bootstrap
doesn't yet auto-set up Homebrew. See [macos.md](macos.md) for the
manual brew steps.

That gives the following distro matrix:

| Distro | Status | Notes |
| --- | --- | --- |
| **Ubuntu 22.04 LTS** | ✅ Supported | Native binary, FFmpeg 4 ABI |
| **Debian 12 (Bookworm)** | ✅ Supported | Native binary, FFmpeg 4 ABI |
| **Ubuntu 24.04 LTS** | ✅ Supported | Native binary, FFmpeg 6 ABI |
| **Debian 13 (Trixie)** | ✅ Supported | Native binary, FFmpeg 6 ABI |
| **macOS 13+ (Intel)** | ✅ Supported | Native binary, Homebrew FFmpeg 7+ |
| **macOS 14+ (Apple Silicon)** | ✅ Supported | Native binary, Homebrew FFmpeg 7+ |
| **Ubuntu 20.04 LTS** | ❌ Too old | glibc 2.31 < 2.35; binary won't link even with FFmpeg 4 installed. |
| **Debian 11 (Bullseye)** | ❌ Too old | glibc 2.31. |
| **RHEL 9 / Rocky 9 / Alma 9** | ⚠️ Untested | glibc 2.34. May work; runtime deps via `dnf install ffmpeg-libs tesseract leptonica freetype fontconfig`. Please [report](support.md) success/failure. |
| **RHEL 8 / Rocky 8** | ❌ Too old | glibc 2.28. |
| **CentOS 7** | ❌ Too old | glibc 2.17. End-of-life — please upgrade. |
| **Amazon Linux 2023** | ⚠️ Untested | glibc 2.34. Same situation as RHEL 9. |
| **Amazon Linux 2** | ❌ Too old | glibc 2.26. |
| **Alpine Linux** | ❌ Not supported | musl libc, not glibc. Use a glibc-based distro or Docker. |
| **Windows** | ❌ Not yet | Run via WSL 2 with Ubuntu 22.04. |

When in doubt, **use the Docker image** — it bundles all the right
runtime libraries and works on any Linux distro with Docker installed:

```bash
docker run -p 8080:8080 \
           -v synapcores-data:/var/lib/synapcores \
           -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
           ghcr.io/synapcores/community:latest
```

## Hardware sizing

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

## Runtime dependencies

The bare-metal install path runs `install-ce.sh` from the release
tarball. That script calls the bootstrap pre-flight check and then
auto-installs the runtime deps appropriate for your distro.

If you skip `install-ce.sh` (e.g., `SYNAPCORES_BINARY_ONLY=1` or your
own provisioning system), install these packages manually:

### Ubuntu 22.04 / Debian 12

```bash
sudo apt-get update
sudo apt-get install -y \
    libavutil56 libavformat58 libavcodec58 libavfilter7 libavdevice58 \
    libswresample3 libswscale5 \
    libtesseract4 libleptonica5 \
    libfreetype6 libfontconfig1 \
    ca-certificates curl
```

### RHEL 9 / Rocky 9 / Alma 9 (untested)

```bash
sudo dnf install -y epel-release   # FFmpeg lives in the RPM Fusion or EPEL repos
sudo dnf install -y \
    ffmpeg-libs tesseract leptonica \
    freetype fontconfig \
    ca-certificates curl
```

If this works for you, please [open an issue](support.md) to confirm
the package list — RHEL-family distros are currently untested and we'd
like to formalize support.

## What the installer checks

When you run `curl -fsSL https://get.synapcores.com | sh`, the bootstrap
verifies before downloading anything:

- **OS** is Linux (errors with a Docker hint on macOS, errors on anything else)
- **Architecture** is `x86_64` or `aarch64`
- **Distro** is in the supported list above (warns and exits early if not)
- **glibc** ≥ 2.35

After download, the install-ce.sh system installer additionally:

- Verifies the tarball's SHA-256 against the published checksum
- Verifies the binary self-reports as Community
- Installs runtime dependencies via apt-get (Ubuntu 22.04 / Debian 12 path)
- Creates the `synapcores` system user
- Lays out `/opt/synapcores/{synapcores,aidb_data}` and `/etc/synapcores/`
- Drops a hardened systemd unit (`NoNewPrivileges`, `ProtectSystem=strict`,
  `LimitNOFILE=65536`)

What the installer does **not** check (your responsibility):

- RAM headroom
- Disk space at the data dir
- Whether `:8080` / `:8443` are already in use
- AVX2 availability (llama-cpp will work without it but is slower)

## Local LLM inference

CE ships with **built-in local LLM inference** via the embedded
[`llama-cpp`](https://github.com/ggerganov/llama.cpp) Rust binding. The
`llm-inference` Cargo feature is on by default — there is nothing extra
to install for AI chat, NL2SQL, or embedding generation to work. CE uses
quantized GGUF models loaded in-process.

**No GPU required.** The published CE binaries are CPU-only — no CUDA,
Metal, Vulkan, or OpenCL backends are linked in.

Performance bands you can expect on a 7B-parameter Q4\_K\_M model:

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
