# Service Context

This file summarizes the current service inventory. Each service README and
Compose definition remain authoritative for service-specific work.

## Inventory

| Service | Status | Host | Browser hostname | Origin | Durable state |
| --- | --- | --- | --- | --- | --- |
| Open WebUI | Current | Mac mini | `ai.damienmurphy.net` | Configurable localhost host port to container port `8080` | Open WebUI data |
| Calibre-Web | Current | Mac mini | `books.damienmurphy.net` | Configurable localhost host port to container port `8083` | Application configuration and Calibre library |
| Forgejo | Prepared, not deployed | Mac mini | `git.damienmurphy.net` | Localhost HTTP port `3002`; private Tailscale SSH port `2222` | Repositories, SQLite database, configuration, keys, and application data |
| Uptime Kuma | Current | ThinkPad X1 | `status.damienmurphy.net` | `127.0.0.1:3001` | Uptime Kuma application data |

The three current browser hostnames use Cloudflare Tunnel and Cloudflare Access.
Forgejo is configured to follow the same browser exposure model after it is
deployed.
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

## Forgejo

Purpose: private Git repository hosting and code collaboration.

Forgejo is prepared for the Mac mini with SQLite and a single external `/data`
mount. Do not describe it as running until deployment and validation are
complete.
Durable state includes repositories, the database, application configuration,
generated secrets, SSH keys, attachments, releases, and other enabled
application data.

Access is intentionally split:

- browser access uses `git.damienmurphy.net` through Cloudflare Access and a
  localhost-bound HTTP origin
- Git-over-SSH uses the Mac mini's private Tailscale address and advertised
  MagicDNS hostname
- Git-over-HTTPS, automated API clients, inbound webhooks, Actions runners, and
  package clients require separate access and security decisions

Validation must include:

- container and application health
- authorized and unauthorized browser access
- private repository creation
- authenticated clone, push, and fetch over Tailscale SSH
- persistence across restart
- a restorable, encrypted, off-host backup

Do not initialize new state if repositories appear missing until the `/data`
mount has been verified.

Source files:

- `compose/forgejo/README.md`
- `compose/forgejo/docker-compose.yml`

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
