# Music stack: beets, Navidrome, Lidarr, Prowlarr, FlareSolverr

Defined in [compose/music.yml](../compose/music.yml). Adds automated music
collection management (Lidarr + Prowlarr, wired to the existing qBittorrent),
a tagger/organizer for the pre-existing messy `downloads/music/` tree (beets,
via the beets-flask web UI), a streaming server (Navidrome) over the
cleaned-up result, and FlareSolverr (a Cloudflare-solving proxy Prowlarr can
hand off Cloudflare-protected indexers to).

None of the cross-app wiring below is compose-configurable: Lidarr and
Prowlarr each generate their own API key on first boot (stored in their
`config.xml` under `data/<service>/`), so the integration has to be done
once through each app's web UI after `docker compose up -d`.

## One-time setup before first boot

```sh
mkdir -p downloads/music-library
chown 1000:1000 downloads/music-library
```

**Navidrome and beets-flask run directly as `user: "1000:1000"` (no
root-then-drop-privileges entrypoint like the linuxserver.io images have),
so they cannot fix ownership on their own bind mounts.** If `data/navidrome`
or `config/beets` get auto-created by Docker on first `up`, they come out
owned by `root:root` and both containers crash-loop (Navidrome: `unable to
open database file`) or 500 (beets-flask: `PermissionError: [Errno 13]
Permission denied: '/config/beets'`) until fixed:

```sh
sudo chown 1000:1000 data/navidrome
sudo chown -R 1000:1000 config/beets
docker compose restart navidrome beets-flask
```

Lidarr and Prowlarr don't need this — their linuxserver.io images `chown`
their own `/config` as root before switching to the `PUID`/`PGID` user.

Add router DNS entries (no wildcard on this LAN — see
[config-review.md](config-review.md)) for the five new hostnames, pointing
at `192.168.1.52`: `beets.microserver`, `navidrome.microserver`,
`lidarr.microserver`, `prowlarr.microserver`, `flaresolverr.microserver`.

## Data flow

```
downloads/music/            existing messy inbox (untouched structure)
        |  beets-flask watches this, import-preview + tag
        v
downloads/music-library/    clean, tagged output -- shared by beets AND
        |                   Lidarr's root folder
        +--> Navidrome (read-only mount, streams from here)
```

Lidarr and beets-flask both write into `downloads/music-library/`. This is
the simple option (vs. giving each tool its own folder) — if the two start
fighting over renames, split them into separate target folders later.

## Wiring Prowlarr → Lidarr → qBittorrent

1. **Prowlarr** (`https://prowlarr.microserver`): set an admin login on
   first load. Then:
   - *Settings → Indexers → Indexer Proxies → Add → FlareSolverr* — host
     `http://flaresolverr:8191`, leave tags empty for now. Only needed for
     indexers that return a Cloudflare challenge page; tag those specific
     indexers with this proxy under *Settings → Indexers → (indexer) →
     Tags* rather than applying it globally.
   - *Settings → Indexers → Add Indexer* — add the torrent indexers you use.
   - *Settings → Download Clients → Add → qBittorrent* — host `gluetun`,
     port `8080`, qBittorrent WebUI credentials (qBittorrent has no network
     identity of its own; it shares gluetun's namespace, see
     [compose/vpn.yml](../compose/vpn.yml)).
   - *Settings → Apps → Add → Lidarr* — host `lidarr`, port `8686`, paste
     in Lidarr's API key (*Lidarr → Settings → General → Security*, visible
     after Lidarr's first boot). This pushes Prowlarr's indexer list into
     Lidarr automatically and keeps it in sync going forward.
2. **Lidarr** (`https://lidarr.microserver`):
   - *Settings → Download Clients → Add → qBittorrent* — same host/port/
     credentials as above (`gluetun:8080`).
   - *Settings → Media Management* — confirm the root folder is `/music`
     (maps to `downloads/music-library/` on the host).
   - Indexers should already be present via the Prowlarr sync from step 1;
     add manually if not.

## beets config

beets-flask writes its default config templates into `./config/beets/` on
first launch. After first `docker compose up -d beets-flask`:

1. Inspect what was generated under `config/beets/`.
2. Edit `config.yaml` (MusicBrainz settings, path format, plugins) and the
   inbox definition — set `path: /music/inbox`, and start with
   `autotag: preview` rather than `auto` given the size of the existing
   backlog, so nothing gets silently reorganized without a look first.
   Switch to `auto` for new arrivals once the backlog is triaged.
3. Restart the container — config changes only apply on restart.
4. Commit the resulting `config/beets/config.yaml` to git (non-secret,
   matches the `config/<service>` convention used elsewhere, e.g.
   `config/copyparty`).

## Navidrome

No pre-seedable admin env var — visit `https://navidrome.microserver` after
first boot and create the admin account through the setup screen.

## Homepage

FlareSolverr gets a plain link tile (`config/homepage/services.yaml`, under
*Downloads*) — no API key, since it has no widget type in Homepage.

### Homepage widget credentials

Add these to `.env` (template in [.env.example](../.env.example)) so the
Lidarr/Prowlarr/Navidrome dashboard tiles show live data instead of widget
errors:

```
# Lidarr API key: Settings -> General -> Security
HOMEPAGE_VAR_LIDARR_KEY=

# Prowlarr API key: Settings -> General -> Security
HOMEPAGE_VAR_PROWLARR_KEY=

# Navidrome: create a dedicated user for the widget (Settings -> Users), or reuse admin
HOMEPAGE_VAR_NAVIDROME_USERNAME=
HOMEPAGE_VAR_NAVIDROME_PASSWORD=
```

All four apps generate their API key only after first boot, so these can't
be filled in until Lidarr/Prowlarr have been started at least once (see
*Wiring* above for where to find each key). After editing `.env`, **recreate**
homepage (not just restart — `restart` reuses the environment the container
already has baked in; only `up -d` re-reads `.env`):

```sh
docker compose up -d homepage
```

## Verify

```sh
docker compose config                       # validate merged compose
docker compose up -d beets-flask navidrome prowlarr lidarr flaresolverr
docker compose logs -f lidarr                # watch for clean startup

# confirm lidarr/prowlarr can reach qbittorrent through gluetun
docker exec lidarr wget -qO- http://gluetun:8080   # expect a login page, not a connection error

# confirm prowlarr can reach flaresolverr
docker exec prowlarr wget -qO- http://flaresolverr:8191   # expect a small HTML status page
```
