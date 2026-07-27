# Service Context

This file summarizes the current service inventory. Each service README and
Compose definition remain authoritative for service-specific work.

## Inventory

| Service | Host | Browser hostname | Origin | Durable state |
| --- | --- | --- | --- | --- |
| Open WebUI | Mac mini | `ai.damienmurphy.net` | Configurable localhost host port to container port `8080` | Open WebUI data |
| Calibre-Web | Mac mini | `books.damienmurphy.net` | Configurable localhost host port to container port `8083` | Application configuration and Calibre library |
| Uptime Kuma | ThinkPad X1 | `status.damienmurphy.net` | `127.0.0.1:3001` | Uptime Kuma application data |

All three browser hostnames use Cloudflare Tunnel and Cloudflare Access.
Administrative host access remains private through Tailscale or the trusted
local network.

## Open WebUI

Purpose: private browser interface for locally hosted language models.

Important dependencies:

- OrbStack and Compose-compatible tooling
- local Ollama or an equivalent API-compatible model runtime
- Cloudflare Tunnel, DNS, and Access
- persistent local storage
- Tailscale for host administration

Durable state may include accounts, conversations, settings, uploaded knowledge
files, connection configuration, and application metadata. Model files are
managed separately and may be reproducible.

Validation must cover more than HTTP availability:

- Compose and container state
- local application response
- model discovery and connectivity
- successful inference
- unauthorized Cloudflare access denied
- authorized external access succeeds

Source files:

- `compose/openweb-ui/README.md`
- `compose/openweb-ui/docker-compose.yml`

## Calibre-Web

Purpose: private browser interface for browsing, managing, and accessing the
personal ebook library.

Durable state consists of:

- application configuration, users, settings, `app.db`, and optional integration
  data
- the Calibre library, including `metadata.db`, ebooks, covers, and metadata

The ebook library is the higher-value data set. Do not create a new library or
database when data appears missing until mount paths and the original data
location have been verified.

Validation should confirm:

- Compose and container health
- library detection
- visible books and metadata
- successful open, read, or download
- unauthorized Cloudflare access denied
- authorized external access succeeds

Source files:

- `compose/calibre-web/README.md`
- `compose/calibre-web/docker-compose.yml`

## Uptime Kuma

Purpose: independent availability monitoring and operational status. It is part
of the management plane rather than a user workload.

Most monitor configuration is stored in the application database. Durable state
includes monitors, history, users, notification settings and credentials,
status pages, and application settings.

Uptime Kuma must remain independent of the Mac mini. It should eventually be
checked by an independent mechanism because it cannot fully verify itself.

Public and private checks answer different questions:

- public checks validate DNS, Cloudflare, Access behavior, tunnel routing, and
  application availability
- private origin checks validate the application independently of Cloudflare
- host checks validate node availability

Name these checks clearly so failures are interpretable.

Validation should cover:

- Compose and container state
- local dashboard response
- restored monitor definitions and active checks
- network reachability to public and private targets
- unauthorized Cloudflare access denied
- authorized dashboard access
- test notification delivery where configured

Source files:

- `compose/uptime-kuma/README.md`
- `compose/uptime-kuma/docker-compose.yml`

