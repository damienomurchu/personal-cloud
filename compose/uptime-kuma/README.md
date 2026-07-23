# Uptime Kuma

Self-hosted status/monitoring dashboard.

Runs on the X1 management node and is exposed externally via Cloudflare Tunnel:

```text
status.damienmurphy.net
```

## Compose

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped

    ports:
      - "127.0.0.1:3001:3001"

    volumes:
      - /Users/me/code/data/uptime-kuma:/app/data
```

## Local endpoint

```text
http://127.0.0.1:3001
```

Bound to localhost only.

No LAN exposure.
No direct public exposure.
Cloudflare Tunnel is the public ingress path.

## Public endpoint

```text
https://status.damienmurphy.net
```

Cloudflare Tunnel should route:

```text
status.damienmurphy.net -> http://127.0.0.1:3001
```

## Data

Persistent app data lives here:

```text
/Users/me/code/data/uptime-kuma
```

Back this up.

Contains monitor config, history, settings, and local application state.

## Start

From this directory:

```bash
docker compose up -d
```

## Stop

```bash
docker compose down
```

## Restart

```bash
docker compose restart
```

## Logs

```bash
docker compose logs -f
```

## Upgrade

```bash
docker compose pull
docker compose up -d
```

## Verify

Local:

```bash
curl -I http://127.0.0.1:3001
```

Container:

```bash
docker ps --filter name=uptime-kuma
```

Tunnel/public:

```bash
curl -I https://status.damienmurphy.net
```

DNS:

```bash
dig status.damienmurphy.net
```

## Cloudflare Tunnel notes

Expected setup:

```text
Public hostname: status.damienmurphy.net
Service:         http://127.0.0.1:3001
```

If public access fails:

1. Check container is running.
2. Check local endpoint responds.
3. Check Cloudflare Tunnel is running on the node.
4. Check hostname route points at `http://127.0.0.1:3001`.
5. Flush local DNS cache if hostname behaviour looks stale.

## Monitors to add

Suggested first monitors:

* `ai.damienmurphy.net`
* `books.damienmurphy.net`
* `status.damienmurphy.net`
* Cloudflare Tunnel health
* Local router/gateway
* Mac mini SSH
* X1 SSH
* Paperless-ngx, when deployed
* Open WebUI
* Calibre-Web

## Security posture

Uptime Kuma listens on localhost only:

```text
127.0.0.1:3001
```

Do not change to:

```text
3001:3001
```

unless deliberately exposing it on the host network.

Public access should remain mediated by Cloudflare Tunnel / Zero Trust policy.

## Backup priority

High.

