# Umami

Self-hosted web analytics.

## Service

| Item            | Value                                  |
| --------------- | -------------------------------------- |
| Host            | `mgt-1`                                |
| Container       | `umami`                                |
| Database        | PostgreSQL 15                          |
| Local port      | `127.0.0.1:3002`                       |
| Persistent data | `~/personal-cloud-data/umami/postgres` |
| Exposure        | Cloudflare Tunnel                      |

## Files

```text
.
├── docker-compose.yml
├── .env
└── README.md
```

Runtime data:

```text
~/personal-cloud-data/umami/
└── postgres/
```

## Configuration

Secrets are stored in `.env`.

Required:

```dotenv
UMAMI_DB_PASSWORD=
UMAMI_APP_SECRET=
UMAMI_TWO_FACTOR_ENCRYPTION_KEY=
```

Generate secrets with:

```bash
openssl rand -hex 32
```

Do not commit `.env`.

## Deploy

```bash
mkdir -p ~/personal-cloud-data/umami/postgres

docker compose pull
docker compose up -d
```

## Status

```bash
docker compose ps
```

Health endpoint:

```bash
curl http://127.0.0.1:3002/api/heartbeat
```

## Logs

```bash
docker compose logs -f umami
```

Database logs:

```bash
docker compose logs -f db
```

## Restart

```bash
docker compose restart
```

## Stop

```bash
docker compose down
```

Persistent database data is retained.

## Update

Update the pinned Umami image version in `docker-compose.yml`.

Then:

```bash
docker compose pull
docker compose up -d
docker image prune
```

Verify:

```bash
docker compose ps
curl http://127.0.0.1:3002/api/heartbeat
```

## Cloudflare Tunnel

Tunnel target:

```text
http://localhost:3002
```

The tracking endpoint must remain publicly reachable.

Do not protect the entire Umami hostname with mandatory Cloudflare Access authentication unless tracking endpoints are explicitly bypassed.

## Initial Login

```text
Username: admin
Password: umami
```

Change the default password immediately after deployment.

## Backup

Do not rely on filesystem backup of the live PostgreSQL data directory.

Create logical database dumps and include them in the existing Restic backup workflow.

Example:

```bash
mkdir -p ~/personal-cloud-data/umami/backups

docker compose exec -T db \
  pg_dump -U umami umami \
  | gzip > ~/personal-cloud-data/umami/backups/umami.sql.gz
```

## Restore

Stop Umami:

```bash
docker compose stop umami
```

Restore database:

```bash
gunzip -c ~/personal-cloud-data/umami/backups/umami.sql.gz \
  | docker compose exec -T db psql -U umami umami
```

Start Umami:

```bash
docker compose start umami
```

## Useful Commands

```bash
docker compose ps
docker compose logs -f
docker compose restart
docker compose pull
docker compose up -d
docker compose down
```

