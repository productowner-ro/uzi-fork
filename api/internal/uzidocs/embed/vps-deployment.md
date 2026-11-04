---
title: Run uzi on a VPS with Docker Compose
order: 15
audience: operator
---

# Run uzi on a VPS with Docker Compose

This tutorial adds uzi to a VPS that you already run. At the end, uzi answers at `https://your_domain`, and a worker is ready to take an issue from your repository.

uzi runs as a Docker Compose stack of four services:

1. `db` keeps the data in PostgreSQL.
2. `api` signs users in, keeps the configuration and hands work to workers.
3. `web` serves the interface and passes API calls to `api`.
4. `agent` is the worker: it runs the work against your repositories.

Only `web` opens a port, and only on `127.0.0.1`. Your existing reverse proxy brings HTTPS traffic to it.

The stack comes from the [server Compose file](../deploy/compose/compose.yaml) of a release, with signed images of the same release. Its worker has no Docker inside; see [Worker setup](./worker-setup.md) for other workers.

Allow about 15 minutes once the prerequisites are ready. A script of these steps took about 3 minutes on a new 4 vCPU, 8 GB server, and the image download took 2 of them.

New words such as forge, worker or plan gate are in the [Glossary](./glossary.md).

## Prerequisites

**The server**

1. A Debian or Ubuntu VPS, an account that can use `sudo`, and a firewall that lets HTTP and HTTPS in. See [Initial Server Setup with Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu).
2. Docker Engine and the Docker Compose v2 plugin. Follow Docker's [Ubuntu guide](https://docs.docker.com/engine/install/ubuntu/), then add your account to the `docker` group with the [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) and log in again.
3. 4 vCPUs, 8 GB of RAM and 20 GB of free disk for uzi alone. The worker takes 2 CPUs and 4 GiB of these. See [worker sizing](./worker-setup.md#resource-stats-and-sizing). If your provider has no server of this size, choose one by your needs.
4. `curl`, Bash, `openssl`, `jq` and `cosign`. Install `cosign` from its [`.deb` package](https://docs.sigstore.dev/cosign/system_config/installation/), because `apt` on Ubuntu does not carry it.

**The domain and the proxy**

5. A domain whose A record points to your VPS. Replace `your_domain` with it in each command below.
6. A reverse proxy that ends TLS. If you have none, install [Caddy](https://caddyserver.com/docs/install#debian-ubuntu-raspbian): it gets the certificate by itself. For nginx, see [Other setups](#other-setups).

**The accounts**

7. A bot account on your forge, with its token, added to your repository. uzi works through this account, never through yours. Follow steps 1 to 4 of the [GitHub bot setup](./github-bot-setup.md). For GitLab or Forgejo, follow the [GitLab](./gitlab-bot-setup.md) or [Forgejo](./forgejo-bot-setup.md) guide.
8. A model login. For Codex, sign in with a login only for uzi, as in step 1 of [Codex credentials](./codex-credentials.md#1-sign-in-with-a-login-dedicated-to-uzi). A person must do this sign-in. For Claude, see [Anthropic token](./anthropic-token.md).

**The backup**

9. A backup routine that copies files off the VPS. We recommend [restic](https://restic.net/).

## Step 1 - Download the release files

Create the folder of the stack and enter it. Keep this path: Compose names the stack and its volumes after the folder.

```sh
sudo mkdir -p /opt/uzi
sudo chown "$USER":"$(id -gn)" /opt/uzi
cd /opt/uzi
```

Choose a release that contains `deploy/compose/compose.yaml` on [GitHub Releases](https://github.com/vtmocanu/uzi/releases). Set its number, with no leading `v`:

```sh
UZI_VERSION='<release-number-without-v>'
```

Download the three files of that release. `set -e` stops the block at the first failed download:

```bash
set -e
BASE="https://raw.githubusercontent.com/productowner-ro/uzi-fork/v${UZI_VERSION}"
curl -fsSLo compose.yaml "$BASE/deploy/compose/compose.yaml"
curl -fsSLo .env.example "$BASE/.env.example"
mkdir -p scripts
curl -fsSLo scripts/init-env.sh "$BASE/scripts/init-env.sh"
```

Create `.env` and add the version to it. The script writes three new secrets and sets the file to mode `0600`. It never overwrites an existing `.env`:

```sh
bash scripts/init-env.sh
printf '\nUZI_VERSION=%s\n' "$UZI_VERSION" >> .env
```

Check the signatures of the three images. Stop if one check fails:

```sh
for image in api web agent-base; do
  cosign verify \
    --certificate-identity "https://github.com/vtmocanu/uzi/.github/workflows/release.yml@refs/tags/v${UZI_VERSION}" \
    --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
    "ghcr.io/vtmocanu/uzi/${image}:${UZI_VERSION}" > /dev/null && echo "$image ok"
done
```

See [Container signing](./container-signing.md) for what the signature proves.

## Step 2 - Set your values in `.env`

Open `.env` in your editor. Each entry below is already in the file: change it in place, and do not add a second copy. Do not print the file or paste it anywhere, because it holds secrets.

You must change the first five values. Set the other values as shown:

```dotenv
# --- Change these ---

# Your public address. Both values must be the same https URL.
# The https scheme marks the session cookie Secure.
FRONTEND_ORIGIN=https://your_domain
UZI_PUBLIC_BASE_URL=https://your_domain

# The first admin account. The API creates it at its first start.
# Use 12 characters or more, in single quotes.
UZI_SEED_EMAIL=you@example.com
UZI_SEED_PASSWORD='replace-with-a-unique-admin-password'

# The forge that uzi may connect to. Use https://gitlab.com or your own
# GitLab or Forgejo URL for those forges.
FORGE_ALLOWED_BASE_URLS=https://github.com

# --- Set these as shown ---

# Nobody else can create an account.
UZI_REGISTRATION_ENABLED=false
# The display name of the first admin account.
UZI_SEED_NAME=Administrator
# Empty: the API trusts no forwarded client address. See "Other setups".
TRUSTED_PROXIES=
# Empty for now. Step 6 writes the worker token here.
UZI_WORKER_TOKEN=
```

[Configuration](./configuration.md) explains every other value, and `.env.example` carries a comment for each one.

**Copy `.env` to your backup now.** It holds `UZI_SECRET_KEY`, which encrypts the forge and model tokens. If you lose this key, uzi cannot read them.

## Step 3 - Start uzi

Download the images. The worker image alone is about 10 GB:

```sh
docker compose --profile agent pull
```

Start the database, the API and the web service. The worker stays off until Step 6:

```sh
docker compose up -d
```

Wait until `db` and `api` show `healthy`:

```sh
docker compose ps
```

The API updates the database at each start. If a service fails, read its log with `docker compose logs api`. Never run `docker compose down -v`: it deletes the data.

## Step 4 - Send HTTPS traffic to uzi

Add this site to `/etc/caddy/Caddyfile`. You can keep the default `:80` block of a new Caddy install:

```caddyfile
your_domain {
    reverse_proxy 127.0.0.1:8080
}
```

Check the file, then reload Caddy:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
```

Open `https://your_domain` and sign in with the email and password of the first admin account. Right after the reload, the first visit can fail with a TLS error while Caddy gets its certificate. Wait a few seconds and try again.

## Step 5 - Sign in from the shell

Steps 6 and 7 call the uzi API with your admin session. The `uzi` CLI cannot do these steps yet: [vtmocanu/uzi#3760](https://github.com/vtmocanu/uzi/issues/3760) asks for a setup command on the server for them.

```sh
URL=https://your_domain
# Type the admin password. The terminal does not show it.
read -rs UZI_PASSWORD
# Sign in, and keep the session cookie in the file uzi-cookies.
jq -n --arg email you@example.com --arg password "$UZI_PASSWORD" '{email:$email,password:$password}' \
  | curl -fsS -c uzi-cookies -H "Origin: $URL" -H 'Content-Type: application/json' --data @- -o /dev/null "$URL/api/auth/login"
# Each call that changes data needs the CSRF value of the session.
CSRF=$(awk '$6=="uzi_csrf"{print $7}' uzi-cookies)
# api sends the cookie and the CSRF value with each call.
api() { curl -fsS -b uzi-cookies -H "Origin: $URL" -H "X-CSRF-Token: $CSRF" -H 'Content-Type: application/json' "$@"; }
```

You delete `uzi-cookies` at the end of Step 7.

## Step 6 - Start the worker

Create the worker, and write its join token into `.env`. uzi shows the token only once. The name is what the **Workers** page shows:

```sh
TOKEN=$(api -d '{"name":"your_worker","template":"base"}' "$URL/api/workers" | jq -r .token)
sed -i "s|^UZI_WORKER_TOKEN=.*|UZI_WORKER_TOKEN=$TOKEN|" .env
unset TOKEN
```

Start the worker:

```sh
docker compose --profile agent up -d agent
```

Wait until the worker shows `online`. In our test this took about a minute:

```sh
api "$URL/api/workers" | jq -r '.workers[] | [.name, .status] | @tsv'
```

If it stays `offline`, read `docker compose logs agent`. See [Worker setup](./worker-setup.md) for more help.

## Step 7 - Connect the forge, the repository and the model login

Connect the bot token from the prerequisites, then check its rights. The check prints `ok`:

```sh
# Type the bot token. The terminal does not show it.
read -rs BOT_TOKEN
# Always send forge_type: the API picks GitLab when it is missing.
CONNECTION=$(jq -n --arg token "$BOT_TOKEN" '{base_url:"https://github.com",forge_type:"github",token:$token}' \
  | api --data @- "$URL/api/forge/connections" | jq -r .connection.id)
api -X POST "$URL/api/forge/connections/$CONNECTION/privilege-check" | jq -r .report.status
```

Turn on your repository, then open its board once. The first open creates the board columns, and runs need them (see [Board](./board.md#columns)):

```sh
REPO=$(api "$URL/api/forge/connections/$CONNECTION/projects" \
  | jq -r '.repos[] | select(.path_with_namespace=="your_org/your_repo") | .id')
api -X PUT -d '{"enabled":true}' "$URL/api/repos/$REPO" > /dev/null
api "$URL/api/repos/$REPO/board" | grep -q 'Human Review' && echo 'board ready'
```

Copy the `auth.json` that the Codex sign-in wrote into `/opt/uzi`, then store the login:

```sh
jq '{token: ({access_token:.tokens.access_token, refresh_token:.tokens.refresh_token} | tojson), label:"uzi-codex", default:true}' auth.json \
  | api --data @- "$URL/api/me/secrets/codex_auth" > /dev/null
```

Delete `auth.json` and `uzi-cookies`.

Your worker can now take an issue from this repository.

## Step 8 - Back up uzi

A backup needs two things together: a database dump and `.env`. A dump without the `UZI_SECRET_KEY` of its `.env` cannot give back the encrypted tokens.

Write a dump of the database:

```sh
docker compose exec -T db pg_dump -U uzi -Fc uzi > "uzi-$(date +%F).dump"
```

Copy the dump, `.env` and `compose.yaml` off the VPS with your backup routine. Test a restore away from the live database before you rely on it.

For a backup on a timer, see [A backup container](#a-backup-container).

## Upgrades

1. Take a backup, as in Step 8.
2. Download the files of the new release into a separate folder, with the commands of Step 1. Compare them with your files, and carry your own changes over. Never overwrite `.env`: add only the new entries that the release notes ask for.
3. Set `UZI_VERSION` in `.env` to the new number, and check the signatures as in Step 1.
4. Wait until the worker has no active run. Then pull and restart:

```sh
docker compose --profile agent pull
docker compose up -d
docker compose --profile agent up -d agent
```

Check that you can sign in and that the worker shows `online`. Keep the same folder and volumes. The worker keeps its tools in the `agentnix` volume, so a new image does not refresh them; see [Tool provisioning](./worker-setup.md#tool-provisioning) to refresh them. An older image may not run on a database that a newer API updated: read the release notes before you go back.

## Other setups

These cover the cases that the main path leaves out.

**Port 8080 is in use.** In `compose.yaml`, change the one entry under `services.web.ports` to `"127.0.0.1:18080:8080"`. Then use port `18080` in your proxy.

**nginx instead of Caddy.** Add these locations to the TLS `server` block of `your_domain`. The `/api/ws` location keeps the live updates open:

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

Check with `sudo nginx -t`, then run `sudo systemctl reload nginx`.

**The proxy runs in a container.** There, `127.0.0.1` is the proxy container, not the VPS. Connect the proxy to the `web` service on a Docker network instead. Keep `api` and `db` without a published port.

**All visitors share one sign-in limit.** With `TRUSTED_PROXIES` empty, the API counts sign-in attempts for all visitors together, so one noisy client can block the others for a minute. This is the safe setting. See [Auth design](./auth-design.md#rate-limiting-and-the-x-forwarded-for-trust-model) before you change it.

**Create the first admin by sign-up instead of `.env`.** Leave the three `UZI_SEED_*` values empty and set `UZI_REGISTRATION_ENABLED=true`. The first account to sign up becomes admin, so keep the site closed to others until you sign up. Then set `UZI_REGISTRATION_ENABLED=false` and run `docker compose up -d --force-recreate api`.

**Check the session cookie.** In your browser's developer tools, the `uzi_auth` cookie must show `Secure` and `HttpOnly`. If `Secure` is missing, `FRONTEND_ORIGIN` does not start with `https://`. Correct it, run `docker compose up -d --force-recreate api`, and sign in again.

**A run stops at the plan and asks for Docker.** Before it plans, uzi scans your repository. A `Dockerfile`, a `docker-compose` file or a testcontainers dependency marks the run as one that needs Docker. The worker of this tutorial has no Docker, so the run stops at the plan, even when the plan uses no Docker. If the work needs no Docker, approve with `uzi run approve <run-id> --override-capabilities`, or press **Run without docker** in the web interface. See [Capability scheduling](./capability-scheduling.md) and [Docker inside a worker](./worker-docker.md).

**A run stops for your decision after a failed check.** This can happen when a check fails for a reason outside the change. See [Completion holds](./run-completion-hold.md).

### A backup container

This optional service writes a daily database dump to `/opt/uzi/backups` on the VPS. It uses [nfrastack/db-backup 5.0.5](https://nfrastack.com/db-backup/container). Your backup routine must still copy the dumps off the VPS.

Add it under the existing `services` key of `compose.yaml`, next to `db`:

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

Create the folder for its user `10000`, start the service, and make a first dump at once:

```sh
sudo install -d -m 0700 -o 10000 -g 10000 /opt/uzi/backups
docker compose up -d db-backup
docker compose exec db-backup backup-now
```

See the [backup](https://nfrastack.com/db-backup/recipes/backup) and [restore](https://nfrastack.com/db-backup/recipes/restore) recipes for the options.

## Reference

The evidence below is from the current branch. Compare it with your release: a later change may not be in its files or images.

| Scope | File and line evidence | What to check |
|---|---|---|
| Current branch | [scripts/init-env.sh:20](../scripts/init-env.sh#L20), [22](../scripts/init-env.sh#L22), [35](../scripts/init-env.sh#L35), [57](../scripts/init-env.sh#L57), [61](../scripts/init-env.sh#L61), [104](../scripts/init-env.sh#L104) | Downloaded layout, existing-file guard, secret generation, private file creation, and final write. |
| Current branch | [deploy/compose/compose.yaml:26](../deploy/compose/compose.yaml#L26), [119](../deploy/compose/compose.yaml#L119), [121](../deploy/compose/compose.yaml#L121), [126](../deploy/compose/compose.yaml#L126), [127](../deploy/compose/compose.yaml#L127), [138](../deploy/compose/compose.yaml#L138), [142](../deploy/compose/compose.yaml#L142), [166](../deploy/compose/compose.yaml#L166), [177](../deploy/compose/compose.yaml#L177) | Versioned application images, loopback port, worker profile, sizing, volumes, and secret wiring. |
| Current branch | [deploy/compose/compose.yaml:33](../deploy/compose/compose.yaml#L33), [80](../deploy/compose/compose.yaml#L80), [88](../deploy/compose/compose.yaml#L88), [.env.example:11](../.env.example#L11) | Origin, public URL, empty trusted proxies, and encryption-key loss warning. |
| Current branch | [api/cmd/server/main.go:1982](../api/cmd/server/main.go#L1982), [api/internal/handler/auth.go:96](../api/internal/handler/auth.go#L96), [206](../api/internal/handler/auth.go#L206), [api/internal/config/config.go:1464](../api/internal/config/config.go#L1464) | Create-only seed, registration closure, first-user admin assignment, and Secure-cookie derivation. |
| Current branch | [api/internal/middleware/ratelimit.go:229](../api/internal/middleware/ratelimit.go#L229), [239](../api/internal/middleware/ratelimit.go#L239), [web/nginx.conf:36](../web/nginx.conf#L36), [45](../web/nginx.conf#L45), [61](../web/nginx.conf#L61), [77](../web/nginx.conf#L77), [107](../web/nginx.conf#L107) | Direct-peer trust boundary, rightmost non-trusted hop, WebSocket route, and packaged forwarding-header overwrites. |
| Current branch | [api/internal/handler/forge.go:243](../api/internal/handler/forge.go#L243), [api/internal/handler/cli_auth_livedb_test.go:498](../api/internal/handler/cli_auth_livedb_test.go#L498) | The GitLab default for a missing `forge_type`, and the session-only routes for worker creation and forge connection. |
