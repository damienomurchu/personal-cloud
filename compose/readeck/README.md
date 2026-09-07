# Readeck

Self-hosted read-later and article archiving service.

## Service

* Image: `codeberg.org/readeck/readeck:0.23.2`
* Host port: `14020`
* Container port: `8000`
* Bind: `127.0.0.1`
* Data: `~/personal-cloud-data/readeck`

## Start

```bash
mkdir -p ~/personal-cloud-data/readeck
docker compose up -d
```

## Status

```bash
docker compose ps
docker compose logs --tail=100
```

## Access

```text
http://127.0.0.1:14020
```

External access should be provided through the configured reverse proxy/tunnel rather than exposing the host port publicly.

## Stop

```bash
docker compose down
```

## Update

Update the pinned image tag in `docker-compose.yml`, then:

```bash
docker compose pull
docker compose up -d
```

Verify:

```bash
docker compose ps
docker compose logs --tail=100
```

## Backup

Back up:

```text
~/personal-cloud-data/readeck
```

This contains Readeck configuration, database, archived content, and application state.

## Restore

1. Stop Readeck.
2. Restore `~/personal-cloud-data/readeck`.
3. Start Readeck.

```bash
docker compose down
docker compose up -d
```

## Troubleshooting

```bash
docker compose ps
docker compose logs -f
docker inspect --format='{{json .State.Health}}' "$(docker compose ps -q readeck)"
```

Restart:

```bash
docker compose restart readeck
```

