# torrentbox

Home server stack for `microserver` (192.168.1.52), managed with Docker Compose.
Every service is reverse-proxied by Traefik with HTTPS certificates issued by a
local [step-ca](https://smallstep.com/docs/step-ca/) ACME authority.

| Service | URL | What |
|---|---|---|
| Homepage | https://glance.microserver | dashboard |
| qBittorrent | https://flood.microserver | torrents, tunnelled through ProtonVPN (gluetun) |
| Jellyfin | https://media.microserver | media server |
| Copyparty | https://fs.microserver | file browser over `downloads/` |
| MinIO | https://minio.microserver (S3) / https://minioadmin.microserver (console) | object storage |
| Paperless-ngx | https://paperless.microserver | document archive |
| Traefik | https://traefik.microserver | proxy dashboard |
| Samba | `\\192.168.1.52` (`Mount`, `Downloads`) | SMB shares |
| Twingate | — | remote access connector |

DNS for `*.microserver` names is served by the router (192.168.1.1) — each new
hostname needs an entry there pointing at `192.168.1.52`.

## Layout

```
docker-compose.yml   entry point: just include:s the files below
compose/             one file per service
config/              committed service configs (secrets referenced via env)
data/                app state — gitignored
downloads/           bulk media — gitignored, never touched by the tooling
.env                 all secrets — gitignored; template in .env.example
```

## Bootstrap on a fresh machine

1. `cp .env.example .env` and fill in secrets;
   `cp config/copyparty/copyparty.conf.example config/copyparty/copyparty.conf`
   and set the account password.
2. `docker compose up -d step-ca` — first start initializes the CA into
   `data/stepca/` (root cert, keys, ACME provisioner).
3. Optional but recommended: lengthen ACME cert lifetime so renewals are ~90 days
   instead of daily — add to the ACME provisioner in `data/stepca/config/ca.json`:
   `"claims": {"maxTLSCertDuration": "2160h", "defaultTLSCertDuration": "2160h"}`,
   then `docker compose restart step-ca`.
4. `docker compose up -d` — everything else.

## Trusting the CA on your devices (one-time)

The CA root certificate is `data/stepca/certs/root_ca.crt`, also served at
`https://192.168.1.52:9000/roots.pem`. Per-OS install instructions, fingerprint
verification, and how to issue certificates for *other* machines on the LAN
(step CLI, certbot, Caddy, ...) are in [docs/local-ca.md](docs/local-ca.md).

## Operations

```sh
docker compose up -d                 # apply config changes
docker compose logs -f gluetun      # watch the VPN (forwarded port shows here)
docker compose pull && docker compose up -d   # update images
```

- qBittorrent's listen port is set automatically from ProtonVPN's forwarded port
  (NAT-PMP) on every reconnect — no manual port juggling.
- qBittorrent WebUI auth: browsers on the LAN (192.168.1.0/24) skip the login
  (Traefik passes the real client IP via reverse-proxy headers; qBittorrent trusts
  proxies on the docker subnet); anything else needs the login from `.env`
  (`HOMEPAGE_VAR_QBT_*`) — Homepage's widget uses the same credentials.
- Paperless: drop files into `data/paperless/consume/` (or the web UI) to ingest.
- Homepage widgets need API credentials in `.env` (`HOMEPAGE_VAR_*`); they show
  errors until filled in.

See [docs/config-review.md](docs/config-review.md) for the security review and
open recommendations (secret rotation!).
