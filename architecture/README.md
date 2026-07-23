# Personal Cloud Architecture

## Purpose

This document describes the architecture of the personal cloud: the machines involved, their responsibilities, how services are exposed, where state lives, and the principles guiding future evolution.

The platform is intentionally small. It is designed to provide useful self-hosted services while remaining understandable, secure, recoverable, and inexpensive to operate.

The repository should describe the system that is actually running. Future-state ideas should be clearly identified as such.

## Architecture goals

The platform is designed to:

* run useful personal services on owned infrastructure
* keep administrative access private
* avoid inbound router port forwarding
* separate configuration from persistent state
* make individual hosts replaceable
* detect failures independently of the primary compute node
* minimise operational complexity
* provide a practical environment for learning platform engineering
* evolve incrementally as real constraints emerge

## Current architecture

The personal cloud currently consists of two primary nodes.

```text
                         Internet
                            │
                  ┌─────────┴─────────┐
                  │                   │
           Cloudflare Access     Tailscale
                  │                   │
           Cloudflare Tunnel     Private network
                  │                   │
                  ▼                   ▼
        ┌─────────────────┐   ┌─────────────────┐
        │   Mac mini      │   │ ThinkPad X1    │
        │  Compute node   │   │ Management node│
        │                 │   │                 │
        │ Open WebUI      │   │ Uptime Kuma     │
        │ Calibre-Web     │   │ Administration  │
        │ Local LLMs      │   │ Troubleshooting │
        └────────┬────────┘   └────────┬────────┘
                 │                     │
                 └──── Local network ──┘
```

## Nodes

### Compute node

The primary compute node is a Mac mini.

Its responsibilities include:

* running user-facing application containers
* hosting local AI models
* providing the majority of application compute capacity
* storing service data that currently requires local high-performance access
* exposing selected web services through Cloudflare Tunnel

Current workloads include:

* Open WebUI
* local language models
* Calibre-Web

The compute node is designed to provide capacity rather than act as the management authority for the entire platform.

A failure of this node may make applications unavailable, but it should not remove the ability to observe, diagnose, or rebuild the platform.

### Management node

The management node is a Fedora-based ThinkPad X1 Carbon.

Its responsibilities include:

* running monitoring services
* providing an independent administration environment
* acting as a troubleshooting and recovery entry point
* maintaining private connectivity to other nodes
* supporting future automation and platform-management workloads

Current workloads include:

* Uptime Kuma
* Cloudflare Tunnel for the status service
* Tailscale
* container tooling

The management node is deliberately separate from the compute node.

This allows it to:

* detect failures of the compute node
* remain accessible during compute-node maintenance
* provide a stable control and recovery path
* avoid monitoring the primary host from the host being monitored

## Service placement

Services are placed according to their resource and operational requirements.

| Service           | Host        | Reason                                              |
| ----------------- | ----------- | --------------------------------------------------- |
| Open WebUI        | Mac mini    | Requires access to local AI models and compute      |
| Local LLM runtime | Mac mini    | Primary high-memory compute node                    |
| Calibre-Web       | Mac mini    | User-facing application with local library access   |
| Uptime Kuma       | ThinkPad X1 | Must remain independent of the primary compute node |

Service placement may change as dedicated storage or additional nodes are introduced.

The intended rule is:

> Place services according to failure boundaries and resource needs, not merely according to available capacity.

## Connectivity

The platform uses three main connectivity paths.

### Local network

The local network provides direct connectivity between trusted devices and platform nodes.

It is used for:

* host-to-host communication
* local administration
* storage access
* high-bandwidth traffic
* service access where no external route is required

The local network is not assumed to be fully trusted. Sensitive services should still require authentication.

### Tailscale

Tailscale provides private remote access to hosts and administrative services.

It is the preferred path for:

* SSH access
* host administration
* private dashboards
* direct service access
* recovery when a public endpoint is unavailable
* machine-to-machine communication across locations

Administrative services should generally be reachable only through Tailscale or the local network.

### Cloudflare Tunnel and Access

Cloudflare Tunnel provides browser-based access to selected services without exposing inbound ports on the home router.

Cloudflare Access provides identity-aware authentication in front of those services.

The request path is:

```text
Browser
  │
  ▼
Cloudflare Access
  │
  ▼
Cloudflare edge
  │
  ▼
Outbound Cloudflare Tunnel
  │
  ▼
Local service
```

This model provides:

* no inbound router port forwarding
* centralised access policy
* TLS termination
* reduced direct exposure of the home network
* browser access without requiring every client to join the tailnet

Cloudflare exposure should be limited to services that benefit from browser-based access.

## Trust boundaries

The architecture contains several distinct trust boundaries.

```text
Public Internet
    │
    ▼
Cloudflare identity and edge controls
    │
    ▼
Cloudflare Tunnel
    │
    ▼
Application container
    │
    ▼
Host operating system
    │
    ▼
Persistent local storage
```

Separate trust boundaries also exist between:

* the local network and platform nodes
* Tailscale clients and hosts
* containers and their host
* services and their backing data
* configuration repositories and runtime secrets

A successful login to one layer should not imply unrestricted trust in every other layer.

## Service exposure model

Services fall into three categories.

### Publicly reachable through identity controls

These services are reachable from the internet but protected by Cloudflare Access.

Examples:

* Open WebUI
* Calibre-Web
* Uptime Kuma status interface

They should not be directly exposed through router port forwarding.

### Private network only

These services are available only through Tailscale or the local network.

Examples may include:

* SSH
* administrative dashboards
* internal metrics
* container management endpoints
* databases
* host-level services

This is the default category for administrative interfaces.

### Localhost only

Some components should only be reachable from the host on which they run.

Examples may include:

* databases used by a single service
* local model APIs
* container control sockets
* internal helper services

Services should be exposed at the narrowest practical scope.

## Container runtime

Applications are currently deployed using Docker Compose-compatible tooling.

The runtime differs by host:

* OrbStack on the Mac mini
* Docker on the Fedora management node

Docker Compose is used because it provides:

* low operational overhead
* readable service definitions
* straightforward local deployment
* explicit volume and network configuration
* an appropriate abstraction for the current platform size

Kubernetes is not currently required.

It may become justified if the platform develops needs such as:

* multi-node scheduling
* automated failover
* standardised workload identity
* stronger reconciliation
* more complex deployment automation
* many interdependent services

Adopting an orchestrator is not itself an architectural goal.

## Configuration and repository boundaries

The Git repository is the source of truth for:

* Docker Compose definitions
* architecture documentation
* Architecture Decision Records
* service documentation
* operational runbooks
* example configuration
* reusable scripts

It is not the source of truth for:

* application databases
* uploaded files
* generated logs
* credentials
* OAuth secrets
* local environment files
* container runtime state
* large model files

The main boundary is:

```text
Git repository
    └── Defines how the system should run

Host configuration
    └── Supplies host-specific settings and secrets

Persistent storage
    └── Contains data produced or consumed by services

Container runtime
    └── Executes replaceable application instances
```

## Persistent storage

Persistent service data should live outside the repository.

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

Compose definitions should mount these directories explicitly.

This separation makes it possible to:

* replace containers without losing data
* review configuration without exposing runtime state
* back up data independently from source code
* rebuild a host from repository configuration
* apply different retention policies to different data

Some current service directories may still contain runtime files. These should be migrated out of the repository.

## Secrets

Secrets are currently managed outside Git and passed to services through local environment files or mounted files.

Secrets include:

* passwords
* API keys
* access tokens
* OAuth client secrets
* Cloudflare Tunnel credentials
* private certificates
* application secret keys

The current model prioritises simplicity while preserving the core rule that secrets must not be committed.

A future dedicated secret-management layer may use OpenBao or HashiCorp Vault when the operational value outweighs the added complexity.

## DNS and domains

Services use subdomains of `damienmurphy.net`.

Examples include:

* `ai.damienmurphy.net`
* `books.damienmurphy.net`
* `status.damienmurphy.net`

DNS is managed through Cloudflare.

Each externally reachable service typically has:

* a DNS record associated with a Cloudflare Tunnel
* a tunnel route to a local service
* a Cloudflare Access policy
* a local service listener that is not directly exposed to the internet

Domain and tunnel configuration should be documented per service.

## Observability

The current observability model is intentionally lightweight.

Uptime Kuma provides:

* external endpoint monitoring
* service availability checks
* host reachability checks
* a central operational status view

Additional signals currently come from:

* container status
* application logs
* host operating-system logs
* Cloudflare Tunnel status
* direct health checks

Future observability capabilities may include:

* system metrics
* container resource usage
* centralised logs
* backup success monitoring
* storage capacity alerts
* certificate-expiry checks
* notification routing through ntfy
* synthetic user journeys

New observability components should be introduced to answer specific operational questions.

## Failure model

The architecture assumes that individual components will fail.

Expected failure modes include:

* a container stopping
* an application upgrade failing
* a host becoming unavailable
* a disk filling
* a tunnel disconnecting
* a credential expiring
* the home network losing internet access
* a machine requiring replacement
* configuration drifting from the repository
* persistent data becoming corrupted

The system should progressively support recovery from these failures without relying on undocumented knowledge.

## Recovery model

The intended recovery hierarchy is:

1. Restart the affected container.
2. Redeploy the service from its Compose definition.
3. Restore service configuration and data from backup.
4. Rebuild the host and redeploy all assigned services.
5. Move the service to another capable host.

The repository provides the configuration and operational knowledge required for steps two through five.

A complete recovery capability also requires:

* verified backups
* documented secret recovery
* host bootstrap instructions
* tested restore procedures

## Security model

The platform follows several core security principles.

### No direct inbound exposure

Normal operation should not require inbound router port forwarding.

### Private administration

SSH and administrative interfaces should be limited to:

* Tailscale
* trusted local devices
* localhost

### Identity before network location

Browser-accessible services should rely on explicit identity controls rather than assuming that network reachability implies trust.

### Least privilege

Services should receive only the:

* ports
* filesystem paths
* credentials
* network access
* host capabilities

that they require.

### Replaceable workloads

Containers should be treated as disposable. Important data must live in explicit persistent storage.

### Defence through recoverability

Backups, reproducible configuration, and documented recovery procedures are part of the security model.

## Architecture evolution

The platform is expected to evolve incrementally.

Likely future additions include:

* dedicated storage or NAS capacity
* automated backups
* backup restore testing
* centralised notifications
* Home Assistant
* Paperless-ngx
* Memos
* Jellyfin
* centralised secrets management
* host bootstrap automation
* shared Compose patterns
* dependency and image update automation
* stronger metrics and logging
* additional compute or storage nodes

Each addition should answer at least one of these questions:

* What problem does it solve?
* Which failure mode does it reduce?
* Which manual process does it remove?
* Which capability does it make reusable?
* What new operational burden does it introduce?

## Target architecture direction

The likely long-term direction is a small multi-node personal platform with clearer role separation.

```text
                       Users and devices
                              │
              ┌───────────────┴───────────────┐
              │                               │
       Cloudflare Access                 Tailscale
              │                               │
              └───────────────┬───────────────┘
                              │
                    Private service network
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
   Management node       Compute node       Storage node
          │                   │                   │
   Monitoring            Applications          Backups
   Automation            Local AI              Media
   Notifications         Processing             Documents
   Recovery tooling      User services          Shared data
```

This is a direction rather than a commitment.

Additional nodes should be introduced only where they create a useful failure boundary, capacity boundary, or ownership boundary.

## Architectural decision records

Significant decisions should be recorded in `../adrs/`.

An ADR is appropriate when a decision:

* is expensive to reverse
* affects multiple services
* creates a new platform-wide pattern
* introduces meaningful operational complexity
* changes a trust boundary
* changes how data is stored or recovered
* explains why an apparently obvious alternative was rejected

Examples include:

* choosing Cloudflare Tunnel and Access
* using Tailscale for administrative access
* separating monitoring from compute
* keeping Docker Compose as the current deployment model
* introducing a dedicated storage node
* selecting a secret-management system

## Current limitations

The current architecture has several known gaps:

* backups are not yet fully automated or restore-tested
* secrets are not centrally managed
* host provisioning is not fully reproducible
* service deployment remains primarily manual
* monitoring is focused on availability rather than full system health
* some runtime state still needs to be moved outside the repository
* the Mac mini remains a significant single point of failure
* storage architecture is still evolving

These limitations are accepted for the current scale but should remain visible.

## Architecture principles

The platform should continue to follow these principles:

1. Keep the system understandable.
2. Prefer explicit configuration over hidden state.
3. Separate configuration, secrets, and persistent data.
4. Keep administrative access private.
5. Expose services only when there is a clear user need.
6. Place monitoring outside the systems it monitors.
7. Make hosts rebuildable and workloads movable.
8. Document decisions and recovery paths.
9. Add complexity only when it removes a larger constraint.
10. Build the smallest platform that provides the required capability.

