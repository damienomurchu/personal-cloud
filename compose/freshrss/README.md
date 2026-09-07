# FreshRSS

Self-hosted RSS/Atom feed reader.

## Service

* Image: `freshrss/freshrss:1.29.1`
* Host port: `14030`
* Container port: `80`
* Bind: `127.0.0.1`
* Data: `~/personal-cloud-data/freshrss/data`
* Extensions: `~/personal-cloud-data/freshrss/extensions`
* Feed refresh: twice hourly at `:07` and `:37`

## Start

```bash
mkdir -p \
  ~/personal-cloud-data/freshrss/data \
  ~/personal-cloud-data/freshrss/extensions

docker compose up -d
```

## Status

```bash
docker compose ps
docker compose logs --tail=100
```

## Access

```text
http://127.0.0.1:14030
```

External access should be provided through the configured reverse proxy/tunnel.

Preferred subdomain:

```text
feeds.<domain>
```

## Initial Setup

Complete the FreshRSS web installer and select:

```text
Database: SQLite
```

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
~/personal-cloud-data/freshrss
```

This includes application data, SQLite state, configuration, users, feeds, and extensions.

## Restore

1. Stop FreshRSS.
2. Restore `~/personal-cloud-data/freshrss`.
3. Start FreshRSS.

```bash
docker compose down
docker compose up -d
```

## Troubleshooting

Logs:

```bash
docker compose logs -f
```

Health:

```bash
docker inspect --format='{{json .State.Health}}' "$(docker compose ps -q freshrss)"
```

Restart:

```bash
docker compose restart freshrss
```

