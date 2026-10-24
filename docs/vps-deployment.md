---
title: Deploy on a single VPS
order: 15
audience: operator
---

# Deploy on a single VPS

This procedure runs the published API, web app and optional worker on one Linux/amd64 Debian or Ubuntu VPS, with PostgreSQL and host Caddy for HTTPS. It uses `/opt/uzi` and the Compose project `uzi`. Commands below run in a root shell (`sudo -i`); keep SSH access available while changing firewall rules. This is an operator procedure, not an automated deployment or backup service.

## 1. Prepare the host and DNS

Choose a supported Debian/Ubuntu release and an amd64 VPS. Budget for the database and web services **in addition to** the worker's default 2 CPUs / 4 GiB limit ([docker-compose.yml:348–349](../docker-compose.yml#L348)); see [worker sizing](./worker-setup.md#resource-stats-and-sizing). A starting host recommendation is 4 vCPUs, 8 GiB RAM and 60 GiB SSD; size storage and concurrency for your repositories.

Point your domain's DNS A record at the VPS. Add an AAAA record only if IPv6 reaches this host. Allow inbound TCP 80 and 443 in the provider firewall and host firewall, retaining your SSH rule; do not expose 8080 or PostgreSQL. Outbound access must reach the package repositories, GitHub/GHCR, certificate authority, forge and model services. See [Caddy's automatic HTTPS requirements](https://caddyserver.com/docs/automatic-https#overview) and [Docker's firewall caveats](https://docs.docker.com/engine/install/ubuntu/#firewall-limitations).

Install base tools, then use Docker's official apt repository (see the supported distributions and prerequisites for [Debian](https://docs.docker.com/engine/install/debian/) or [Ubuntu](https://docs.docker.com/engine/install/ubuntu/)). On a fresh host:

```sh
test "$(uname -m)" = x86_64
apt-get update
apt-get install -y ca-certificates curl git openssl gnupg debian-keyring debian-archive-keyring apt-transport-https
. /etc/os-release
case "$ID" in debian|ubuntu) ;; *) echo 'Use Debian or Ubuntu' >&2; exit 1 ;; esac
install -m 0755 -d /etc/apt/keyrings
curl -fsSL "https://download.docker.com/linux/$ID/gpg" -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/$ID
Suites: $VERSION_CODENAME
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker version
docker compose version
```

Use the Compose plugin (`docker compose`, v2 or newer), not legacy `docker-compose`. Install host Caddy from its [official Debian/Ubuntu package repository](https://caddyserver.com/docs/install#debian-ubuntu-raspbian):

```sh
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' -o /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg /etc/apt/sources.list.d/caddy-stable.list
apt-get update
apt-get install -y caddy
```

Install a current `cosign` using the [official installation instructions](https://docs.sigstore.dev/cosign/system_config/installation/), then check `cosign version`. Version 3.1.3 or newer is recommended by [container signing](./container-signing.md#verifying-manually); complete this before pulling application images.

## 2. Choose and verify one published release

Choose an existing [published release](https://github.com/vtmocanu/uzi/releases) and use it throughout. `v0.85.2` below illustrates naming, **not the latest release recommendation**; substitute your chosen tag. The image version drops the leading `v` ([release.yml:72](../.github/workflows/release.yml#L72)). The published [API](../.github/workflows/release.yml#L290), [web](../.github/workflows/release.yml#L356) and [agent template](../.github/workflows/release.yml#L572) tags use that version; their platforms are Linux/amd64 ([API:279](../.github/workflows/release.yml#L279), [web:345](../.github/workflows/release.yml#L345), [agent:562](../.github/workflows/release.yml#L562)).

```sh
RELEASE=v0.85.2
VERSION=${RELEASE#v}
git clone --branch "$RELEASE" --depth 1 https://github.com/vtmocanu/uzi.git /opt/uzi
cd /opt/uzi
```

Verify each image before use. These are the command and identity expression from [container-signing.md:49–52](./container-signing.md#verifying-manually), with the chosen version substituted and repeated for the other two images. Keep the single backslashes in the shell single-quoted expression. Stop if verification fails.

```sh
cosign verify \
  --certificate-identity-regexp '^https://github\.com/vtmocanu/uzi/\.github/workflows/release\.yml@refs/tags/.*$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/vtmocanu/uzi/api:<version>
cosign verify \
  --certificate-identity-regexp '^https://github\.com/vtmocanu/uzi/\.github/workflows/release\.yml@refs/tags/.*$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/vtmocanu/uzi/web:<version>
cosign verify \
  --certificate-identity-regexp '^https://github\.com/vtmocanu/uzi/\.github/workflows/release\.yml@refs/tags/.*$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/vtmocanu/uzi/agent-base:<version>
```

Replace each literal `<version>` before running, for example with `0.85.2` for tag `v0.85.2`.

## 3. Prepare the deployment copy and secrets

Create a local deployment copy, keeping the repository template for reference:

```sh
cd /opt/uzi
cp docker-compose.yml compose.vps.yml
```

Edit `/opt/uzi/compose.vps.yml` on the VPS. **Replace**, rather than supplement, the API's `build: ./api` ([line 29](../docker-compose.yml#L29)), web's entire `build:` mapping ([lines 256–258](../docker-compose.yml#L256)), and agent's entire `build:` mapping **including nested `args`** ([lines 306–310](../docker-compose.yml#L306)) with these respective entries, substituting the verified version:

```yaml
# Under services.api:
    image: ghcr.io/vtmocanu/uzi/api:<version>
# Under services.web:
    image: ghcr.io/vtmocanu/uzi/web:<version>
# Under services.agent:
    image: ghcr.io/vtmocanu/uzi/agent-base:<version>
```

Leave no `build` key in those three services. Keep their remaining settings, including healthchecks, profiles, capabilities, volumes and secrets. Preserve web's `127.0.0.1:8080:8080` binding ([docker-compose.yml:260](../docker-compose.yml#L260)); host Caddy will be the public entrypoint. Do not publish API or database ports.

```sh
cd /opt/uzi
umask 077
cp .env.example .env
chmod 600 .env
openssl rand -hex 64
openssl rand -base64 32
openssl rand -hex 24
```

Put the three outputs into `JWT_SECRET`, `UZI_SECRET_KEY` and `POSTGRES_PASSWORD`, respectively ([installation.md:32–34](./installation.md#configure)). Keep the database/user defaults `uzi` ([docker-compose.yml:14–16](../docker-compose.yml#L14)). Set these additional values in `.env`, replacing the domain and admin credentials:

```dotenv
FRONTEND_ORIGIN=https://uzi.example.com
UZI_PUBLIC_BASE_URL=https://uzi.example.com
TRUSTED_PROXIES=
UZI_REGISTRATION_ENABLED=false
UZI_SEED_EMAIL=admin@example.com
UZI_SEED_PASSWORD='<a unique password of at least 12 characters>'
```

Single-quote the seed password in `.env` so Compose treats characters such as `$` literally ([Compose dotenv syntax](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/#env-file-syntax)).

`FRONTEND_ORIGIN` determines the cookie's `Secure` attribute ([config.go:1464](../api/internal/config/config.go#L1464)); `UZI_PUBLIC_BASE_URL` is the public-link base, including Slack deep links ([config.go:1352–1356](../api/internal/config/config.go#L1352), [settings_slack.go:23–25](../api/internal/settings/settings_slack.go#L23)). Both must use your public HTTPS origin here. Compose passes these into the API ([docker-compose.yml:36](../docker-compose.yml#L36), [line 124](../docker-compose.yml#L124)).

Provision the admin before public startup. The seed creates an admin if its email is absent and leaves existing users untouched; the password floor is 12 characters ([.env.example:214–216](../.env.example#L214), [main.go:1982–2014](../api/cmd/server/main.go#L1982)). Disabled registration does not block seeding ([.env.example:66](../.env.example#L66)); Compose forwards both seed fields ([docker-compose.yml:100–102](../docker-compose.yml#L100)).

Store a secure, encrypted off-host copy of `.env` and the edited Compose file, with access restricted to operators. Losing or rotating `UZI_SECRET_KEY` makes previously encrypted tokens unreadable; there is no re-encryption path ([.env.example:15–17](../.env.example#L15)). A database dump without that key cannot recover those credentials. Keep the database password stable too: it initializes the persistent database on first run ([installation.md:37–40](./installation.md#configure)).

### Keep the fail-safe proxy trust default

Leave `TRUSTED_PROXIES` empty ([docker-compose.yml:177](../docker-compose.yml#L177)). nginx **overwrites** X-Forwarded-For with `$remote_addr` in every API location: [`/api/ws` at 36, overwrite at 45](../web/nginx.conf#L36), [`/api/` at 54, overwrite at 61](../web/nginx.conf#L54), [the run-input/decision regex at 68, overwrite at 77](../web/nginx.conf#L68), and [`/api/v1/files` at 95, overwrite at 107](../web/nginx.conf#L95). Thus the host proxy/Docker address replaces the browser IP; Caddy's incoming forwarded header does not survive this hop.

With empty trust, the API ignores XFF and keys the IP limiter by its direct peer, the web/nginx container ([ratelimit.go:220–230](../api/internal/middleware/ratelimit.go#L220)). Browsers therefore share the login bucket. This fails safe by being stricter, but can cause shared login throttling. Setting `172.16.0.0/12` is unsafe: it also trusts the agent on the default Compose network, letting a compromised worker forge XFF to evade brute-force limits ([auth-design.md](./auth-design.md#rate-limiting-and-the-x-forwarded-for-trust-model)). Host Caddy alone does not fix client-IP attribution in this topology.

## 4. Start locally, then enable HTTPS

Validate without printing the resolved secrets, pull the signed application images and database, and start without builds:

```sh
cd /opt/uzi
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml config --quiet
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml pull db api web
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml up -d --no-build --wait db api web
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml ps
```

After confirming the seed account exists (check API logs if startup fails), remove `UZI_SEED_EMAIL` and `UZI_SEED_PASSWORD` from `.env` and recreate the API:

```sh
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml up -d --no-build --force-recreate --wait api
```

Until both removal and recreation, the seed password remains in the host file and/or the API container's configured environment ([docker-compose.yml:100–102](../docker-compose.yml#L100)). Removing it does not change the account's stored password: seeding is create-only ([main.go:1986–1988](../api/cmd/server/main.go#L1986)). Also remove it from any retained configuration copies.

An alternative is to omit the seed, temporarily enable registration, register the first account, then set `UZI_REGISTRATION_ENABLED=false` and run the same API recreation command ([registration behavior](./auth-design.md#first-user-admin), [Compose forwarding:45](../docker-compose.yml#L45)). This avoids retaining a seed password in container configuration, but public first registration opens a first-admin race; restrict access until your registration completes.

Replace `/etc/caddy/Caddyfile` on this dedicated host with your actual domain:

```caddyfile
uzi.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

Caddy obtains and renews HTTPS certificates automatically. Its [`reverse_proxy`](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#defaults) preserves Host for this HTTP upstream and [handles WebSocket upgrades](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#streaming) automatically; no separate `/api/ws` stanza is needed. nginx's [WebSocket location:36](../web/nginx.conf#L36) forwards [Upgrade, Connection and the original Host:41–43](../web/nginx.conf#L41) to the API for its origin check.

```sh
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
systemctl enable --now caddy
systemctl reload caddy
systemctl status caddy --no-pager
```

See [Caddy service administration](https://caddyserver.com/docs/running#using-the-service). If certificate issuance fails, check DNS, inbound 80/443 and `journalctl -u caddy` before proceeding.

In a browser, check that HTTP redirects to HTTPS, the certificate is valid, and the seeded admin can log in. In developer tools, confirm `uzi_auth` has `Secure` and `HttpOnly` ([cookie.go:46–54](../api/internal/auth/cookie.go#L46)), and `/api/ws` connects over `wss://` with status 101. Check that live UI updates arrive. If login is throttled, remember the shared bucket; do not broaden `TRUSTED_PROXIES` as a workaround.

## 5. Register and start your worker

Open **Settings → Workers → Add a worker → Register your own worker** and copy the once-shown join token ([worker-setup.md:15](./worker-setup.md#1-generate-a-join-token)). Add `UZI_WORKER_TOKEN=<join token>` to `/opt/uzi/.env`, keeping mode 0600.

That host variable is the Compose secret **source**, not an agent container environment variable: the [top-level secret:597–598](../docker-compose.yml#L597) is attached by the [service secrets:430–431](../docker-compose.yml#L430), and the agent receives `UZI_WORKER_TOKEN_FILE=/run/secrets/worker_token` ([line 358](../docker-compose.yml#L358)). Startup enforces `0400 worker:worker` ownership ([proc-hardening.md:63–69](./proc-hardening.md#what-each-close-now-rests-on)). Preserve this file-delivery path.

```sh
cd /opt/uzi
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml --profile agent config --quiet
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml --profile agent pull agent
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml --profile agent up -d --no-build agent
```

Confirm the worker is **online** in Settings. The bundled agent uses `http://api:8080` internally ([docker-compose.yml:351](../docker-compose.yml#L351)); a standalone worker on another host must instead dial your public `https://uzi.example.com` through `UZI_API_URL` and use a token file. Follow [worker setup §3](./worker-setup.md#3-run-the-worker), retaining its capabilities and writable token mount, with your verified `agent-base` release image in place of the local build. Save a [model credential](./anthropic-token.md) (or [Codex credential](./codex-credentials.md)) and [connect a forge](./getting-started.md) before starting work.

**If using Coolify or another PaaS:** check whether it injects stack environment variables into every container. Injecting `JWT_SECRET` or `UZI_SECRET_KEY` into the agent breaks the intended service boundary: these belong to the [API environment:32–34](../docker-compose.yml#L32), while the [agent environment:351–358](../docker-compose.yml#L351) needs its API URL and token-file path. If the platform cannot scope secrets per service, deploy the worker as a separate stack with only worker configuration and its token file.

## 6. Back up nightly and copy off-host

PostgreSQL 17 uses database/user `uzi` ([docker-compose.yml:12–16](../docker-compose.yml#L12)) and persistent `pgdata` ([installation.md:81](./installation.md#persistence)). The repository already uses `pg_dump` through the db service ([smoke-prd3.sh:22](../scripts/smoke-prd3.sh#L22)); this recommended operator procedure uses PostgreSQL's [custom archive format (`-Fc`)](https://www.postgresql.org/docs/17/app-pgdump.html), suitable for [`pg_restore`](https://www.postgresql.org/docs/17/app-pgrestore.html). It is not a backup job shipped with the app.

Install the scheduler and SSH client on a minimal host:

```sh
apt-get install -y cron openssh-client
systemctl enable --now cron
```

Create `/opt/uzi/backup.sh` with this content, replacing `backup@example.net:/srv/backups/uzi/` with your own restricted off-host SSH destination:

```sh
#!/bin/sh
set -eu
cd /opt/uzi
umask 077
mkdir -p /opt/uzi/backups
chmod 700 /opt/uzi/backups
stamp=$(date -u +%Y%m%dT%H%M%SZ)
partial=$(mktemp /opt/uzi/backups/.dump.XXXXXX)
trap 'rm -f "$partial"' EXIT HUP INT TERM
docker compose --project-name uzi --file /opt/uzi/compose.vps.yml exec -T db pg_dump -U uzi -d uzi -Fc > "$partial"
test -s "$partial"
dump="/opt/uzi/backups/uzi-$stamp.dump"
mv "$partial" "$dump"
scp -B "$dump" backup@example.net:/srv/backups/uzi/
```

A dump failure removes the partial file and stops; a copy failure exits nonzero while retaining the local dump. Provision a dedicated backup SSH key, verify the remote host key interactively, and test `/opt/uzi/backup.sh` before scheduling. Restrict and encrypt off-host backups: dumps contain account and application data. Keep the encryption key/configuration copy separately protected, and arrange retention for both local and off-host dumps, plus failure alerts.

```sh
chmod 700 /opt/uzi/backup.sh
/opt/uzi/backup.sh
crontab -e
```

Add `0 2 * * * /opt/uzi/backup.sh` to root's crontab for a nightly 02:00 run in the host's timezone; see [Debian crontab syntax](https://manpages.debian.org/bookworm/cron/crontab.5.en.html). Ensure cron failures reach an operator through configured mail or monitoring; an unobserved schedule is not a verified backup.

## 7. Test a restore without touching the live database

Run after the admin exists and after a successful dump. Retrieve an off-host copy into `/opt/uzi/backups/` for a stronger test. Run this in Bash, replacing the filename with an existing archive. The subshell cleans up when the test finishes. It starts an isolated, disposable PostgreSQL 17 container, uses no live volumes, and verifies a restored admin row rather than merely listing the archive. The [seed writes `users.is_admin`](../api/cmd/server/main.go#L2001).

```bash
(
set -euo pipefail
cd /opt/uzi
dump=/opt/uzi/backups/uzi-YYYYMMDDTHHMMSSZ.dump
test -s "$dump"
restore_name="restore-check-$(openssl rand -hex 8)"
restore_id=
cleanup() {
  if [ -n "$restore_id" ]; then docker rm -f "$restore_id" >/dev/null; fi
}
trap cleanup EXIT
restore_id=$(docker run -d --name "$restore_name" --network none \
  --tmpfs /var/lib/postgresql/data \
  -e POSTGRES_HOST_AUTH_METHOD=trust -e POSTGRES_USER=uzi -e POSTGRES_DB=uzi \
  postgres:17)
ready=false
for attempt in {1..60}; do
  if docker exec "$restore_id" pg_isready -h 127.0.0.1 -U uzi -d uzi >/dev/null 2>&1; then
    ready=true
    break
  fi
  sleep 1
done
[ "$ready" = true ]
docker exec -i "$restore_id" pg_restore --exit-on-error -U uzi -d uzi < "$dump"
docker exec "$restore_id" psql -U uzi -d uzi -v ON_ERROR_STOP=1 -c \
  "DO \$\$ BEGIN IF NOT EXISTS (SELECT 1 FROM users WHERE is_admin) THEN RAISE EXCEPTION 'no restored admin'; END IF; END; \$\$;"
docker exec "$restore_id" psql -U uzi -d uzi -c 'SELECT count(*) AS restored_users FROM users;'
)
```

Trust authentication here is confined to the network-disabled throwaway container ([PostgreSQL image initialization options](https://hub.docker.com/_/postgres)). Its data is tmpfs: allow enough RAM for the restored database, or use a dedicated test host. The exit trap removes exactly the container created by this shell; tmpfs state disappears with it. No live Compose teardown is needed. A successful database restore does not test decryption: also verify that you can securely recover the matching `UZI_SECRET_KEY` and deployment configuration. Plan a separate isolated application recovery exercise before relying on the backups for disaster recovery.
