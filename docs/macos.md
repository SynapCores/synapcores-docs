# Run on macOS

CE v1.0 ships **Linux binaries only**. macOS native binaries are
deferred to v1.1 (the multimedia subsystem needs to migrate to the
FFmpeg 5+ API before the macOS build is unblocked — tracked separately).

In the meantime, two well-supported paths get you a working CE
deployment on macOS in minutes. Pick whichever matches your existing
toolchain.

## Option A — Multipass (recommended)

[Multipass](https://multipass.run) runs lightweight Ubuntu VMs natively
on macOS using Apple's Virtualization framework (Apple Silicon) or
HyperKit (Intel). Boots in seconds, uses far less RAM than Docker
Desktop, no background daemon to manage.

### One-time setup

```bash
brew install --cask multipass
```

### Launch an Ubuntu VM and install CE

```bash
# Spin up a 22.04 VM sized for CE (8 GB RAM, 4 vCPU, 20 GB disk).
multipass launch 22.04 --name synapcores --memory 8G --cpus 4 --disk 20G

# Drop into a shell on the VM.
multipass shell synapcores

# Now inside the VM — install CE the standard way:
curl -fsSL https://get.synapcores.com/install.sh | sh
```

The installer creates the `synapcores` user, lays out
`/opt/synapcores`, drops the systemd unit, and prints the auto-
generated admin password. Capture it from `journalctl -u synapcores`.

### Start the service and grab the VM's IP

Inside the VM:

```bash
sudo systemctl edit synapcores
# Add:
#   [Service]
#   Environment="AIDB_JWT_SECRET=$(openssl rand -base64 32)"
sudo systemctl start synapcores
exit                                # back to your Mac
```

Back on macOS:

```bash
multipass info synapcores | grep IPv4
# IPv4:           192.168.64.7
```

Open `http://192.168.64.7:8080/` in your Mac browser. The bundled Web
UI loads. Log in as `admin` with the password from the VM's logs.

### Lifecycle

```bash
# Stop without deleting (keeps data):
multipass stop synapcores

# Resume:
multipass start synapcores

# Tear down completely (deletes all data):
multipass delete synapcores
multipass purge
```

### Pros / cons

**Pros:**
* No daemon process consuming RAM when not in use
* Apple's virtualization framework means near-native disk and network throughput on M-series chips
* Snapshots and shell access feel natural
* The CE binary runs unchanged — same as a real Linux deployment

**Cons:**
* You're managing a VM's lifecycle (stop/start/snapshot)
* No volume mounts to the Mac filesystem by default (use
  `multipass mount /local/path synapcores:/remote/path` if needed)

## Option B — Docker

If you already have Docker Desktop, this is one-line and probably
faster to set up than installing Multipass.

### Run the official image

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
| Already have Docker Desktop running for other projects | **Docker** |
| Want a "real Linux" experience for ops / debugging | **Multipass** |
| Care about RAM headroom | **Multipass** |
| Need GPU passthrough (Apple Silicon) for inference | Neither — wait for v1.1 native macOS, or use a remote Linux box |

For a polished demo or evaluation on a MacBook, Multipass usually
feels lighter — you get a real shell on a real Linux box and you can
treat it the same as a deployment target.

## Apple Silicon vs Intel

Both options work on both architectures. Multipass auto-selects the
right virtualization backend (Apple Virtualization on M-series, HyperKit
on Intel). The CE Linux/aarch64 binary runs natively on M-series VMs;
on Intel Macs you'll get the linux-x86_64 binary. Either way, no
emulation, no Rosetta in the path.
