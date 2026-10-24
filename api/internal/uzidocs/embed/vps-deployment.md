---
title: Run uzi on a VPS with Docker Compose
order: 15
audience: operator
---

# Run uzi on a VPS with Docker Compose

You can run uzi alongside the services already on your VPS, using your existing reverse proxy and backup routine. In this tutorial, we will publish uzi at `https://their-domain` and bring a worker online without exposing the API or database directly.

The stack has four roles: `db` stores application data in PostgreSQL; `api` handles authentication, configuration, and work coordination; `web` serves the interface and proxies API requests; and `agent` runs work against your repositories. Docker Compose keeps their network, persistent volumes, and startup dependencies together while letting you start the worker separately. We will use the source configuration and signed images from release **v0.85.2**, rather than build application images on the VPS.

## Prerequisites

Before you begin, check that you have:

- A Debian or Ubuntu VPS with a sudo-capable account and an existing firewall policy allowing your reverse proxy's HTTP/HTTPS traffic. See [Initial Server Setup with Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu) if you need to prepare these first.
- Docker Engine and **Docker Compose 2.24.4 or newer**. See Docker's [Ubuntu installation guide](https://docs.docker.com/engine/install/ubuntu/) and [Compose plugin installation guide](https://docs.docker.com/compose/install/linux/). This tutorial uses the [Compose reset and override tags](https://docs.docker.com/reference/compose-file/merge/).
- A domain whose A record points to your VPS. Follow the [DigitalOcean DNS record guide](https://docs.digitalocean.com/products/networking/dns/how-to/manage-records/) with your DNS provider, and replace `their-domain` throughout this tutorial.
- An existing TLS reverse proxy serving other applications. We include Caddy and nginx routing examples below; if you need to set one up first, see [How To Host a Website with Caddy on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-with-caddy-on-ubuntu-22-04).
- Spare capacity for uzi in addition to your existing services. The worker alone defaults to **2 CPUs and 4 GiB RAM**; allow additional memory, CPU, and persistent disk for PostgreSQL, the API, web, repositories, and tool caches. See [worker sizing](./worker-setup.md#resource-stats-and-sizing).
- `curl`, `tar`, and `openssl`, plus a current `cosign` installed using the [cosign installation guide](https://docs.sigstore.dev/cosign/system_config/installation/). See [Container signing](./container-signing.md) for the supported verifier guidance.
- An existing way to protect credentials and copy backups off the VPS.

## Step 1 - Download the release and verify its images

We will keep the release configuration in a stable directory so Compose continues to find the same project and volumes on later visits.

Create `/opt/uzi` for this deployment; choose another stable path if your server uses a different service layout:

```sh
sudo mkdir -p /opt/uzi
```

Give your current account ownership of this new directory so you can maintain its configuration:

```sh
sudo chown "$USER":"$(id -gn)" /opt/uzi
```

Download the upstream source archive for the release we will deploy:

```sh
curl -fL https://github.com/vtmocanu/uzi/archive/refs/tags/v0.85.2.tar.gz -o /opt/uzi/v0.85.2.tar.gz
```

Extract the archive into that directory, removing the archive's enclosing directory component:

```sh
tar -xzf /opt/uzi/v0.85.2.tar.gz -C /opt/uzi --strip-components=1
```

Enter the deployment directory and run the remaining Compose commands from here:

```sh
cd /opt/uzi
```

Verify the API image against the upstream release workflow identity and GitHub Actions issuer. Stop if verification fails:

```sh
cosign verify --certificate-identity 'https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v0.85.2' --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' ghcr.io/vtmocanu/uzi/api:0.85.2
```

Verify the matching web image with the same identity:

```sh
cosign verify --certificate-identity 'https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v0.85.2' --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' ghcr.io/vtmocanu/uzi/web:0.85.2
```

Verify the base-template worker image as well:

```sh
cosign verify --certificate-identity 'https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v0.85.2' --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' ghcr.io/vtmocanu/uzi/agent-base:0.85.2
```

These commands verify registry images, not the source archive; see [Container signing](./container-signing.md) for the signing boundary and digest-based verification. Keep this directory name stable: changing it changes Compose's default project name and can make existing named volumes appear to be missing. Do not extract another release over this deployment's configuration during an upgrade.

You now have the v0.85.2 configuration in a stable directory and have verified its three application images.

## Step 2 - Configure secrets and choose an admin bootstrap

We will configure the public origin and choose how to create the first administrator before the API starts.

Copy the release's `.env.example` to a new `.env` file; do this for a fresh deployment, since it replaces an existing destination:

```sh
cp .env.example .env
```

Restrict access to the file before filling in credentials:

```sh
chmod 600 .env
```

Generate a session signing key and paste its output into `JWT_SECRET` in `.env`:

```sh
openssl rand -hex 64
```

Generate the encryption master key and paste its output into `UZI_SECRET_KEY`:

```sh
openssl rand -base64 32
```

Generate the database password and paste its output into `POSTGRES_PASSWORD`:

```sh
openssl rand -hex 24
```

Edit `.env` using your usual editor. Keep the generated values and the default `POSTGRES_USER=uzi` and `POSTGRES_DB=uzi`. Set the following entries, replacing the domain, seed email, and seed password. Choose a unique seed password of at least 12 characters; single-quote it in `.env` if it contains characters Compose might interpolate:

```dotenv
FRONTEND_ORIGIN=https://their-domain
UZI_PUBLIC_BASE_URL=https://their-domain
TRUSTED_PROXIES=
UZI_REGISTRATION_ENABLED=false
UZI_SEED_EMAIL=you@example.com
UZI_SEED_PASSWORD='replace-with-a-unique-admin-password'
UZI_SEED_NAME=Administrator
UZI_WORKER_TOKEN=
```

The recommended bootstrap creates a seeded admin with public registration disabled. Its trade-off is that the seed password remains in `.env` and the API container environment. The startup seed is **create-only**: it creates the configured email if absent and does not reset an existing account's password or permissions. This is a fresh-install procedure, not an admin recovery method.

Alternatively, leave `UZI_SEED_EMAIL`, `UZI_SEED_PASSWORD`, and `UZI_SEED_NAME` empty and set `UZI_REGISTRATION_ENABLED=true`. The first registered account becomes admin. Restrict access to the site until you register: exposing an empty installation with registration enabled creates a first-account race. Step 5 closes registration immediately after your account is created.

Keep `UZI_WORKER_TOKEN=` explicitly defined but empty for now. Compose's environment-backed secret source distinguishes a missing variable from an empty value; an empty source lets you resolve the configuration before a join token exists. Do not start `agent` with this empty token. In Step 6, you will replace it with an issued token. Clear any conflicting exported shell variables, since they take precedence over `.env` during Compose configuration.

Back up `.env` securely, especially `UZI_SECRET_KEY`: losing or replacing that key makes existing encrypted forge and model tokens unreadable. See [Configuration](./configuration.md) for the remaining options.

Your installation now has persistent secrets, a matching HTTPS origin and public URL, and a bootstrap policy chosen before startup.

## Step 3 - Select release images and start the application

We will override the archive's local build definitions so the VPS runs the verified release images.

Create `compose.override.yml` in `/opt/uzi` with the following content. Keep the image versions together when choosing a different release. `!reset null` removes each inherited build definition; Compose 2.24.4 or newer also supports the optional `!override` port replacement below:

```yaml
services:
  api:
    build: !reset null
    image: ghcr.io/vtmocanu/uzi/api:0.85.2
  web:
    build: !reset null
    image: ghcr.io/vtmocanu/uzi/web:0.85.2
  agent:
    build: !reset null
    image: ghcr.io/vtmocanu/uzi/agent-base:0.85.2
```

The base file publishes web at `127.0.0.1:8080`. If that port is occupied, incorporate this replacement into the **existing `web` stanza** in your override, retaining its build reset and image. Do not add a second `web` key. Then use port `18080` in both proxy examples in Step 4:

```yaml
  web:
    build: !reset null
    image: ghcr.io/vtmocanu/uzi/web:0.85.2
    ports: !override ["127.0.0.1:18080:8080"]
```

Render the merged configuration, including the worker profile, **before pulling or starting services**. The output contains resolved secrets, including database and seed credentials: inspect it privately and do not paste it into logs or support messages:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent config
```

Check that `api`, `web`, and `agent` have no `build` entries and use the three `0.85.2` images above; `web` publishes only your selected loopback port; `api` and `db` have no published ports; and `agent` retains its `agent` profile. Confirm `pgdata` mounts at `/var/lib/postgresql/data`, `agentdata` at `/data`, `agentnix` at `/nix`, and `worker_token` feeds `/run/secrets/worker_token` with `UZI_WORKER_TOKEN_FILE` pointing there. Stop and correct the files if these checks fail. The token source is intentionally empty at this stage.

Pull the database and three application images without activating the worker:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent pull db api web agent
```

Start the database, API, and web service, forbidding local image builds:

```sh
docker compose -f docker-compose.yml -f compose.override.yml up -d --no-build db api web
```

Inspect their status and wait for the database and API health checks to pass:

```sh
docker compose -f docker-compose.yml -f compose.override.yml ps
```

The API runs database migrations during startup. If a service fails, inspect its logs privately with your usual operational tools before continuing; do not delete volumes to retry. Keep `pgdata`, `agentdata`, and `agentnix` attached to this project, and avoid `down -v`, which deletes persistent data.

Your database, API, and web service are now running from the selected release, with web reachable through the chosen loopback port.

## Step 4 - Route HTTPS through your existing reverse proxy

We will route the public domain to web so the browser uses one HTTPS origin for the interface, API, and live WebSocket.

Choose the example for your existing proxy. These examples assume the proxy runs on the VPS host. For a containerized proxy, `127.0.0.1` refers to that proxy container, not the host: adapt its network and web upstream for your topology while keeping the API and database unpublished.

For **Caddy**, add this site block to your existing `/etc/caddy/Caddyfile`. Replace `their-domain`, and change `8080` to `18080` if you selected the alternate port. Caddy preserves Host for this HTTP upstream and handles WebSocket upgrades; the explicit `/api/ws` handler documents that route:

```caddyfile
their-domain {
    handle /api/ws {
        reverse_proxy 127.0.0.1:8080
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}
```

Validate your combined Caddy configuration before reloading it:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

If validation succeeds, reload your existing Caddy service:

```sh
sudo systemctl reload caddy
```

For **nginx**, adapt the following locations inside your existing TLS `server` block for `their-domain`. Keep your working certificate settings, and replace the upstream port if needed. Host and the WebSocket upgrade headers must reach web; the exact `/api/ws` location keeps its connection open for live updates. Set upload limits and timeouts for your workload using the existing packaged web configuration as a reference:

```nginx
location = /api/ws {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Host $http_host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 3600s;
}
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Validate the complete nginx configuration before reloading it:

```sh
sudo nginx -t
```

If validation succeeds, reload your existing nginx service:

```sh
sudo systemctl reload nginx
```

Leave `TRUSTED_PROXIES` empty for this tutorial. The API ignores forwarding headers with this setting and keys authentication rate limits on its direct peer, the web container. This safe default prevents the agent from forging client IPs, but shares the auth rate-limit bucket among visitors; one noisy client can exhaust the budget for others.

Distinct client IPs need two changes: `web/nginx.conf` must append to `X-Forwarded-For` instead of overwriting it, and `TRUSTED_PROXIES` must name only the web container.

Those changes are necessary, not sufficient. The API selects the rightmost non-trusted entry in `X-Forwarded-For`; after an append, that entry can still be the outer proxy or Docker bridge, rather than the browser. The outer proxy must sanitize client-supplied forwarding headers, and topology-specific hop handling must be verified. We do not change the packaged nginx configuration or achieve distinct client IPs here. Do not trust broad private-network CIDRs that include the agent. See [Auth design](./auth-design.md#rate-limiting-and-the-x-forwarded-for-trust-model) for the boundary.

Your existing proxy now serves the uzi interface and `/api/ws` through HTTPS, with the shared auth rate-limit bucket retained.

## Step 5 - Complete the administrator bootstrap

We will verify administrator access over the public HTTPS origin before issuing a worker token.

Open `https://their-domain`. For the recommended seeded bootstrap, sign in using the seed email and password and confirm that you can open the admin settings. Registration remains disabled. Changing the seed password in `.env` does not overwrite a password already stored in the database.

For the alternative bootstrap, register your account while access remains restricted, confirm that it is admin, and immediately edit `.env` to set `UZI_REGISTRATION_ENABLED=false`. Recreate the API so it reads that change:

```sh
docker compose -f docker-compose.yml -f compose.override.yml up -d --no-build --force-recreate api
```

Confirm that registration is disabled and your account can still sign in before opening access more widely. In your browser's developer tools, inspect the `uzi_auth` cookie and confirm that it has `Secure` and `HttpOnly` set. **`FRONTEND_ORIGIN` controls the Secure flag** through its HTTPS scheme. `UZI_PUBLIC_BASE_URL` should match your public URL, but does not control that flag. If the cookie is not Secure, correct `FRONTEND_ORIGIN`, recreate the API with the command above, and sign in again.

You now have a working administrator session over HTTPS with registration closed and the session cookie marked Secure.

## Step 6 - Register and start your worker

We will issue a join token for your account and deliver it through the release's existing Compose secret wiring.

Open **Settings → Workers → Add a worker → Register your own worker**. Select the base template and copy the token, which is shown once. Edit the existing `UZI_WORKER_TOKEN=` entry in `.env` to contain that value, keeping the file private. This entry is the **source for the Compose secret**, not an environment variable to add to the agent container.

The following excerpt is **already present in `docker-compose.yml`**; inspect it rather than adding duplicate definitions. It connects the `.env` source to the file the worker reads:

```yaml
services:
  agent:
    environment:
      UZI_WORKER_TOKEN_FILE: /run/secrets/worker_token
    secrets:
      - worker_token
secrets:
  worker_token:
    environment: UZI_WORKER_TOKEN
```

Render the merged configuration again in a private terminal to check that the source is populated and the secret/file wiring remains intact; the output may reveal the token as well as other credentials:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent config
```

Start the image-based agent with its profile enabled and builds forbidden:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent up -d --no-build agent
```

Inspect the worker container's status:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent ps
```

Return to **Settings → Workers** and wait for the worker to appear online. If it remains offline, check the issued token, secret source, and API connectivity before assigning work. See [Worker setup](./worker-setup.md) for troubleshooting and further worker options.

Your base-template worker is now online and registered to your account.

## Step 7 - Back up the database and rehearse a restore

We will preserve application data together with its encryption key and prove that a backup can be restored away from the live database.

You can use your existing PostgreSQL-aware backup tool. For a manual custom-format dump with the default `uzi` role and database, run the following command from `/opt/uzi`; select a fresh output filename each time so you do not overwrite an earlier backup:

```sh
docker compose -f docker-compose.yml -f compose.override.yml exec -T db pg_dump -U uzi -d uzi -Fc > uzi-backup.dump
```

Protect this dump with your existing backup controls. Back up `.env` above all, including **the original `UZI_SECRET_KEY`**, along with `compose.override.yml` and your release version record. A database dump without its encryption key cannot recover the encrypted tokens. Preserve the worker volumes if you need its local repositories and caches; a database dump does not include them. Copy backups and protected configuration off-host using your existing routine.

If you want a dedicated backup container, add the following `db-backup` service **under the same `services` mapping in `compose.override.yml`**; retain the three application overrides. This optional example uses [nfrastack/db-backup 5.0.5](https://nfrastack.com/db-backup/container), PostgreSQL settings, a first backup five minutes after startup, and a daily interval. Change the schedule and host directory to suit your routine. `/opt/uzi/backups` must be writable by its documented backup uid, `10000`; protect the directory and its archives. The service uses the default Compose network to reach `db`:

```yaml
  db-backup:
    image: docker.io/nfrastack/db-backup:5.0.5
    restart: unless-stopped
    environment:
      TIMEZONE: Etc/UTC
      DB01_TYPE: postgres
      DB01_HOST: db
      DB01_USER: ${POSTGRES_USER:-uzi}
      DB01_PASS: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      DB01_NAME: ${POSTGRES_DB:-uzi}
      DB01_BACKUP_BEGIN: "+5"
      DB01_BACKUP_INTERVAL: "1440"
    volumes:
      - /opt/uzi/backups:/backup
    depends_on:
      db:
        condition: service_healthy
```

Create its persistent backup directory before starting the service:

```sh
sudo install -d -m 0700 -o 10000 -g 10000 /opt/uzi/backups
```

Render the configuration privately and inspect the added backup service's credentials, schedule, volume, and database target:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent config
```

Pull the optional backup image:

```sh
docker compose -f docker-compose.yml -f compose.override.yml pull db-backup
```

Start the backup service without building images:

```sh
docker compose -f docker-compose.yml -f compose.override.yml up -d --no-build db-backup
```

Run its built-in `backup-now` helper to test a backup immediately rather than wait for the schedule:

```sh
docker compose -f docker-compose.yml -f compose.override.yml exec db-backup backup-now
```

Inspect the backup logs and select a completed archive under `/opt/uzi/backups`. This tool uses its own PostgreSQL SQL/COPY export, with **Zstandard compression by default**, rather than the `pg_dump -Fc` archive shown earlier. Restore its archives with its own restore command; use PostgreSQL's `pg_restore` for the manual custom-format dump. See the upstream [backup recipes](https://nfrastack.com/db-backup/recipes/backup) and [restore recipes](https://nfrastack.com/db-backup/recipes/restore) for supported options and archive selection. The auto-generated configuration lives at `/config/db-backup.yml`; mounting an existing config there changes how environment-based configuration is applied, so this example leaves that path unmounted.

For a disposable restore target, add this `restore-db` service under the **same override's `services` mapping**. It has no published port, uses a separate database and anonymous data volume, and starts only when requested. It reuses the database password from `.env`; no application service points to it:

```yaml
  restore-db:
    image: postgres:17
    profiles: ["restore"]
    environment:
      POSTGRES_USER: uzi
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: uzi_restore
    volumes:
      - /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U uzi -d uzi_restore"]
      interval: 5s
      timeout: 5s
      retries: 10
```

Render the expanded configuration privately, confirming that `restore-db` is separate from `db` and mounts no live database volume:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile agent --profile restore config
```

Pull the disposable PostgreSQL image:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile restore pull restore-db
```

Start the target and wait for its health check to pass:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile restore up -d --no-build --wait restore-db
```

For a backup-container archive, replace `/backup/selected-archive.sql.zst` with the actual selected archive's path inside the backup container. `env://DB01_PASS` reads the password already configured in that container; the disposable target uses the same password. Explicit target flags override filename inference and direct the restore to **`restore-db:5432/uzi_restore`**, not live `db`:

```sh
docker compose -f docker-compose.yml -f compose.override.yml exec db-backup /app/dbb --config /config/db-backup.yml restore --file /backup/selected-archive.sql.zst --type postgres --host restore-db --port 5432 --user uzi --pass env://DB01_PASS --name uzi_restore
```

Alternatively, restore the manual `pg_dump -Fc` archive into that same disposable target using `pg_restore`, with errors stopping the restore:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile restore exec -T restore-db pg_restore -U uzi -d uzi_restore --exit-on-error < uzi-backup.dump
```

Use a fresh, empty target for each rehearsal; do not run both restore methods into the same populated database. Inspect restored data and verify application recovery with the preserved encryption key in an isolated environment before relying on your recovery procedure. Do not point the live API at this rehearsal database. When finished, remove just the disposable service container and its anonymous volume:

```sh
docker compose -f docker-compose.yml -f compose.override.yml --profile restore rm -s -f -v restore-db
```

You now have a database backup, protected configuration copied off-host, and a restore rehearsal that leaves the live database untouched.

## Conclusion

You have uzi behind your existing HTTPS proxy and a worker ready to accept work. Next, connect your forge using the [GitLab](./gitlab-bot-setup.md), [GitHub](./github-bot-setup.md), or [Forgejo](./forgejo-bot-setup.md) guide. Add [Claude credentials](./anthropic-token.md) or [Codex credentials](./codex-credentials.md), then review [Worker setup](./worker-setup.md) to run a worker on another host or explore further worker options before running your first issue.

## Reference

The release configuration above is pinned to v0.85.2. Current-branch evidence below explains the implementation boundaries; it does not imply that later branch changes are present in those release images.

| Scope | File and line evidence | What to check |
|---|---|---|
| Release v0.85.2 | [docker-compose.yml:29](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L29), [253](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L253), [303](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L303) | Inherited API, web, and agent build definitions removed by the override. |
| Release v0.85.2 | [docker-compose.yml:257](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L257), [308](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L308), [345](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L345), [355](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L355), [584](https://github.com/vtmocanu/uzi/blob/v0.85.2/docker-compose.yml#L584) | Loopback publication, worker profile and limits, token file, and environment-backed secret. |
| Current branch | [docker-compose.yml:29](../docker-compose.yml#L29), [256](../docker-compose.yml#L256), [306](../docker-compose.yml#L306), [260](../docker-compose.yml#L260), [311](../docker-compose.yml#L311), [348](../docker-compose.yml#L348), [358](../docker-compose.yml#L358), [597](../docker-compose.yml#L597) | Corresponding build, port, profile, sizing, and secret wiring. |
| Current branch | [docker-compose.yml:36](../docker-compose.yml#L36), [124](../docker-compose.yml#L124), [177](../docker-compose.yml#L177), [.env.example:11](../.env.example#L11) | Origin, public URL, empty trusted proxies, and encryption-key loss warning. |
| Current branch | [api/cmd/server/main.go:1982](../api/cmd/server/main.go#L1982), [api/internal/handler/auth.go:96](../api/internal/handler/auth.go#L96), [188](../api/internal/handler/auth.go#L188), [api/internal/config/config.go:1464](../api/internal/config/config.go#L1464) | Create-only seed, registration closure, first-user admin assignment, and Secure-cookie derivation. |
| Current branch | [api/internal/middleware/ratelimit.go:229](../api/internal/middleware/ratelimit.go#L229), [239](../api/internal/middleware/ratelimit.go#L239), [web/nginx.conf:36](../web/nginx.conf#L36), [45](../web/nginx.conf#L45), [61](../web/nginx.conf#L61), [77](../web/nginx.conf#L77), [107](../web/nginx.conf#L107) | Direct-peer trust boundary, rightmost non-trusted hop, WebSocket route, and packaged forwarding-header overwrites. |
| Compose implementation | [compose v2.24.4 pkg/compose/secrets.go:52](https://github.com/docker/compose/blob/v2.24.4/pkg/compose/secrets.go#L52), [compose-go v2.4.4 loader/environment.go:81](https://github.com/compose-spec/compose-go/blob/v2.4.4/loader/environment.go#L81) | A defined empty environment secret differs from an absent variable; inspect your merged configuration before startup. |
| Backup container 5.0.5 | [container/rootfs/usr/local/bin/backup-now:43](https://github.com/nfrastack/db-backup/blob/5.0.5/container/rootfs/usr/local/bin/backup-now#L43), [container/rootfs/container/defaults/10-dbbackup:7](https://github.com/nfrastack/db-backup/blob/5.0.5/container/rootfs/container/defaults/10-dbbackup#L7), [container/rootfs/container/functions/10-dbbackup:36](https://github.com/nfrastack/db-backup/blob/5.0.5/container/rootfs/container/functions/10-dbbackup#L36), [94](https://github.com/nfrastack/db-backup/blob/5.0.5/container/rootfs/container/functions/10-dbbackup#L94) | Built-in immediate backup, generated config path, existing-config handling, and compression fallback. |
| Backup implementation | [5.0.5 src/internal/database/engine/postgres/postgres.go:147](https://github.com/nfrastack/db-backup/blob/5.0.5/src/internal/database/engine/postgres/postgres.go#L147), [src/cmd/dbb/restore.go:65](https://github.com/nfrastack/db-backup/blob/5.0.5/src/cmd/dbb/restore.go#L65) | Pinned native PostgreSQL exporter, explicit restore target flags, and environment-sourced password support. |
