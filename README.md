# Personal Cloud

Infrastructure, architecture, and operational documentation for my personal cloud.

This repository contains the configuration and documentation used to operate a small collection of self-hosted services across personally owned infrastructure.

The platform is intended to remain:

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
├── README.md
├── HANDBOOK.md
├── adrs/
│   └── 0001-service-exposure-model.md
├── architecture/
│   └── README.md
├── compose/
│   ├── calibre-web/
│   ├── openweb-ui/
│   └── uptime-kuma/
└── docs/
    └── service-onboarding.md
```

### `HANDBOOK.md`

Defines the platform-wide engineering principles, standards, service lifecycle, documentation model, and operational expectations.

See the [Platform Handbook](HANDBOOK.md).

### `adrs/`

Architecture Decision Records capturing significant platform decisions and their trade-offs.

Current decisions:

* [ADR 0001: Service Exposure Model](adrs/0001-service-exposure-model.md)

### `architecture/`

Current-state and target-state platform architecture.

* [Personal Cloud Architecture](architecture/README.md)

### `compose/`

Docker Compose definitions and service-specific operational documentation.

Each service directory should contain:

```text
compose/<service>/
├── docker-compose.yml
└── README.md
```

Host-specific environment files, secrets, and persistent runtime data must not be committed.

### `docs/`

Platform-wide operational documentation and runbooks.

Current documentation:

* [Service Onboarding](docs/service-onboarding.md)

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

New services should follow the workflow documented in:

* [Service Onboarding](docs/service-onboarding.md)
* [Platform Handbook](HANDBOOK.md#service-lifecycle)

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

## Known gaps

The platform currently has several accepted limitations:

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

Significant architectural changes should be introduced through ADRs rather than undocumented implementation drift.

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

