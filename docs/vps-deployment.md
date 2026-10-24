---
title: Run uzi on a VPS with Docker Compose
order: 15
audience: operator
---

# Run uzi on a VPS with Docker Compose

You can run uzi alongside the services already on your VPS, using your existing reverse proxy and backup routine. In this tutorial, we will publish uzi at `https://your_domain` and bring a worker online without exposing the API or database directly.

The stack has four roles: `db` stores application data in PostgreSQL; `api` handles authentication, configuration, and work coordination; `web` serves the interface and proxies API requests; and `agent` runs work against your repositories. Docker Compose keeps their network, persistent volumes, and startup dependencies together while letting you start the worker separately. We will use the dedicated [server Compose file](../deploy/compose/compose.yml) and signed images from your selected release. This file uses the base-agent template and excludes the optional Docker-in-Docker (DinD) services; see [Worker setup](./worker-setup.md) for other worker options.

## Prerequisites

Before you begin, check that you have:

- A Debian or Ubuntu VPS with a sudo-capable account and an existing firewall policy allowing your reverse proxy's HTTP/HTTPS traffic. See [Initial Server Setup with Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu) if you need to prepare these first.
- Docker Engine and the **Docker Compose v2 plugin**. See Docker's [Ubuntu installation guide](https://docs.docker.com/engine/install/ubuntu/) and [Compose plugin installation guide](https://docs.docker.com/compose/install/linux/).
- A domain whose A record points to your VPS. Follow the [DigitalOcean DNS record guide](https://docs.digitalocean.com/products/networking/dns/how-to/manage-records/) with your DNS provider, and replace `your_domain` throughout this tutorial.
- An existing TLS reverse proxy serving other applications. We include Caddy and nginx routing examples below; if you need to set one up first, see [How To Host a Website with Caddy on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-with-caddy-on-ubuntu-22-04).
- Spare capacity for uzi in addition to your existing services. The worker alone defaults to **2 CPUs and 4 GiB RAM**; allow additional memory, CPU, and persistent disk for PostgreSQL, the API, web, repositories, and tool caches. See [worker sizing](./worker-setup.md#resource-stats-and-sizing).
- `curl`, Bash, and `openssl`, plus a current `cosign` installed using the [cosign installation guide](https://docs.sigstore.dev/cosign/system_config/installation/). See [Container signing](./container-signing.md) for the supported verifier guidance.
- An existing way to protect credentials and copy backups off the VPS.

## Step 1 - Download the release configuration and verify images

We will keep the release configuration in a stable directory so Compose continues to find the same project and volumes on later visits.

Create `/opt/uzi` for this deployment; choose another stable path if your server uses a different service layout:

```sh
sudo mkdir -p /opt/uzi
```

Give your current account ownership of this new directory so you can maintain its configuration:

```sh
sudo chown "$USER":"$(id -gn)" /opt/uzi
```

Enter the deployment directory and run the remaining Compose commands from here:

```sh
cd /opt/uzi
```

Choose a release at [GitHub Releases](https://github.com/vtmocanu/uzi/releases) **that contains `deploy/compose/compose.yml`**. Older releases, including `0.85.2`, do not contain this file. Set the shell variable below to your chosen release number **without the leading `v`**, replacing the placeholder before running it:

```sh
UZI_VERSION='<release-number-without-v>'
```

Download the three files from that same release. Run this block in Bash; `set -e` stops it on a failed download. If a download fails (including a missing manifest), stop and check your release selection before continuing:

```bash
set -e
curl -fsSLo compose.yml "https://raw.githubusercontent.com/vtmocanu/uzi/v${UZI_VERSION}/deploy/compose/compose.yml"
curl -fsSLo .env.example "https://raw.githubusercontent.com/vtmocanu/uzi/v${UZI_VERSION}/.env.example"
mkdir -p scripts
curl -fsSLo scripts/init-env.sh "https://raw.githubusercontent.com/vtmocanu/uzi/v${UZI_VERSION}/scripts/init-env.sh"
```

Initialize the deployment environment:

```sh
bash scripts/init-env.sh
```

The initializer finds `.env.example` in the parent of its `scripts/` directory, so this downloaded layout works unchanged. It creates a new `.env` with mode `0600` and generates `JWT_SECRET`, `UZI_SECRET_KEY`, and `POSTGRES_PASSWORD` without displaying their values. It leaves an existing `.env` untouched and fails if that file lacks a required secret. Stop if initialization fails; do not continue to edit the file or start services.

Add your chosen version to the newly generated `.env`:

```sh
printf '\nUZI_VERSION=%s\n' "$UZI_VERSION" >> .env
```

Do not source or print `.env`, which contains secrets. Step 2 covers the other values to edit.

Verify the API image against the upstream release workflow identity and GitHub Actions issuer. Stop if verification fails:

```sh
cosign verify --certificate-identity "https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v${UZI_VERSION}" --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' "ghcr.io/vtmocanu/uzi/api:${UZI_VERSION}"
```

Verify the matching web image with the same identity:

```sh
cosign verify --certificate-identity "https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v${UZI_VERSION}" --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' "ghcr.io/vtmocanu/uzi/web:${UZI_VERSION}"
```

Verify the base-template worker image as well:

```sh
cosign verify --certificate-identity "https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v${UZI_VERSION}" --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' "ghcr.io/vtmocanu/uzi/agent-base:${UZI_VERSION}"
```

These commands verify registry images; see [Container signing](./container-signing.md) for the signing boundary and digest-based verification. Keep this directory name stable: changing it changes Compose's default project name and can make existing named volumes appear to be missing. Preserve this deployment's `.env` and local `compose.yml` edits during an upgrade; see [Upgrades](#upgrades).

You now have your selected release's configuration in a stable directory and have verified its three application images.

## Step 2 - Configure secrets and choose an admin bootstrap

We will configure the public origin and choose how to create the first administrator before the API starts.

Edit the `.env` initialized in Step 1 using your usual editor. Retain the generated secrets, your selected `UZI_VERSION`, and the default `POSTGRES_USER=uzi` and `POSTGRES_DB=uzi`. Set the following entries, replacing the domain, seed email, and seed password. Choose a unique seed password of at least 12 characters; single-quote it in `.env` if it contains characters Compose might interpolate:

```dotenv
FRONTEND_ORIGIN=https://your_domain
UZI_PUBLIC_BASE_URL=https://your_domain
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

The downloaded `compose.yml` already selects the release images using `UZI_VERSION` from `.env`.

It publishes web at `127.0.0.1:8080`. If that port is occupied, edit the **single existing entry** under `services.web.ports` in `compose.yml` to:

```yaml
      - "127.0.0.1:18080:8080"
```

Then use port `18080` in both proxy examples in Step 4.

Render the configuration, including the worker profile, **before pulling or starting services**. The output contains resolved secrets, including database and seed credentials: inspect it privately and do not paste it into logs or support messages:

```sh
docker compose -f compose.yml --profile agent config
```

Check that `api`, `web`, and `agent` use the three verified images with your selected version; `web` publishes only your selected loopback port; `api` and `db` have no published ports; and `agent` retains its `agent` profile. Confirm `pgdata` mounts at `/var/lib/postgresql/data`, `agentdata` at `/data`, `agentnix` at `/nix`, and `worker_token` feeds `/run/secrets/worker_token` with `UZI_WORKER_TOKEN_FILE` pointing there. Stop and correct the configuration if these checks fail. The token source is intentionally empty at this stage.

Pull the database and three application images without activating the worker:

```sh
docker compose -f compose.yml --profile agent pull db api web agent
```

Start the database, API, and web service, forbidding local image builds:

```sh
docker compose -f compose.yml up -d --no-build db api web
```

Inspect their status and wait for the database and API health checks to pass:

```sh
docker compose -f compose.yml ps
```

The API runs database migrations during startup. If a service fails, inspect its logs privately with your usual operational tools before continuing; do not delete volumes to retry. Keep `pgdata`, `agentdata`, and `agentnix` attached to this project, and avoid `down -v`, which deletes persistent data.

Your database, API, and web service are now running from the selected release, with web reachable through the chosen loopback port.

## Step 4 - Route HTTPS through your existing reverse proxy

We will route the public domain to web so the browser uses one HTTPS origin for the interface, API, and live WebSocket.

Choose the example for your existing proxy. These examples assume the proxy runs on the VPS host. For a containerized proxy, `127.0.0.1` refers to that proxy container, not the host: adapt its network and web upstream for your topology while keeping the API and database unpublished.

For **Caddy**, add this site block to your existing `/etc/caddy/Caddyfile`. Replace `your_domain`, and change `8080` to `18080` if you selected the alternate port. Caddy preserves Host for this HTTP upstream and automatically handles WebSocket upgrades:

```caddyfile
your_domain {
    reverse_proxy 127.0.0.1:8080
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

For **nginx**, adapt the following locations inside your existing TLS `server` block for `your_domain`. Keep your working certificate settings, and replace the upstream port if needed. Host and the WebSocket upgrade headers must reach web; the exact `/api/ws` location keeps its connection open for live updates. Set upload limits and timeouts for your workload using the existing packaged web configuration as a reference:

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

Open `https://your_domain`. For the recommended seeded bootstrap, sign in using the seed email and password and confirm that you can open the admin settings. Registration remains disabled. Changing the seed password in `.env` does not overwrite a password already stored in the database.

For the alternative bootstrap, register your account while access remains restricted, confirm that it is admin, and immediately edit `.env` to set `UZI_REGISTRATION_ENABLED=false`. Recreate the API so it reads that change:

```sh
docker compose -f compose.yml up -d --no-build --force-recreate api
```

Confirm that registration is disabled and your account can still sign in before opening access more widely. In your browser's developer tools, inspect the `uzi_auth` cookie and confirm that it has `Secure` and `HttpOnly` set. **`FRONTEND_ORIGIN` controls the Secure flag** through its HTTPS scheme. `UZI_PUBLIC_BASE_URL` should match your public URL, but does not control that flag. If the cookie is not Secure, correct `FRONTEND_ORIGIN`, recreate the API with the command above, and sign in again.

You now have a working administrator session over HTTPS with registration closed and the session cookie marked Secure.

## Step 6 - Register and start your worker

We will issue a join token for your account and deliver it through the release's existing Compose secret wiring.

Open **Settings → Workers → Add a worker → Register your own worker**. Select the base template and copy the token, which is shown once. Edit the existing `UZI_WORKER_TOKEN=` entry in `.env` to contain that value, keeping the file private. This entry is the **source for the Compose secret**, not an environment variable to add to the agent container.

The following excerpt is **already present in `compose.yml`**; inspect it rather than adding duplicate definitions. It connects the `.env` source to the file the worker reads:

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

Render the configuration again in a private terminal to check that the source is populated and the secret/file wiring remains intact; the output may reveal the token as well as other credentials:

```sh
docker compose -f compose.yml --profile agent config
```

Start the image-based agent with its profile enabled and builds forbidden:

```sh
docker compose -f compose.yml --profile agent up -d --no-build agent
```

Inspect the worker container's status:

```sh
docker compose -f compose.yml --profile agent ps
```

Return to **Settings → Workers** and wait for the worker to appear online. If it remains offline, check the issued token, secret source, and API connectivity before assigning work. See [Worker setup](./worker-setup.md) for troubleshooting and further worker options.

Your base-template worker is now online and registered to your account.

## Step 7 - Back up the database and configuration

We will preserve application data together with its encryption key. You can use your existing PostgreSQL-aware backup tool or the optional backup container below.

Back up `.env` above all, including **the original `UZI_SECRET_KEY`**, along with your locally edited `compose.yml`, `.env.example`, `scripts/init-env.sh`, and your release version record. A database dump without its encryption key cannot recover the encrypted tokens. Preserve the worker volumes if you need its local repositories and caches; a database dump does not include them. Copy backups and protected configuration off-host using your existing routine.

If you want a dedicated backup container, add the following `db-backup` service **under the existing `services` mapping in `compose.yml`**, alongside `db`, `api`, `web`, and `agent`. Do not add a second `services` key. This optional example uses [nfrastack/db-backup 5.0.5](https://nfrastack.com/db-backup/container), PostgreSQL settings, a first backup five minutes after startup, and a daily interval. Change the schedule and host directory to suit your routine. `/opt/uzi/backups` must be writable by its documented backup uid, `10000`; protect the directory and its archives. The service uses the default Compose network to reach `db`:

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
docker compose -f compose.yml --profile agent config
```

Pull the optional backup image:

```sh
docker compose -f compose.yml pull db-backup
```

Start the backup service without building images:

```sh
docker compose -f compose.yml up -d --no-build db-backup
```

Run its built-in `backup-now` helper to test a backup immediately rather than wait for the schedule:

```sh
docker compose -f compose.yml exec db-backup backup-now
```

Inspect the backup logs and confirm that a completed archive exists under `/opt/uzi/backups`. See the upstream [backup recipes](https://nfrastack.com/db-backup/recipes/backup) for supported options and archive selection. The auto-generated configuration lives at `/config/db-backup.yml`; mounting an existing config there changes how environment-based configuration is applied, so this example leaves that path unmounted.

Follow the upstream [restore recipe](https://nfrastack.com/db-backup/recipes/restore) to rehearse recovery in an isolated environment away from the live database. Verify application recovery with the preserved encryption key before relying on your recovery procedure; this tutorial does not perform a restore rehearsal.

You now have a database backup and protected configuration to copy off-host using your existing routine.

## Upgrades

Before upgrading, take a database backup and protect an off-host copy of `.env` and your current `compose.yml`. Record the running version and review the target release's notes for migration requirements.

Choose a target release containing the server manifest and set the shell `UZI_VERSION` to its number without `v`. Download its manifest, example environment, and initializer into a separate staging directory using the versioned URLs from Step 1. Compare them with your installed files before replacing anything. Carry forward local configuration edits, including the web port and optional `db-backup` service; merge any new required environment settings into your existing `.env`. Retain the original encryption key, database password, and other credentials. Do not regenerate `.env` or overwrite it with the new example.

Update the existing `UZI_VERSION=` assignment in `.env` to match the shell variable, then repeat the three explicit `cosign verify` checks in Step 1 for the target release. Stop on verification failure. Inspect the configuration privately and pull the verified release before recreating services:

```sh
docker compose -f compose.yml --profile agent config
docker compose -f compose.yml --profile agent pull db api web agent
docker compose -f compose.yml up -d --no-build db api web
```

Let active worker runs finish before recreating the agent. If the release changes the worker's baked toolchain, the persisted `agentnix` volume can retain an old store: stop and remove only the agent container, then remove only this project's `agentnix` volume by its exact name before recreating the worker. Preserve `pgdata` and `agentdata`; never use `down -v` to refresh the toolchain.

```sh
docker compose -f compose.yml --profile agent up -d --no-build agent
docker compose -f compose.yml --profile agent ps
```

Confirm service health, administrator login, and worker connectivity. Keep the same deployment directory and volumes. Database migrations run at API startup; check release-specific rollback requirements before attempting to run an older image against a migrated database.

## Conclusion

You have uzi behind your existing HTTPS proxy and a worker ready to accept work. Next, connect your forge using the [GitLab](./gitlab-bot-setup.md), [GitHub](./github-bot-setup.md), or [Forgejo](./forgejo-bot-setup.md) guide. Add [Claude credentials](./anthropic-token.md) or [Codex credentials](./codex-credentials.md), then review [Worker setup](./worker-setup.md) to run a worker on another host or explore further worker options before running your first issue.

## Reference

Current-branch evidence below explains the implementation boundaries. Compare it with your chosen release; later branch changes may not be present in its configuration or images.

| Scope | File and line evidence | What to check |
|---|---|---|
| Current branch | [scripts/init-env.sh:20](../scripts/init-env.sh#L20), [22](../scripts/init-env.sh#L22), [35](../scripts/init-env.sh#L35), [57](../scripts/init-env.sh#L57), [61](../scripts/init-env.sh#L61), [104](../scripts/init-env.sh#L104) | Downloaded layout, existing-file guard, secret generation, private file creation, and final write. |
| Current branch | [deploy/compose/compose.yml:26](../deploy/compose/compose.yml#L26), [117](../deploy/compose/compose.yml#L117), [119](../deploy/compose/compose.yml#L119), [124](../deploy/compose/compose.yml#L124), [125](../deploy/compose/compose.yml#L125), [136](../deploy/compose/compose.yml#L136), [140](../deploy/compose/compose.yml#L140), [162](../deploy/compose/compose.yml#L162), [173](../deploy/compose/compose.yml#L173) | Versioned application images, loopback port, worker profile, sizing, volumes, and secret wiring. |
| Current branch | [deploy/compose/compose.yml:33](../deploy/compose/compose.yml#L33), [80](../deploy/compose/compose.yml#L80), [88](../deploy/compose/compose.yml#L88), [.env.example:11](../.env.example#L11) | Origin, public URL, empty trusted proxies, and encryption-key loss warning. |
| Current branch | [api/cmd/server/main.go:1982](../api/cmd/server/main.go#L1982), [api/internal/handler/auth.go:96](../api/internal/handler/auth.go#L96), [206](../api/internal/handler/auth.go#L206), [api/internal/config/config.go:1464](../api/internal/config/config.go#L1464) | Create-only seed, registration closure, first-user admin assignment, and Secure-cookie derivation. |
| Current branch | [api/internal/middleware/ratelimit.go:229](../api/internal/middleware/ratelimit.go#L229), [239](../api/internal/middleware/ratelimit.go#L239), [web/nginx.conf:36](../web/nginx.conf#L36), [45](../web/nginx.conf#L45), [61](../web/nginx.conf#L61), [77](../web/nginx.conf#L77), [107](../web/nginx.conf#L107) | Direct-peer trust boundary, rightmost non-trusted hop, WebSocket route, and packaged forwarding-header overwrites. |
