# Config review — torrentbox (2026-07-11)

Review of the original single-file `docker-compose.yml` (kept as `docker-compose.yml.bak`),
with what was changed during the restructure and what remains as recommendations.

## Fixed in the restructure

| Finding | Severity | Resolution |
|---|---|---|
| Plaintext secrets in compose file (Samba password, MinIO root creds, Homarr encryption key, Twingate access+refresh tokens) | high | Moved to gitignored `.env`, referenced as `${VAR}`. Copyparty's password stays in the gitignored `config/copyparty/copyparty.conf` (committed `.example` is sanitized) |
| Traefik dashboard wide open: `--api.insecure=true` **published on host port 8080** | high | Port no longer published. Dashboard now at `https://traefik.microserver` (`api@internal` router). The insecure API port still listens *inside* the docker network only, for the Homepage widget — see recommendations |
| All web UIs served over plain HTTP | medium | step-ca local CA + Traefik ACME; every router on `websecure` with auto-issued certs; global HTTP→HTTPS redirect |
| `dperson/openvpn-client` unmaintained (~5 years), no explicit kill-switch config | high | Replaced with `qmcgaw/gluetun` (built-in kill switch, healthcheck, ProtonVPN NAT-PMP port forwarding pushed straight into qBittorrent's listen port) |
| `wernight/qbittorrent` unmaintained (last update ~6 years, old qBittorrent version) | high | Replaced with `lscr.io/linuxserver/qbittorrent`, existing profile migrated |
| `dperson/samba` unmaintained | medium | Replaced with `ghcr.io/servercontainers/samba`, same shares/ports |
| VPN container DNS `4.4.4.4` — not a Google resolver (that's 8.8.4.4); a random third party saw all torrent DNS | medium | Gone with gluetun (DNS-over-TLS inside the tunnel by default) |
| `qbittorrent` `depends_on: openvpn` without health condition — could start before the tunnel was up | medium | `depends_on: gluetun: condition: service_healthy`; gluetun also firewalls everything until the tunnel exists |
| No image pinning (`traefik`, `minio/minio` = whatever `latest` was at pull time) | medium | Pinned to major tags where the upstream publishes them: `traefik:v3`, `jellyfin:10`, `gluetun:v3`, `paperless-ngx:2`, `postgres:16-alpine`, `redis:7-alpine`. See recommendations for the rest |
| Docker socket mounted read-write into Traefik | low | `:ro` (Homepage's socket mount is `:ro` too) |
| `JELLYFIN_PublishedServerUrl` advertised http | low | Now `https://media.microserver` |
| Secrets/data/config mixed in one directory tree | low | `compose/` (definitions) vs `config/` (committed config) vs `data/` (state, gitignored) vs `downloads/` (untouched) |
| Homarr encryption key hardcoded; service replaced | — | Homarr replaced by Homepage (config-as-code, committed to git). `homarr/` appdata kept on disk until you confirm, then delete |

## Action required (you)

1. **Rotate every secret that lived in the old compose file** — it's in shell history,
   backups, and this file's git-less past: Samba password (a short numeric one — also weak),
   MinIO root password, Twingate tokens (generate a new connector token pair in the
   Twingate admin console), and ideally the copyparty password (same value as Samba's).
   Update `.env` / `copyparty.conf` after rotating.
2. **Add router DNS entries** for `stepca.microserver` and `kiwix.microserver`
   → `192.168.1.52`. stepca lets other devices request certificates from the CA
   (see `docs/local-ca.md`); kiwix needs the name resolvable before Traefik can
   pass the ACME challenge and get its certificate.
   `traefik.microserver` and `paperless.microserver` already resolve. Optionally add
   `home.microserver` if you want the dashboard off the legacy `glance.microserver`
   name (then change the router rule and `HOMEPAGE_ALLOWED_HOSTS` in
   `compose/homepage.yml`).
3. **Install the CA root cert** (`data/stepca/certs/root_ca.crt`) on your devices —
   see `docs/local-ca.md`.

## Recommendations (not done, low urgency)

- **Traefik internal API**: the Homepage traefik widget uses `http://traefik:8080`
  (unpublished, docker-network-only). If you don't care about the widget, drop
  `--api.insecure=true` and the widget for a smaller surface.
- **Docker socket proxy**: Traefik and Homepage read the docker socket; a
  `socket-proxy` (e.g. `lscr.io/linuxserver/socket-proxy`) would limit them to
  read-only container queries.
- **Replace MinIO** (upstream is dead: web console gutted from the community
  edition May 2025, repo archived April 2026). Best fit for "backup target with
  easy account management": **SFTPGo** — web admin UI for creating users, plain
  files on disk (root it under `downloads/backups/` so backups are visible via
  Samba/Copyparty), speaks WebDAV + SFTP + FTP/S + HTTPS; restic/rclone/kopia
  all support those. If a real S3 *API* turns out to be needed by some app,
  add **Garage** alongside (S3-only, opaque blob storage, no official UI);
  RustFS is still alpha, SeaweedFS is overkill for one box.
- **Remaining `latest` tags**: `minio/minio` (only date-stamped tags upstream),
  `smallstep/step-ca`, `linuxserver/qbittorrent`, `servercontainers/samba`,
  `copyparty/ac`, `gethomepage/homepage` publish no stable major tags (or move fast).
  Consider renovate-style updates or periodic manual pinning.
- **Samba guest access**: both shares allow `guest ok = yes` (kept from the old
  config). If everything on the LAN should authenticate, flip to `guest ok = no`.
- **vuetorrent UI** is a static copy from Jan 2025 (`data/qbittorrent/ui`). The
  linuxserver image supports `DOCKER_MODS=ghcr.io/vuetorrent/vuetorrent-lsio-mod`
  for an auto-updating copy (then set WebUI RootFolder to `/vuetorrent`).
- **Leftover directories**: `plex/`, `portainer/`, `flood/`, `glance/`, `homarr/`,
  `vpn/` are unused after this migration. `homarr/` and `vpn/` are kept as rollback
  material; delete all of them once the new stack has been stable for a while.
- **`bkp.tar.gz` (17 GB)** sits inside the repo dir (gitignored). Move backups to a
  different disk/host — a backup on the same filesystem doesn't survive disk failure.
- **Twingate connector tokens** in `.env` go stale after the connector rotates them
  internally; if the container ever fails auth after a long stop, issue fresh tokens.
- **`downloads/.deleted` is root-owned** (`drwx--S--- root:ice`, since Oct 2025) —
  copyparty (uid 1000) logs a PermissionError scanning it. Pre-existing, harmless;
  `sudo chown -R ice:ice downloads/.deleted` if the log noise bothers you.
- **Network layout**: web services sit on `torrentbox_proxy`; paperless db/redis are
  on an `internal: true` network; samba/twingate live on the default network with no
  Traefik exposure. Jellyfin/copyparty could additionally be split from each other,
  but on a single-user LAN box that's diminishing returns.
