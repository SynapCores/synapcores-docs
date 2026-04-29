# Quickstart

This is the operator-facing install + first-run guide for SynapCores
Community Edition. For the CE/EE feature boundary, see
[Community vs Enterprise](enterprise.md). For sizing, see
[Hardware requirements](requirements.md).

## Install on Linux

```bash
curl -fsSL https://get.synapcores.com | sh
```

That script does three things: detects your platform, downloads the
matching release tarball, verifies its SHA-256, and hands off to the
local installer (`install-ce.sh`) which:

- creates the `synapcores` system user
- lays out `/opt/synapcores/{synapcores,aidb_data}` and `/etc/synapcores/gateway.toml`
- installs a hardened `synapcores.service` systemd unit (NoNewPrivileges,
  ProtectSystem=strict, ReadWritePaths scoped to the data dir)
- enables the service but does **not** start it (you decide when)

To install the binary only, without touching systemd or system users:

```bash
curl -fsSL https://get.synapcores.com | SYNAPCORES_BINARY_ONLY=1 sh
```

To pin a specific release:

```bash
curl -fsSL https://get.synapcores.com | SYNAPCORES_VERSION=v1.0.0 sh
```

## First run

1. Set a JWT secret (the gateway will not start without one):

   ```bash
   sudo systemctl edit synapcores
   ```

   Add:

   ```
   [Service]
   Environment="AIDB_JWT_SECRET=<paste a 256-bit random string here>"
   ```

2. (Optional) enable TLS by editing `/etc/synapcores/gateway.toml`:

   ```toml
   [server.tls]
   enabled  = true
   cert_file = "/etc/synapcores/certs/server.crt"
   key_file  = "/etc/synapcores/certs/server.key"
   ```

3. Start it:

   ```bash
   sudo systemctl start synapcores
   sudo systemctl status synapcores
   ```

4. Tail logs:

   ```bash
   sudo journalctl -u synapcores -f
   ```

   The first line you'll see is the edition banner:

   ```
   Starting SynapCores Community edition (v0.1.0, build 275, git ...)
   License loaded: id=lic_community_default tier=Community edition=Community customer=(none)
   ```

5. Sanity check the version:

   ```bash
   /opt/synapcores/synapcores --version
   # synapcores 0.1.0 Community (build 275, abc1234)
   ```

6. **Capture the first-boot admin password.** It's printed exactly
   once to the gateway logs and never shown again:

   ```bash
   sudo journalctl -u synapcores -n 200 --no-pager | grep -A 7 "FIRST-BOOT"
   ```

   Save the password somewhere safe — you'll need it for first
   login.

7. **Open the Web UI.** Point a browser at the gateway's HTTP port:

   ```
   http://<your-host>:8080/
   ```

   Or `https://<your-host>:8443/` if you enabled TLS in step 2. Log
   in as `admin` with the password you just captured. Change it
   immediately under **Settings → Account**.

   See [Web UI](web-ui.md) for the full feature tour.

8. **Try Filesystem Collections.** Point a folder at the gateway and
   watch CSVs, PDFs, images, and audio get ingested and indexed
   automatically. See [Filesystem Collections](filesystem-rag.md) for
   the walkthrough.

## Docker

```bash
docker run --rm -p 8080:8080 \
           -v synapcores-data:/var/lib/synapcores \
           -e AIDB_JWT_SECRET="$(openssl rand -base64 32)" \
           ghcr.io/synapcores/community:latest
```

The image runs as the unprivileged `synapcores` user, with data persisted
to the `synapcores-data` volume. Mount that volume to keep state across
upgrades.

## Upgrading

CE upgrades are in-place: re-run the installer, restart the service.
Data on `/opt/synapcores/aidb_data` (or `/var/lib/synapcores` in Docker)
is portable across CE versions.

```bash
curl -fsSL https://get.synapcores.com | sh
sudo systemctl restart synapcores
```

## Migrating CE → Enterprise

1. Stop the service.
2. Replace the binary with the Enterprise build.
3. Drop your signed license file at `/etc/synapcores/license.json`.
4. Start the service. The boot banner now reads:
   `Starting SynapCores Enterprise edition ...`
   `License loaded: id=lic_<your-license-id> tier=Enterprise ...`

Data carries forward unchanged — the storage format is identical
between editions. Existing tables, vectors, and dashboards keep working.

## Where to ask for help

- Documentation: https://docs.synapcores.com
- Community forum: https://community.synapcores.com
- Issues / bugs: https://github.com/mataluis2k/aidb/issues
- Enterprise sales: sales@synapcores.com
