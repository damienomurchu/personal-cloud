# ADR 0003: Host Port Reservation Schema

* **Status:** Accepted
* **Date:** 2026-08-17

## Context

Personal-cloud services expose host ports for local access, Cloudflare Tunnel, reverse proxies, monitoring, and administration.

Using application-default ports directly on hosts makes assignments inconsistent, increases collision risk, and couples deployment configuration to application internals.

A stable host-port allocation scheme is required.

## Decision

Reserve host ports `10000-19999` for personal-cloud services.

| Range         | Purpose                         |
| ------------- | ------------------------------- |
| `10000-10999` | Management and observability    |
| `11000-11999` | Network and infrastructure      |
| `12000-12999` | Development and source control  |
| `13000-13999` | Knowledge and collaboration     |
| `14000-14999` | Media and content               |
| `15000-15999` | AI and ML                       |
| `16000-16999` | Data and storage administration |
| `17000-17999` | Automation and messaging        |
| `18000-18999` | Experimental services           |
| `19000-19999` | Reserved                        |

Allocate ports in increments of `10` where practical.

Example:

```text
10010  uptime-kuma

12010  forgejo
12020  code-server

13010  docmost
13020  memos

14010  calibre-web

15010  open-webui

17010  ntfy
```

Adjacent ports may be used for multiple interfaces of the same service:

```text
12010  forgejo-http
12011  forgejo-ssh
```

## Rules

1. A host port belongs to the service, not the node.
2. Services retain their port when moved between nodes.
3. Container ports remain application-defined.
4. Only services requiring host exposure receive a host port.
5. New allocations must be recorded in `docs/ports.yaml`.
6. Experimental services use `18000-18999`.
7. Kubernetes NodePorts remain outside this scheme.

Example Docker mapping:

```yaml
ports:
  - "127.0.0.1:10010:3001"
```

Here `10010` is the reserved host port and `3001` is the application's container port.

## Registry

`docs/ports.yaml` is the source of truth.

```yaml
ranges:
  management:     10000-10999
  infrastructure: 11000-11999
  development:    12000-12999
  knowledge:      13000-13999
  media:          14000-14999
  ai:             15000-15999
  data:           16000-16999
  automation:     17000-17999
  experimental:   18000-18999
  reserved:       19000-19999

services:
  uptime-kuma:
    port: 10010
    node: mgt-1

  forgejo:
    port: 12010

  code-server:
    port: 12020

  docmost:
    port: 13010

  memos:
    port: 13020

  calibre-web:
    port: 14010

  open-webui:
    port: 15010

  ntfy:
    port: 17010
```

## Kubernetes

Manual allocations remain below `30000` to avoid Kubernetes' default NodePort range:

```text
30000-32767
```

## Consequences

* Predictable host ports.
* Reduced collision risk.
* Stable service identities across nodes.
* Cleaner Docker Compose and tunnel configuration.
* Requires maintaining `docs/ports.yaml`.
