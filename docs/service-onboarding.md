# Service Onboarding

## Purpose

This document defines the standard process for adding a service to the personal cloud.

It covers the practical work required to move a service from an experiment to an operated part of the platform:

* choosing a host
* defining the service with Docker Compose
* separating configuration, secrets, and persistent data
* selecting an exposure model
* configuring access
* adding monitoring
* documenting operation and recovery

The objective is not merely to start a container. It is to make the service understandable, secure, observable, and recoverable.

The architectural rules behind network exposure are recorded in [`../adrs/0001-service-exposure-model.md`](../adrs/0001-service-exposure-model.md).

## Definition of onboarded

A service is considered onboarded when:

* its purpose and owner are clear
* its host placement is intentional
* its Compose configuration is version controlled
* secrets are excluded from Git
* persistent data is stored outside the repository
* its network exposure is no broader than necessary
* authentication is enabled where appropriate
* it has a documented health check
* it is monitored where useful
* its update and recovery procedures are documented
* the deployed configuration matches the repository

A running container alone does not satisfy this definition.

## Repository layout

Each service should have its own directory under `compose/`.

```text
compose/<service>/
├── docker-compose.yml
├── .env.example
└── README.md
```

Additional committed files may be included where they are static configuration rather than secrets or generated state.

Example:

```text
compose/example/
├── config/
│   └── application.yml
├── docker-compose.yml
├── .env.example
└── README.md
```

The following should normally remain outside Git:

* databases
* uploaded files
* generated configuration
* logs
* credentials
* API tokens
* OAuth client secrets
* private keys
* application runtime data

## 1. Define the service

Before deployment, record:

* what problem the service solves
* who will use it
* whether it is experimental or intended to be durable
* the consequences of it becoming unavailable
* whether it stores important state
* whether an existing service already provides the capability

A service should solve a real need rather than merely provide another system to operate.

## 2. Choose the host

Place the service according to its resource needs and failure characteristics.

### Compute node

Prefer the Mac mini when the service:

* needs significant CPU or memory
* uses local AI models
* requires access to data already stored on the Mac mini
* is primarily a user-facing application
* benefits from OrbStack or macOS-hosted tooling

### Management node

Prefer the Fedora management node when the service:

* provides monitoring or administration
* should remain available when the compute node fails
* supports recovery or troubleshooting
* has low compute requirements
* forms part of the platform control plane

Do not place monitoring exclusively on the system being monitored.

Record the chosen host and rationale in the service README.

## 3. Classify the service state

Determine what data the service produces or depends on.

Classify each path as one of:

| Class                            | Examples                                | Location                      |
| -------------------------------- | --------------------------------------- | ----------------------------- |
| Version-controlled configuration | Compose files, static config, templates | Git repository                |
| Host-specific configuration      | Ports, domains, paths, feature flags    | Local `.env`                  |
| Secrets                          | Passwords, tokens, private keys         | Outside Git                   |
| Persistent state                 | Databases, uploads, libraries           | External data directory       |
| Disposable runtime state         | Caches, temporary files                 | Container or temporary volume |

Persistent data should use an explicit host path or named volume whose location and backup requirements are understood.

A preferred host layout is:

```text
~/personal-cloud-data/<service>/
├── config/
├── data/
└── backups/
```

Not every service needs every directory.

## 4. Create the Compose definition

Create:

```text
compose/<service>/docker-compose.yml
```

The Compose definition should make important behaviour explicit, including:

* container image and version
* restart policy
* mounted volumes
* environment variables
* exposed ports
* service networks
* health checks where supported
* dependency relationships

Example:

```yaml
services:
  example:
    image: example/example:1.2.3
    restart: unless-stopped

    environment:
      SERVICE_URL: ${SERVICE_URL}
      SERVICE_SECRET: ${SERVICE_SECRET}

    volumes:
      - ${PERSONAL_CLOUD_DATA}/example:/app/data

    ports:
      - "127.0.0.1:${SERVICE_PORT}:8080"
```

Prefer binding published ports to `127.0.0.1` when access is provided through a tunnel running on the same host.

Avoid privileged containers, host networking, broad filesystem mounts, and access to the Docker socket unless they are genuinely required.

## 5. Create example configuration

Add a `.env.example` containing required variable names and safe placeholder values.

```dotenv
SERVICE_URL=https://service.example.net
SERVICE_PORT=8080
SERVICE_SECRET=replace-me
PERSONAL_CLOUD_DATA=/path/to/personal-cloud-data
```

Do not include real credentials.

The actual `.env` file should:

* remain local to the host
* be excluded by `.gitignore`
* have restrictive filesystem permissions
* be covered by the secret recovery process where necessary

## 6. Choose the exposure class

Every service must be assigned one exposure class.

### Localhost only

Use when the service is consumed only by another process on the same host.

Examples:

* internal databases
* local model APIs
* helper services
* private application backends

Typical binding:

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

### Private network

Use when the service is needed remotely but only from trusted devices or by administrators.

Access should be through:

* Tailscale
* the trusted local network

Examples:

* SSH
* administrative dashboards
* internal metrics
* container management
* host-level services

This is the default for administration.

### Browser-accessible through Cloudflare

Use when a service benefits from browser access across devices without requiring each client to join the tailnet.

The normal path is:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> Local service
```

Examples:

* Open WebUI
* Calibre-Web
* Uptime Kuma

Internet reachability does not mean anonymous public access. Cloudflare Access should protect services unless anonymous access is an explicit requirement.

The full architectural rationale is documented in [`../adrs/0001-service-exposure-model.md`](../adrs/0001-service-exposure-model.md).

## 7. Configure authentication

Use multiple layers where appropriate.

### Application authentication

Enable the application's own authentication when it provides meaningful protection.

Do not automatically disable application authentication merely because Cloudflare Access is present.

### Cloudflare Access

For browser-exposed services:

* create or reuse the appropriate Access application
* restrict access to approved identities
* avoid broad bypass rules
* test both authorised and unauthorised access
* document any machine-to-machine exceptions

### Tailscale

For private services:

* verify the service is reachable over the tailnet
* avoid exposing administrative ports on all interfaces without need
* use Tailscale ACLs or grants as the network grows
* keep device enrolment and identity under control

## 8. Configure Cloudflare Tunnel

For a browser-exposed service:

1. Confirm the service is reachable locally.
2. Add the hostname to the appropriate Cloudflare Tunnel.
3. Route the hostname to the local service address.
4. Create or validate the DNS entry.
5. Apply a Cloudflare Access policy.
6. Verify TLS and redirect behaviour.
7. Confirm the origin is not independently reachable from the internet.

Example route:

```text
ai.damienmurphy.net
  -> Cloudflare Tunnel
  -> http://127.0.0.1:3000
```

Prefer one tunnel per host unless a stronger isolation requirement justifies one tunnel per service.

Tunnel credentials are secrets and must not be committed.

## 9. Add health validation

Define how the service will be checked after deployment.

At minimum, validate:

```bash
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Where possible, add an application-aware health check.

Example:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 30s
```

A container being in the `running` state does not necessarily mean the application is working.

## 10. Add monitoring

Add the service to Uptime Kuma when an outage would be useful to detect.

Choose the monitor type according to the service:

* HTTPS endpoint for browser services
* TCP port for simple network services
* ping or Tailscale address for host reachability
* keyword or status-code validation for application-aware checks

Monitor from outside the service's own host where practical.

For Cloudflare-protected endpoints, decide whether the monitor should validate:

* the public access path
* the local origin
* or both

These detect different failure modes.

## 11. Define backup requirements

For each persistent path, determine:

* whether the data can be recreated
* whether the data must be backed up
* how frequently it changes
* how much data loss is acceptable
* whether backups require encryption
* where backups will be stored
* how restoration will be tested

A service README should explicitly state one of:

* no backup required
* configuration backup only
* database backup required
* full data backup required

Do not assume that a mounted volume is a backup.

## 12. Write the service README

Each service README should include:

```markdown
# Service Name

## Purpose

## Host

## Exposure

## Dependencies

## Configuration

## Persistent data

## Secrets

## Deployment

## Validation

## Monitoring

## Backup and restore

## Updating

## Troubleshooting

## Removal
```

Keep the document specific to the service.

Platform-wide architectural reasoning should link to the relevant ADR rather than being repeated.

## 13. Deploy the service

From the service directory:

```bash
docker compose config
docker compose pull
docker compose up -d
```

Validate:

```bash
docker compose ps
docker compose logs --tail=100
```

Then test each intended access path:

* localhost
* local network
* Tailscale
* Cloudflare hostname

Only test the paths selected for the service.

## 14. Verify security boundaries

Before considering the service onboarded, confirm:

* no secrets are staged in Git
* persistent runtime data is outside the repository
* only required ports are published
* administrative interfaces are private
* Cloudflare Access protects browser-exposed services
* the service is not reachable through unintended paths
* default credentials have been changed
* unnecessary container privileges are absent
* the deployed image version is intentional

Useful checks include:

```bash
git status
git diff --cached
docker compose config
docker compose ps
```

Inspect host listeners:

```bash
ss -lntup
```

On macOS, use an equivalent listener inspection command where required.

## 15. Record completion

Before merging or committing the onboarding change, verify:

* [ ] Service purpose is documented
* [ ] Host placement is documented
* [ ] Compose file is committed
* [ ] `.env.example` is committed
* [ ] Real `.env` is ignored
* [ ] Persistent state is outside Git
* [ ] Exposure class is documented
* [ ] Authentication is configured
* [ ] Health validation succeeds
* [ ] Monitoring is configured where useful
* [ ] Backup requirements are documented
* [ ] Update procedure is documented
* [ ] Recovery procedure is documented
* [ ] No credentials or runtime data are staged

## Updating an onboarded service

Changes should follow the same boundaries established during onboarding.

Before an update:

1. Review upstream release notes.
2. Check for breaking changes or migrations.
3. Confirm backup status for stateful services.
4. Update the image version.
5. Validate the rendered Compose configuration.
6. Deploy.
7. Verify health and access paths.
8. Update documentation where behaviour changed.

For significant platform-wide changes, create or update an ADR.

## Removing a service

Removal should be intentional and recoverable.

1. Confirm whether persistent data must be retained.
2. Export or back up required data.
3. Stop and remove the containers.
4. Remove Cloudflare Tunnel routes.
5. Remove Cloudflare Access applications or policies.
6. Remove DNS records.
7. Remove Uptime Kuma monitors.
8. Remove unused secrets.
9. Remove or archive the Compose definition.
10. Document retained data and its deletion date, where relevant.

Do not delete persistent data merely because the container has been removed.

## When an ADR is required

A service-specific onboarding change normally does not need an ADR.

Create an ADR when onboarding introduces a broader decision such as:

* a new exposure pattern
* a new identity provider
* a new secret-management mechanism
* a new host role
* direct public access
* router port forwarding
* a shared database
* a new orchestration platform
* a change to platform trust boundaries

Routine implementation details belong in the service README or this runbook. Durable architectural decisions belong in `adrs/`.

