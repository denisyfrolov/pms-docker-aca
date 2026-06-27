# pms-docker-aca — Plex Media Server for Azure Container Apps

A fork of the official [plexinc/pms-docker](https://github.com/plexinc/pms-docker) image, adapted to run reliably on **Azure Container Apps (ACA)** and similar platforms where the container filesystem is **ephemeral** and the instance is **restarted/redeployed frequently**.

On ACA the `/config` directory does not survive a restart, which breaks Plex in subtle ways: the server loses its identity/certificate (clients report *"unable to connect securely"*), SSH host keys change, and the databases are wiped. This edition keeps a small, durable **state volume** and restores the essential pieces on every boot — without persisting the entire (potentially huge) `/config` tree.

## The idea

A single durable volume is mounted at **`/config-defaults`** (e.g. an **Azure Files** share). The container treats it as the source of truth for Plex's *identity and data*:

- On **startup** (`cont-init.d`), the essential state is **restored** from `/config-defaults` into the ephemeral `/config` **before** Plex starts.
- While **running**, changes are **saved back** to `/config-defaults` (file watchers / on-change copies).
- On **shutdown** (s6 `finish`), the databases are backed up after Plex has fully stopped.

Everything else in `/config` (caches, transcoder temp, the bulk of the metadata/artwork library) is allowed to remain ephemeral, keeping the persisted footprint small.

## What gets persisted

All of the following live under the `/config-defaults` volume:

| Item | Saved by | Restored by | Why it matters |
| --- | --- | --- | --- |
| `Preferences.xml` | `prefFile-watch` service | `35-plex-restore` | Server identity: `MachineIdentifier`, `PlexOnlineToken`, `CertificateUUID`. |
| `cert-v2.p12` (plex.direct certificate) | `prefFile-watch` service | `35-plex-restore` | The signed `*.<hash>.plex.direct` cert + key. Reusing it stops the *"unable to connect securely"* errors that appear after repeated restarts (Plex otherwise re-acquires a rate-limited certificate every boot). |
| `ssh-host-keys/` | `30-sshd-setup` (first generation) | `30-sshd-setup` | Stable SSH host keys so tunneling clients don't get host-key-mismatch warnings. |
| `Backup/Databases.bak.*.zip` | `plex-db-backup` (on shutdown) | `36-db-restore` | The Plex databases (watch state, library metadata DB). |

`Preferences.xml` and `cert-v2.p12` are a matched pair (the cert's hash equals the `CertificateUUID`), so they are always persisted and restored **together**. Both the pref and cert copies use timestamped `--backup` suffixes, so a known-good copy is never overwritten in place.

## Added components

These are layered on top of the upstream image via the s6-overlay tree under `root/`:

- **SSH server (`sshd`)** — key-only SSH for port-forwarding/tunneling, enabled only when `SSH_AUTHORIZED_KEYS` is set. Useful for reaching `localhost:32400` to perform Plex first-run setup on a headless/cloud host.
  - `root/etc/cont-init.d/30-sshd-setup` — installs the authorized key, restores/generates persistent host keys, and writes a tunneling-focused `sshd_config`.
  - `root/etc/services.d/sshd/run` — runs `sshd` (idles if SSH is disabled).
- **HTTP healthcheck endpoint** — a tiny `socat`-based listener that returns `{"status":"ok"}`, suitable for an ACA health probe. Defaults to port `9000`.
  - `root/etc/services.d/healthcheck-http/run` and `root/usr/local/bin/healthcheck-respond`.
- **Identity & data persistence** — the restore/backup scripts described in the table above (`35-plex-restore`, `36-db-restore`, `prefFile-watch`, `plex-db-backup`).
- **Graceful shutdown + backup** — `root/etc/services.d/plex/finish` stops Plex, waits for it to exit, then runs the database backup. s6 grace times are raised so the backup fits inside the platform shutdown window.

## Image changes (Dockerfile)

- Extra packages: `openssh-server`, `socat`, `zip`, `unzip` (plus the upstream `curl`, `xmlstarlet`, `uuid-runtime`, `unrar`).
- `EXPOSE` adds `22/tcp` (SSH) and `9000/tcp` (healthcheck) alongside the standard Plex ports.
- `ENV` adds `S6_SERVICES_GRACETIME="120000"` and `S6_KILL_GRACETIME="120000"` (120 s) so the shutdown backup has time to complete.

## Environment variables

In addition to the upstream variables (`TZ`, `PLEX_CLAIM`, `PLEX_UID`, `PLEX_GID`, `ADVERTISE_IP`, `ALLOWED_NETWORKS`, `CHANGE_CONFIG_DIR_OWNERSHIP`, …):

| Variable | Default | Description |
| --- | --- | --- |
| `SSH_AUTHORIZED_KEYS` | _(unset)_ | When set to a public key, enables the SSH service for the `plex` user (key-only, no passwords, no root). When unset, SSH stays disabled. |
| `HEALTHCHECK_HTTP_PORT` | `9000` | Port for the HTTP healthcheck endpoint. |
| `DEBUG` | `false` | When `true`, enables `set -x` tracing in the helper scripts. |

## Volumes

| Mount | Purpose |
| --- | --- |
| `/config-defaults` | **Durable** state volume (e.g. Azure Files). Holds the persisted identity, certificate, SSH host keys, and database backups. This is the volume that must survive restarts. |
| `/config` | Plex data dir. Ephemeral on ACA; restored from `/config-defaults` on boot. |
| `/config/Library/Application Support/Plex Media Server/Metadata` | Optional **durable** volume for posters/artwork bundles. The databases only store references into this dir, so without persisting it, restored libraries show blank posters. Mount it separately when you want artwork to survive restarts. |
| `/transcode` | Optional transcoder temp dir. |
| `/data` | Optional media. |

## Quick start (local / docker compose)

A minimal `docker-compose.yml` is included. The key addition versus upstream is the durable `/config-defaults` mount:

```yaml
services:
  plex:
    build: .
    environment:
      - TZ=Etc/UTC
      - PLEX_CLAIM=claim-xxxxxxxx
      - SSH_AUTHORIZED_KEYS=ssh-ed25519 AAAA... you@example.com
    network_mode: host
    volumes:
      - plex-config-defaults:/config-defaults
      - plex-metadata:/config/Library/Application Support/Plex Media Server/Metadata
```

On Azure Container Apps, back `/config-defaults` with an **Azure Files** volume and set the platform **`terminationGracePeriodSeconds`** generously (e.g. matching the s6 grace times) so the shutdown database backup can finish.

## First-run on a headless / cloud host (SSH tunnel)

Plex only exposes its setup wizard over `http://localhost:32400/web`. To reach it on a remote/cloud host, enable SSH (`SSH_AUTHORIZED_KEYS`) and tunnel:

```
ssh plex@<host> -L 32400:localhost:32400 -N
```

Then open `http://localhost:32400/web` locally.

## Verifying the certificate is reused

The server presents the persisted certificate on its TLS port; the subject is `*.<hash>.plex.direct` where `<hash>` equals your `CertificateUUID`:

```bash
echo | openssl s_client -connect 127.0.0.1:32400 2>/dev/null \
  | openssl x509 -noout -subject -fingerprint -sha256
```

If the SHA-256 fingerprint is unchanged across restarts, Plex is reusing the persisted `cert-v2.p12` instead of re-acquiring one — which is the desired behavior.

---

This image is based on the official Plex Docker image; see the upstream [plexinc/pms-docker](https://github.com/plexinc/pms-docker) for the full set of base features, networking modes (`bridge`/`host`/`macvlan`), and hardware-transcoding details.
