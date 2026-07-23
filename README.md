# Personal Cloud

Infrastructure, architecture, and operational documentation for my personal cloud.

This repository contains the configuration and documentation used to run a small collection of self-hosted services across personally owned infrastructure.

The platform is intended to be:

* understandable
* secure by default
* observable
* recoverable
* inexpensive to operate
* easy to extend

The goal is not simply to run containers. It is to build a small, durable personal platform whose design, operation, and recovery do not depend on memory.

## Current services

| Service                             | Purpose                                     | Host        | Access                       |
| ----------------------------------- | ------------------------------------------- | ----------- | ---------------------------- |
| [Open WebUI](compose/openweb-ui/)   | Private interface for local language models | Mac mini    | Cloudflare Access and Tunnel |
| [Calibre-Web](compose/calibre-web/) | Personal ebook library                      | Mac mini    | Cloudflare Access and Tunnel |
| [Uptime Kuma](compose/uptime-kuma/) | Service and host availability monitoring    | ThinkPad X1 | Cloudflare Access and Tunnel |

Administrative access to platform hosts is provided through Tailscale or the local network.

## Architecture

The platform currently consists of two primary nodes:

* a Mac mini acting as the main compute node
* a Fedora-based ThinkPad X1 acting as the management and monitoring node

Selected user-facing services are exposed through Cloudflare Tunnel and protected by Cloudflare Access.

Administrative services remain private through Tailscale or the trusted local network.

See [architecture/README.md](architecture/README.md) for the full architecture, trust boundaries, service placement, storage model, and future direction.

## Repository structure

```text
.
├── adrs/
│   └── 0001-service-exposure-model.md
├── architecture/
│   └── README.md
├── compose/
│   ├── calibre-web/
│   ├── openweb-ui/
│   └── uptime-kuma/
├── docs/
│   └── service-onboarding.md
└── README.md
```

### `adrs/`

Architecture Decision Records capturing significant platform decisions and their trade-offs.

Current decisions:

* [ADR 0001: Service Exposure Model](adrs/0001-service-exposure-model.md)

### `architecture/`

Current-state and target-state platform architecture.

* [Personal Cloud Architecture](architecture/README.md)

### `compose/`

Docker Compose definitions and service-specific operational documentation.

Each service directory contains:

```text
compose/<service>/
├── docker-compose.yml
└── README.md
```

Host-specific environment files, secrets, and persistent runtime data should not be committed.

### `docs/`

Platform-wide operational documentation and runbooks.

* [Service Onboarding](docs/service-onboarding.md)

## Design principles

### Keep the platform understandable

Prefer explicit, boring, and recoverable systems over unnecessary complexity.

### Use the narrowest practical exposure

Services should be:

1. localhost-only where possible
2. private through Tailscale or the local network where remote access is required
3. exposed through Cloudflare Access and Tunnel only where browser accessibility provides real value

Direct inbound router port forwarding is not part of the normal architecture.

### Separate configuration from state

This repository contains the instructions required to run the platform.

It should not contain:

* application databases
* logs
* uploaded files
* OAuth credentials
* API tokens
* private keys
* generated application state

Persistent data should live outside the repository and be mounted explicitly by Compose.

### Treat documentation as part of the system

A service is not complete when its container starts.

It should also have:

* documented host placement
* known dependencies
* an intentional exposure model
* monitoring
* backup requirements
* update instructions
* recovery procedures

### Add complexity only when it removes a larger constraint

Kubernetes, centralised secrets management, distributed storage, and deeper observability may become useful later.

They are not goals by themselves.

## Deploying a service

Deployment is performed from the relevant service directory.

```bash
cd compose/<service>

docker compose config
docker compose pull
docker compose up -d
```

Validate the deployment:

```bash
docker compose ps
docker compose logs --tail=100
```

See the service README for:

* required configuration
* persistent-data paths
* secrets
* access details
* validation
* monitoring
* backup and restore
* troubleshooting

## Adding a service

New services should follow the onboarding workflow documented in:

[docs/service-onboarding.md](docs/service-onboarding.md)

At minimum, a service should have:

* a clear purpose
* intentional host placement
* a Compose definition
* a service README
* secrets excluded from Git
* persistent state outside the repository
* an assigned exposure class
* a validation method
* documented monitoring and backup requirements

## Security model

The platform assumes that:

* the local network is not inherently trusted
* internet-facing services will be scanned
* containers may contain vulnerabilities
* credentials will require rotation
* machines and disks may fail
* configuration mistakes are likely

The main controls are:

* Cloudflare Access for browser-facing identity
* Cloudflare Tunnel for outbound-only ingress
* Tailscale for private administration
* narrow host-port bindings
* application-level authentication where useful
* explicit persistent storage
* version-controlled configuration
* documented recovery paths

See [ADR 0001](adrs/0001-service-exposure-model.md) for the full exposure decision.

## Persistent data

A target host layout is:

```text
~/personal-cloud-data/
├── calibre-web/
│   ├── config/
│   └── library/
├── openweb-ui/
│   └── data/
└── uptime-kuma/
    └── data/
```

Compose definitions should reference these paths through environment variables or explicit mounts.

The repository itself is not a backup of the platform.

## Known gaps

The current platform still has several accepted limitations:

* backups are not yet fully automated or restore-tested
* secret management is decentralised
* host provisioning is not fully reproducible
* deployments are primarily manual
* monitoring focuses mainly on availability
* storage architecture is still evolving
* the Mac mini remains a significant application failure domain

These are visible constraints rather than hidden assumptions.

## Roadmap

Likely future work includes:

* migrating remaining runtime state out of the repository
* automated backups and restore testing
* reusable Compose patterns
* image and dependency update automation
* host bootstrap automation
* centralised notifications
* storage-capacity monitoring
* dedicated storage infrastructure
* Paperless-ngx
* Home Assistant
* Jellyfin
* ntfy
* Memos
* centralised secret management where justified

## Repository hygiene

Before committing:

```bash
git status
git diff --cached
```

Verify that no secrets or runtime data are staged.

Files such as the following should normally be ignored:

```gitignore
.env
.env.*
!.env.example

*.db
*.sqlite
*.sqlite3
*.log

client_secrets.json
*.pem
*.key

data/
backups/
```

Service-specific ignore rules should preserve useful static configuration while excluding generated state.

## Why this repository exists

Self-hosted systems tend to decay into undocumented containers, hidden dependencies, forgotten credentials, and machines that cannot be safely rebuilt.

This repository exists to prevent that outcome.

It is the durable record of:

* how the platform is structured
* why key decisions were made
* how services are deployed
* where state lives
* how failures are detected
* how the system can be recovered

The aim is not to host the most services.

It is to operate a personal platform that remains understandable as it grows.

