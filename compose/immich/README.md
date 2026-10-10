# Immich evaluation stack

Local-only, disposable deployment for evaluating a selected set of photos. Persistent files live under `~/personal-cloud-data/immich`; Compose configuration can live anywhere.

## Requirements

- Docker Engine or OrbStack with `docker compose`
- Approximately 6 GB RAM available for Immich and its database/ML processes
- An SSD-backed, local filesystem for PostgreSQL (never an SMB/NFS share)
- The `HOME` environment variable must be set in the shell invoking Compose

## Layout

```text
~/personal-cloud-data/immich/
├── library/       # Immich-managed originals, derivatives and metadata files
├── postgres/      # PostgreSQL data
├── model-cache/   # Downloaded machine-learning models
└── external/      # Your test images (mounted read-only)
```

## First start

From the directory containing `docker-compose.yml` and `.env`:

```sh
mkdir -p "$HOME/personal-cloud-data/immich"/{library,postgres,model-cache,external}
# Generate a new password; copy the result into DB_PASSWORD in .env:
openssl rand -hex 24
chmod 600 .env
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

Open http://127.0.0.1:2283 and create the initial admin user. The service is bound to localhost only, not your LAN or Tailscale network.

## Evaluate photos without duplicating originals (recommended)

Place **copies of a selected sample** under `~/personal-cloud-data/immich/external`. In Immich, open **Administration → External Libraries**, create a library for your user, add the import path `/mnt/evaluation`, and scan. The host directory is mounted read-only, so Immich cannot change or delete your source images. Generated thumbnails, embeddings and database metadata still live in the persistent stack data directories.

You can instead upload photos through the web UI, but those become Immich-managed copies under `library/`. If you change files inside `external/`, re-scan the library. Edits to external asset metadata may not work with the read-only mount.

## Operations

```sh
docker compose ps
docker compose logs -f immich-server
docker compose logs -f immich-machine-learning
docker compose stop               # Preserve all data; stop containers
docker compose up -d              # Start again
docker compose down               # Remove containers/network, preserve bind-mounted data
```

## Upgrading

`IMMICH_VERSION` is explicitly pinned, so there are no surprise application upgrades. Review [Immich release notes](https://github.com/immich-app/immich/releases) and the [current upstream Compose file](https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml) before changing it. Database/Valkey compatibility may change between versions. Back up `library/` and `postgres/` consistently before upgrading. Never automatically update this stack with Watchtower.

## Full reset (destructive)

First run `docker compose down`. To remove the evaluation data, manually delete **only** `~/personal-cloud-data/immich` after verifying it contains no irreplaceable images:

```sh
# Inspect first; deletion is intentionally not automated.
ls -la "$HOME/personal-cloud-data/immich"
```

`docker compose down -v` by itself does **not** delete the bind-mounted directories above.

## Security / scope

- No reverse proxy or automatic mobile uploads configured.
- Port 2283 is bound to `127.0.0.1` only.
- Database and Valkey are not published to the host.
- `.env` contains a database credential; do not commit it to Git.
- Only use copies of valuable photos for evaluation, and preserve original backups.
- Host paths assume Docker is running on the machine holding `~/personal-cloud-data/immich`.

References: https://docs.immich.app/install/docker-compose/ and https://docs.immich.app/features/libraries/
