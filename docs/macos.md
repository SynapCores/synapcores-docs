# Run on macOS

Starting with **v1.3.0-ce**, SynapCores Community Edition ships **native
macOS binaries** for both Intel and Apple Silicon Macs running macOS 13
(Ventura) or later. Native install is now the recommended path. Docker
remains as a secondary option for users on older macOS releases or who
prefer container-based deployment.

## Option A — Native binary (recommended)

The same one-liner used on Linux works on macOS 13+:

```bash
curl -fsSL https://get.synapcores.com | sh
```

The bootstrap detects your OS and CPU architecture and downloads the
matching tarball:

- macOS Intel → `synapcores-ce-VER-darwin-x86_64.tar.gz`
- macOS Apple Silicon → `synapcores-ce-VER-darwin-aarch64.tar.gz`

### Runtime dependencies

The macOS binary is dynamically linked against Homebrew-provided
FFmpeg 7+, Tesseract, Leptonica, FreeType, and Fontconfig. The
bootstrap installer does **not** auto-install these for you on
macOS — install them manually before first run:

```bash
brew install ffmpeg tesseract leptonica freetype fontconfig
```

If any of those are missing, the gateway will fail to start with a
dyld load error naming the missing library.

### First run

There is no systemd on macOS. The two supported ways to run the
gateway are:

**Foreground (development / testing):**

```bash
export AIDB_JWT_SECRET="$(openssl rand -base64 32)"
/usr/local/bin/synapcores --config /usr/local/etc/synapcores/gateway.toml
```

**Background via `launchd` (long-lived install):**

Drop a LaunchAgent plist at
`~/Library/LaunchAgents/com.synapcores.gateway.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.synapcores.gateway</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/synapcores</string>
    <string>--config</string>
    <string>/usr/local/etc/synapcores/gateway.toml</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>AIDB_JWT_SECRET</key>
    <string>REPLACE_WITH_256_BIT_RANDOM_STRING</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/usr/local/var/log/synapcores.log</string>
  <key>StandardErrorPath</key>
  <string>/usr/local/var/log/synapcores.err</string>
</dict>
</plist>
```

Load it:

```bash
launchctl load ~/Library/LaunchAgents/com.synapcores.gateway.plist
```

Tail logs:

```bash
tail -f /usr/local/var/log/synapcores.log
```

Capture the first-boot admin password from that log file the same way
you would from `journalctl` on Linux — search for `FIRST-BOOT`.

### Apple Silicon vs Intel

Both architectures are first-class. The bootstrap auto-selects the
correct tarball — no Rosetta in the path on Apple Silicon.

## Option B — Docker

For users on macOS 12 or earlier, or who simply prefer containerized
deployment, the official Docker image still works:

```bash
docker run --rm -p 8080:8080 \
           -v synapcores-data:/var/lib/synapcores \
           -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
           ghcr.io/synapcores/community:latest
```

That mounts a Docker-managed named volume for persistence (so data
survives container removal) and exposes the Web UI on
[http://localhost:8080/](http://localhost:8080/).

### Capture the admin password

The first-boot banner is in the container's stdout — `docker logs`
doesn't help if you used `--rm`. For a first-time install where you
need the password, drop `--rm` and run detached:

```bash
docker run -d --name synapcores -p 8080:8080 \
           -v synapcores-data:/var/lib/synapcores \
           -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
           ghcr.io/synapcores/community:latest

docker logs synapcores 2>&1 | grep -A 7 "FIRST-BOOT"
```

Or pin a known password up front so you don't have to harvest it from
logs:

```bash
docker run -d --name synapcores -p 8080:8080 \
           -v synapcores-data:/var/lib/synapcores \
           -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
           -e AIDB_ADMIN_PASSWORD="<your password, 12+ chars>" \
           ghcr.io/synapcores/community:latest
```

### Pros / cons

**Pros:**

* Works on macOS 12 and earlier (no native binary available there)
* One-line setup if Docker Desktop is already running
* Familiar tooling for most developers

**Cons:**

* Docker Desktop on macOS has a non-trivial RAM and CPU baseline even
  when idle
* The licensing of Docker Desktop changed for some company sizes —
  check before using at work

## Which should I pick?

| If you... | Use |
| --- | --- |
| Are on macOS 13+ and want the lightest install | **Native binary** |
| Care about RAM headroom (no Docker Desktop daemon) | **Native binary** |
| Are on macOS 12 or earlier | **Docker** |
| Already have Docker Desktop running for other projects | **Docker** |
| Need GPU passthrough for inference | Build from source with `--features llama-cpp/metal` |

For most evaluators on a current MacBook, the native binary is now the
fastest and cleanest path to a running gateway.
