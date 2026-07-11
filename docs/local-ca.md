# Using the local CA (step-ca) from other devices

The stack runs a [Smallstep step-ca](https://smallstep.com/docs/step-ca/) container as a
private certificate authority. Traefik already uses it (via ACME) to issue HTTPS
certificates for every `*.microserver` service. This doc covers everything *outside*
the compose stack: trusting the CA on your laptops/phones, and issuing certificates
for other machines on the LAN.

## CA facts

| | |
|---|---|
| CA URL | `https://stepca.microserver:9000` (LAN, port published on the host) |
| Root certificate | `data/stepca/certs/root_ca.crt` on the server |
| Root SHA-256 fingerprint | `aed9753c4625b6308a3f7f1bbc87aa8f8403dae9de2914e83ad35c94b1fd16db` |
| Root valid until | 2036-07-08 |
| ACME directory | `https://stepca.microserver:9000/acme/acme/directory` |
| ACME leaf cert lifetime | 90 days (2160h, raised from step-ca's 24h default) |
| Provisioners | `acme` (no credentials, HTTP-01 challenge), `admin` (JWK, password in `data/stepca/secrets/password` on the server) |

**Prerequisite for everything below:** add a router DNS entry
`stepca.microserver` → `192.168.1.52` (same as the other `*.microserver` names —
the router has no wildcard). The CA's own certificate only carries the
`step-ca` and `stepca.microserver` names, so clients must reach it by that
hostname, not by IP.

## 1. Get the root certificate

From any device on the LAN:

```sh
curl -k https://192.168.1.52:9000/roots.pem -o root_ca.crt
```

(or `scp ice@192.168.1.52:/opt/torrentbox/data/stepca/certs/root_ca.crt .`)

The `-k` is fine *only because* you verify the fingerprint before trusting it:

```sh
openssl x509 -in root_ca.crt -noout -fingerprint -sha256
# must print AE:D9:75:3C:46:25:B6:30:...:B1:FD:16:DB (see table above)
```

## 2. Trust the root on your devices (one-time per device)

- **Debian/Ubuntu Linux**:
  ```sh
  sudo cp root_ca.crt /usr/local/share/ca-certificates/microserver.crt
  sudo update-ca-certificates
  ```
- **Fedora/RHEL**: `sudo cp root_ca.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust`
- **Arch**: `sudo trust anchor --store root_ca.crt`
- **Windows**: double-click `root_ca.crt` → Install Certificate → Local Machine →
  "Trusted Root Certification Authorities" (or `certutil -addstore -f ROOT root_ca.crt`
  from an admin prompt)
- **macOS**: `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain root_ca.crt`
- **iOS**: AirDrop/mail the file → Settings → General → VPN & Device Management →
  install profile → then Settings → General → About → Certificate Trust Settings →
  enable full trust for "microserver Root CA"
- **Android**: Settings → Security & privacy → More security → Encryption & credentials →
  Install a certificate → CA certificate. Browsers honor it; most non-browser apps
  ignore user-installed CAs (Android ≥7) — that's fine for web UIs.
- **Firefox** (uses its own store): Settings → Privacy & Security → Certificates →
  View Certificates → Authorities → Import, tick "Trust this CA to identify websites".
  (Or set `security.enterprise_roots.enabled=true` in `about:config` to use the OS store.)
- **curl/scripts ad hoc**: `curl --cacert root_ca.crt https://...`; Python requests:
  `export REQUESTS_CA_BUNDLE=/path/root_ca.crt`; Node: `export NODE_EXTRA_CA_CERTS=/path/root_ca.crt`.

Test from the device: `https://glance.microserver` should load with a green lock.

## 3. Issue certificates for other machines

Any machine on the LAN can get its own `*.microserver` certificate from the CA.
First add a router DNS entry for the new name pointing at **that machine's** IP.

### Option A — `step` CLI (simplest)

Install [`step`](https://smallstep.com/docs/step-cli/installation/), then once per machine:

```sh
step ca bootstrap --ca-url https://stepca.microserver:9000 \
  --fingerprint aed9753c4625b6308a3f7f1bbc87aa8f8403dae9de2914e83ad35c94b1fd16db
```

(bootstrap downloads and pins the root — no `-k` leap of faith needed.)

Then either:

- **ACME provisioner** (no password; 90-day certs; the machine must be reachable on
  port 80 at the requested name for the HTTP-01 challenge):
  ```sh
  sudo step ca certificate myhost.microserver myhost.crt myhost.key --provisioner acme
  ```
- **`admin` JWK provisioner** (works without port 80, but needs the CA password from
  `data/stepca/secrets/password` on the server; certs last **24h** by default — meant
  for short-lived/automated use):
  ```sh
  step ca certificate myhost.microserver myhost.crt myhost.key --provisioner admin
  ```

Renewal (before expiry, using the existing cert as credential — no challenge re-run):

```sh
step ca renew --force myhost.crt myhost.key      # one-shot, cron-able
step ca renew --daemon myhost.crt myhost.key     # or keep a renewer running
```

### Option B — any ACME client

Point your usual ACME client at the internal directory instead of Let's Encrypt.
The client machine must trust the root (step 2) and pass HTTP-01 on port 80.

- **certbot**:
  ```sh
  sudo REQUESTS_CA_BUNDLE=/usr/local/share/ca-certificates/microserver.crt \
    certbot certonly --standalone -d myhost.microserver \
    --server https://stepca.microserver:9000/acme/acme/directory
  ```
- **Caddy** (global options):
  ```
  acme_ca https://stepca.microserver:9000/acme/acme/directory
  acme_ca_root /path/to/root_ca.crt
  ```
- **acme.sh**:
  ```sh
  acme.sh --issue --standalone -d myhost.microserver \
    --server https://stepca.microserver:9000/acme/acme/directory \
    --ca-bundle /path/to/root_ca.crt
  ```
- **Traefik on another box**: same pattern as this stack's
  [compose/traefik.yml](../compose/traefik.yml) — `caserver` pointing at the directory
  URL and `LEGO_CA_CERTIFICATES=/path/root_ca.crt`.

### Verify a issued cert

```sh
openssl x509 -in myhost.crt -noout -issuer -dates
curl --cacert root_ca.crt https://myhost.microserver
```

## Operational notes

- **The CA must be up for renewals.** Traefik renews its certs at ~2/3 of the 90-day
  lifetime, so short step-ca downtime is harmless; weeks-long downtime is not.
- **`data/stepca/` contains the CA private keys** (encrypted with the password in
  `data/stepca/secrets/password`). It is gitignored — keep it that way, and include it
  in off-machine backups. Anyone with those files + password can impersonate any
  `*.microserver` site for devices that trust the root.
- **Root expiry 2036**: re-run the CA init and re-distribute the root when it gets close
  (future-you problem).
- **Revocation is passive**: step-ca supports it, but nothing on the LAN checks CRLs/OCSP
  by default. If a key leaks, rotate the cert and move on; if the *CA* leaks, delete the
  root from every device.
