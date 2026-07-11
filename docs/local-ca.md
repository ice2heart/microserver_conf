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
- **`admin` JWK provisioner** (works without port 80; needs the CA password from
  `data/stepca/secrets/password` on the server). Its provisioner claims in
  `data/stepca/config/ca.json` allow up to **1 year** (`maxTLSCertDuration: 8760h`),
  defaulting to 90 days — good for appliances you can't easily re-issue. Add
  `--not-after 8760h` for the full year:
  ```sh
  step ca certificate myhost.microserver myhost.crt myhost.key \
    --provisioner admin --not-after 8760h
  ```
  > The `secrets/password` file has a trailing newline; some `step` versions read it
  > literally and fail with `invalid password`. Strip it first:
  > `printf '%s' "$(cat data/stepca/secrets/password)" > /tmp/pw` and pass `--password-file /tmp/pw`.

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

### Option C — appliance that generates its own key (iLO, IPMI, switches, NAS)

Some devices never export their private key: they generate it internally and only give
you a **CSR** to sign. You sign the CSR on the server with the `admin` provisioner and
paste the resulting cert back into the device. The key never leaves the appliance.

Worked example — **HPE iLO** (Security → SSL Certificate):

1. In iLO, fill in the certificate fields and **set Common Name = `ilo.microserver`**
   (must match the router DNS name), then generate the CSR. iLO makes an RSA-2048 key
   internally — the CSR carries iLO's public key, so the signed cert is RSA regardless
   of the CA's own key type. (iLO 4 / older iLO 5 reject ECDSA, so this is the right path.)
2. Copy the CSR PEM to the server as `ilo.csr` and sign it (runs in the `step-ca`
   container, which has the `step` CLI; the host does not):
   ```sh
   docker cp ilo.csr step-ca:/tmp/ilo.csr
   docker exec step-ca sh -c '
     cd /tmp
     printf "%s" "$(cat /home/step/secrets/password)" > /tmp/pw   # strip trailing newline
     step ca sign ilo.csr ilo.crt --provisioner admin --password-file /tmp/pw \
       --not-after 8760h \
       --ca-url https://step-ca:9000 --root /home/step/certs/root_ca.crt --force
     rm -f /tmp/pw'
   docker cp step-ca:/tmp/ilo.crt ./ilo.crt
   docker exec step-ca rm -f /tmp/ilo.csr /tmp/ilo.crt
   ```
   > If the CSR's CN/SAN is the device serial (e.g. `ILOCZ14200051`) instead of
   > `ilo.microserver`, either regenerate the CSR in iLO with the right CN, or add the
   > name at signing time: `--san ilo.microserver`. `step ca sign` sets the SANs it's
   > given (or the CSR's if none) — the cert **must** carry `ilo.microserver` or browsers
   > reject the hostname.
3. `ilo.crt` contains leaf + intermediate. Paste **only the first (leaf) block** into
   iLO's *Import Certificate* field — iLO takes a single cert, not a chain. It reboots
   the management processor (~30 s), then `https://ilo.microserver` is green on any
   device that trusts the root (§2). Ensure the router has `ilo.microserver` → iLO's IP.

To re-issue before/after expiry, just repeat from step 1 (iLO keeps its key, so the same
CSR can be re-signed, or generate a fresh one). Certs from `admin` last up to 1 year.

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
- **`admin` provisioner claims** in `data/stepca/config/ca.json` were raised to
  `maxTLSCertDuration: 8760h` (1 year) so appliance certs don't need frequent re-issue.
  If the admin provisioner ever reports `failed to decrypt JWE: invalid password` despite
  the right password, its JWK was encrypted with a stale password: regenerate it with
  `step crypto jwk create` using `secrets/password`, splice the new public key
  (`kid`/`x`/`y`) and the compact-JWE `encryptedKey` into the provisioner, then
  `docker kill --signal=HUP step-ca` to hot-reload. Keep a `ca.json.bak` first.
