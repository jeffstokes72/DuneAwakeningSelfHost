# DASH self-host installer walkthrough

This walkthrough is for an end user who already has a Linux host suitable for
running the official Dune: Awakening Self-Hosted Server containers and wants to
use `scripts/install-self-host.sh` to prepare the local `.env` file.

The script does not download Funcom's server package and does not put secrets in
git. It writes local-only runtime settings into `.env`, generates RabbitMQ TLS
files under `config/tls/rabbitmq`, and can optionally start a single `Survival_1`
test stack.

## Hardware and platform baseline

Use a dedicated Linux host or VM with:

- x86_64 CPU with AVX2 exposed to the OS.
- Docker Engine with the Compose plugin (`docker compose`, not legacy
  `docker-compose`).
- Local SSD/NVMe storage for the repo, `data/`, and Postgres runtime files.
- Enough RAM and CPU for the number of map containers you intend to run.
- Public router/firewall forwarding for game traffic.
- Private LAN/VPN-only access for the admin panel.

Confidence: high. The repo's tested path assumes Linux containers, Docker
Compose, local filesystem performance, and AVX2. A single `Survival_1` server is
the right first target. The 9-map farm and 30-partition warm pool should be
started only after the single-map stack is healthy because each additional map
starts another game-server process.

## Inputs you need before running the script

Have these values ready:

| Input | What it means | Example placeholder |
| --- | --- | --- |
| Public IP or DNS name | Address players should reach from outside your LAN. | `<PUBLIC_IP>` |
| Funcom self-hosting token | Token from Funcom's account portal. | `<FUNCOM_TOKEN>` |
| Admin password/token | Token used for the local admin panel APIs. | `<ADMIN_PASSWORD>` |
| Steam server directory | Installed official Steam tool path on the Linux host. | `/srv/dune-steam-server` |
| World name | Display name for your server. | `My Dune Server` |
| World unique name | Stable lowercase-ish identifier for service users. | `my-dune-server` |

Keep real tokens and passwords out of shell history, chat logs, committed files,
and screenshots. The examples below use placeholders.

## Install required host tools

On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl jq ripgrep openssl rsync
```

Install Docker Engine from your distribution or Docker's official packages, then
confirm:

```bash
docker compose version
jq --version
rg --version
openssl version
```

The official Steam tool must already be installed on the Linux host or copied to
a local Linux path. Do not commit the Steam package, image tarballs, or extracted
Funcom content into this repo.

## Recommended safe command

Run from the repo checkout:

```bash
cd /srv/DuneAwakeningSelfHost
```

Read secrets without echoing them to the terminal:

```bash
read -r -s -p "Funcom self-hosting token: " DASH_FUNCOM_TOKEN
echo
read -r -s -p "Admin password/token: " DASH_ADMIN_PASSWORD
echo
export DASH_FUNCOM_TOKEN DASH_ADMIN_PASSWORD
```

Create or update `.env` without starting containers:

```bash
./scripts/install-self-host.sh \
  --public-ip "<PUBLIC_IP>" \
  --steam-dir "/srv/dune-steam-server" \
  --world-name "My Dune Server" \
  --world-unique-name "my-dune-server" \
  --world-region "North America"
```

This is configuration-only mode. It is the safest first run because it writes the
env file and TLS files but does not start Docker services.

## What the script changes

The installer first delegates env/TLS creation to:

```bash
./scripts/populate-local-env.sh .env
```

That helper:

- Copies `.env.example` to `.env` if `.env` does not exist.
- Generates random local database, replication, and RabbitMQ secrets.
- Generates RabbitMQ TLS material under `config/tls/rabbitmq`.
- Leaves an existing `.env` unchanged before the installer applies the selected
  values.

Then `install-self-host.sh` writes or updates these important `.env` keys:

| Key | Value source |
| --- | --- |
| `DUNE_STEAM_SERVER_DIR` | `--steam-dir` |
| `WORLD_NAME` | `--world-name` or current env/default |
| `WORLD_UNIQUE_NAME` | `--world-unique-name` or generated/current env |
| `WORLD_REGION` | `--world-region` or current env/default |
| `DUNE_SERVER_DISPLAY_NAME` | Same as `WORLD_NAME` |
| `DUNE_SERVER_LOGIN_PASSWORD` | Same as admin password unless overridden |
| `FLS_SECRET` | `DASH_FUNCOM_TOKEN`, prompt input, or `--funcom-token` |
| `DUNE_ADMIN_TOKEN` | `DASH_ADMIN_PASSWORD` or `--admin-password` |
| `DUNE_ADMIN_REQUIRE_TOKEN` | `true` |
| `EXTERNAL_ADDRESS` | `--public-ip` |
| `GAME_RMQ_PUBLIC_HOST` | Same as `--public-ip` |
| `DUNE_ANNOUNCE_HOST_WORKSPACE` | Current repo path |
| `DUNE_RESTART_HOST_WORKSPACE` | Current repo path |
| `DUNE_ANNOUNCE_RMQ_USER` | Derived from `WORLD_UNIQUE_NAME` |

If you want a different player login password than the admin token, pass:

```bash
--server-login-password "<PLAYER_LOGIN_PASSWORD>"
```

If you want no player login password, pass:

```bash
--no-server-login-password
```

## Validate before startup

After the configuration-only run:

```bash
./scripts/bootstrap-checklist.sh .env
./scripts/preflight.sh .env
```

`bootstrap-checklist.sh` is read-only and reports missing tools, placeholder env
values, and common new-host mistakes. `preflight.sh` is stricter and checks
required env values, Steam image tarball paths, and unsafe host bindings.

If `DUNE_IMAGE_TAG` does not match the Steam package, update it from the package:

```bash
./scripts/check-steam-update.sh .env --write-env
```

## Start a single-map test stack

After validation passes, either rerun the installer with startup enabled:

```bash
./scripts/install-self-host.sh \
  --public-ip "<PUBLIC_IP>" \
  --steam-dir "/srv/dune-steam-server" \
  --world-name "My Dune Server" \
  --world-unique-name "my-dune-server" \
  --world-region "North America" \
  --start-single-map
```

or run the normal steps manually:

```bash
./scripts/load-images.sh .env
docker compose --env-file .env config --quiet
docker compose --env-file .env up -d postgres admin-rmq game-rmq
docker compose --env-file .env run --rm db-init
docker compose --env-file .env up -d rmq-auth-shim text-router gateway director
./scripts/single-survival-partition.sh .env
docker compose --env-file .env up -d admin-panel
./scripts/status.sh .env
```

The `--start-single-map` mode runs the same flow:

1. Checks the Steam package image tag and writes it to `.env` when it can.
2. Loads the official Funcom image tarballs with Docker.
3. Validates Compose config.
4. Starts Postgres and both RabbitMQ services.
5. Bootstraps the database.
6. Starts the auth shim, text router, gateway, and director.
7. Prunes extra unused `Survival_1` partitions for a one-map test unless
   `--skip-single-partition` is set.
8. Starts `survival` and `admin-panel`.
9. Prints current server status.

## Firewall and router forwarding

For a single `Survival_1` test, forward:

```text
7777/udp  -> Dune host
31982/tcp -> Dune host
```

For the 30-partition warm pool, forward:

```text
7777-7806/udp -> Dune host
31982/tcp     -> Dune host
```

Keep Postgres, RabbitMQ management ports, and the admin panel private. Do not
publish the admin panel directly to the internet.

Example UFW scripts live under:

```text
examples/firewall/ufw-single-map.sh
examples/firewall/ufw-full-warm-pool.sh
```

## Admin panel

The installer enables admin token auth:

```env
DUNE_ADMIN_REQUIRE_TOKEN=true
```

The admin token is the admin password/token you provided to the installer. The
default local URL is:

```text
http://127.0.0.1:18080/
```

Use SSH port forwarding or a trusted LAN/VPN reverse proxy for remote admin
access. Keep the panel bound to localhost or a private trusted interface.

## Expanding after the single-map test

Once the single map is healthy and login works, expand deliberately:

- Use `docs/setup.md` for the 9-map standing farm.
- Use `./scripts/start-full-warm-pool.sh .env` for the 30-partition warm pool.
- Use `COMPOSE_FILES='compose.yaml:compose.allmaps.yaml' ./scripts/rmq-health.sh .env`
  to validate the all-map RabbitMQ/auth path.
- Watch host memory, disk, and container restart counts from the admin panel Ops
  page or with `./scripts/profile-runtime.sh .env`.

Confidence: moderate. The exact resource ceiling depends on your CPU, RAM,
storage, image tag, map count, and player behavior. The repo has observed the
game-server image and process as the dominant storage and memory consumers, so
start small and measure before expanding.

## Common failures

### `set --steam-dir to the Steam tool install path`

`DUNE_STEAM_SERVER_DIR` is still empty or still the placeholder from
`.env.example`. Pass the real Linux path to the official Steam tool directory:

```bash
--steam-dir "/path/to/Dune Awakening Self-Hosted Server"
```

### `set --public-ip to a public or routable address`

The script refuses `127.0.0.1` for public self-hosting. Pass the WAN address or
DNS name that clients should use.

### `WORLD_UNIQUE_NAME is empty`

Pass a stable unique name:

```bash
--world-unique-name "my-dune-server"
```

Use one value for the life of the world unless you are intentionally rebuilding
service identities.

### Client sees the server but cannot finish login

Check:

- `FLS_SECRET` is set in `.env`.
- `EXTERNAL_ADDRESS` is the public address clients use.
- `GAME_RMQ_PUBLIC_HOST` matches that public address or public DNS name.
- `GAME_RMQ_PUBLIC_PORT` is reachable from clients, default `31982/tcp`.
- Router/firewall forwarding is in place.

### Admin panel asks for a token

That is expected after running this installer. Enter the admin password/token
you provided to the installer.

## Cleanup local shell secrets

After the script finishes, clear exported secrets from your shell:

```bash
unset DASH_FUNCOM_TOKEN DASH_ADMIN_PASSWORD DASH_SERVER_LOGIN_PASSWORD
```

The real values remain in local `.env`, which is intentionally ignored by git.
