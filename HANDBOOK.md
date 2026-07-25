# Platform Handbook

This handbook defines the engineering principles, standards, lifecycle rules, and operating expectations for the Personal Cloud platform.

It answers the question:

> How should this platform be designed, changed, operated, and maintained?

The root [README](README.md) describes the current platform and repository. Architecture documents explain how the platform is structured. ADRs record why significant decisions were made. Runbooks describe how operational procedures are performed.

This handbook defines the rules that tie those documents together.

## Purpose and scope

The Personal Cloud is a small self-hosted platform used for learning, experimentation, automation, and day-to-day personal services.

It is intentionally treated as an engineering platform rather than a collection of unrelated containers.

This handbook applies to:

* platform hosts
* self-hosted services
* deployment configuration
* persistent storage
* network exposure
* monitoring
* backup and recovery
* automation
* documentation
* operational maintenance

The handbook should change slowly. Current hosts, services, technologies, and roadmap items belong in the root README or architecture documentation.

## Engineering principles

### Keep the platform understandable

A future operator should be able to understand the platform without relying on memory, hidden state, or undocumented conventions.

Prefer explicit configuration, predictable directory structures, readable scripts, and straightforward operational procedures.

Complexity that cannot be clearly explained is not yet justified.

### Prefer boring technology

Use mature, well-understood technology unless a newer or more complex option removes a meaningful constraint.

The platform is not intended to maximise novelty.

Docker Compose, shell scripts, local filesystems, and simple monitoring are valid choices when they meet the requirement.

### Add complexity only when it removes a larger constraint

Kubernetes, GitOps, distributed storage, centralised secrets management, and deeper observability may become useful.

They are not goals by themselves.

A new platform capability should solve a concrete problem that is larger than the operational burden it introduces.

### Treat security as a system property

Security should be provided by the platform architecture rather than depending entirely on each application behaving correctly.

Controls should be layered and should include, where appropriate:

* narrow network exposure
* identity-aware access
* private administrative paths
* application authentication
* least-privilege credentials
* explicit secret handling
* version-controlled configuration
* recoverable infrastructure

### Separate configuration from state

This repository contains the instructions required to operate the platform.

It must not contain runtime state or sensitive material.

The repository should not contain:

* application databases
* uploaded files
* logs
* generated caches
* OAuth credentials
* API tokens
* private keys
* environment-specific secrets
* backup archives

Persistent data must live outside the repository and be mounted explicitly.

### Design for recoverability

A service is not durable merely because it is currently running.

Its configuration, state, dependencies, backup requirements, and recovery process should be understood.

The platform should assume that:

* machines will fail
* disks will fail
* credentials will expire
* containers will contain vulnerabilities
* upgrades will break behaviour
* configuration mistakes will occur
* external services may become unavailable

Recovery should not depend on reconstructing past decisions from memory.

### Make failure visible

Failures should be detectable before they become long-lived hidden conditions.

Services should expose enough information to determine whether they are:

* running
* reachable
* healthy
* out of storage
* failing repeatedly
* no longer backed up
* using expired credentials or certificates

Monitoring depth should be proportionate to service importance.

### Treat documentation as part of the system

Documentation is an operational control.

A service is not complete when its container starts. It is complete when another operator could understand how it is deployed, validated, maintained, and recovered.

Documentation should reduce future cognitive load and prevent repeated rediscovery.

### Evolve incrementally

Prefer small, reversible changes over large redesigns.

A working platform should not be destabilised merely to make it more theoretically elegant.

The platform should improve through repeated, bounded changes that leave it more understandable than before.

## Platform standards

## Service ownership

Each service must have an explicit purpose and a clear operational home.

A service definition should identify:

* why it exists
* where it runs
* what it depends on
* where its state lives
* how it is accessed
* how it is monitored
* how it is backed up
* how it is recovered
* how it is retired

Even where there is only one operator, ownership should remain explicit.

## Host placement

Service placement should be intentional.

Placement decisions should consider:

* hardware capabilities
* availability requirements
* storage dependencies
* network dependencies
* failure domains
* energy consumption
* operational separation
* recovery complexity

A service should not be placed on a host merely because that host is available.

Significant placement decisions should be documented in architecture documentation or an ADR.

## Configuration

Configuration should be declarative where practical.

Docker Compose files should:

* be readable
* use explicit service names
* declare persistent mounts clearly
* avoid unnecessary privileges
* define restart behaviour
* use health checks where useful
* avoid embedding secrets
* use environment variables consistently
* pin versions where stability requires it

Generated configuration should not be committed unless it is intentionally treated as source.

## Persistent state

Persistent state must be stored outside the repository.

State paths should be:

* explicit
* predictable
* documented
* included in backup scope
* portable enough to support recovery or migration

A target convention is:

```text
~/personal-cloud-data/
└── <service>/
    ├── config/
    ├── data/
    └── backups/
```

The exact layout may vary by host or service, but the chosen layout must be documented.

The repository itself is not a backup of the platform.

## Secrets

Secrets must not be committed to Git.

Secrets include:

* passwords
* API tokens
* client secrets
* private keys
* authentication cookies
* recovery codes
* encrypted credentials where the decryption key is colocated

Secrets should be provided through environment files, host-level secret stores, or a dedicated secrets-management system where justified.

Each service README should identify:

* which secrets are required
* where they are stored
* how they are created
* how they are rotated
* what depends on them

Secret-management complexity should remain proportionate to the size and risk of the platform.

## Networking and exposure

Services should use the narrowest practical exposure model.

The preferred order is:

1. local-only
2. private through Tailscale or the trusted local network
3. browser-accessible through Cloudflare Access and Tunnel
4. directly internet-accessible only where explicitly justified

Direct inbound router port forwarding is not part of the normal architecture.

Administrative interfaces should remain private unless there is a compelling operational reason otherwise.

Internet-exposed services should be assumed to be continuously scanned and probed.

The authoritative exposure decision is documented in:

* [ADR 0001: Service Exposure Model](adrs/0001-service-exposure-model.md)

## Authentication and authorisation

Identity should be enforced as close to the platform boundary as practical.

Cloudflare Access is the preferred identity layer for browser-facing services.

Application-level authentication should remain enabled where it adds meaningful defence in depth.

Tailscale is the preferred mechanism for private remote administration.

Shared credentials should be avoided where individual identity is available.

## Monitoring

Every operationally important service should have a documented monitoring approach.

At minimum, monitoring should answer:

* is the service reachable?
* is the host reachable?
* is the container running?
* is the service restarting repeatedly?
* is storage capacity becoming constrained?
* has a critical dependency failed?

Availability monitoring alone may be sufficient for low-risk services.

More important services may justify:

* application health checks
* resource monitoring
* storage alerts
* certificate monitoring
* backup monitoring
* dependency monitoring
* log inspection

Monitoring should produce actionable signals rather than noise.

## Backup and recovery

Every service containing non-reproducible state must have a backup strategy.

The strategy should identify:

* what data must be backed up
* where backups are stored
* how frequently backups occur
* how long backups are retained
* how encryption is handled
* how restoration is performed
* how restoration is verified

A backup is not considered trustworthy until it has been restored successfully.

Restore testing should be performed periodically for important services.

Configuration that can be recreated from Git does not need to be backed up as runtime state, but external dependencies and secrets required for recovery must still be documented.

## Updates and dependency management

Updates should be deliberate rather than automatic by default.

Before updating a service:

1. review the relevant release notes
2. identify breaking changes
3. verify backup availability
4. pull the intended image or package version
5. validate the resulting configuration
6. perform the update
7. verify service health
8. inspect logs
9. confirm external access
10. record any required documentation changes

Automatic updates may be introduced for low-risk components where rollback and observability are sufficient.

Production-like services should avoid unbounded use of floating image tags where unexpected changes would create unacceptable risk.

## Resource management

Services should not consume unlimited host resources by accident.

Where relevant, document or configure:

* CPU expectations
* memory requirements
* disk growth
* network usage
* retention policies
* cache behaviour
* concurrency limits

Capacity constraints should be visible before they cause service failure.

## Repository hygiene

Before committing:

```bash
git status
git diff
git diff --cached
```

Verify that no secrets, generated state, or large runtime files are staged.

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

Large or suspicious additions should be reviewed before they enter Git history.

## Service lifecycle

## Evaluation

Before adding a service, establish:

* the problem it solves
* the expected users
* whether an existing service already solves the problem
* whether self-hosting provides meaningful value
* the operational cost
* the security implications
* the state and backup requirements
* the likely long-term maintenance burden

A service should not be added merely because it is interesting to deploy.

## Onboarding

A new service should generally include:

* a dedicated directory under `compose/`
* a Docker Compose definition
* an `.env.example` where useful
* a service README
* explicit persistent-data mounts
* an assigned exposure model
* a validation procedure
* monitoring requirements
* backup and recovery requirements
* relevant repository ignore rules

The detailed workflow is documented in:

* [Service Onboarding](docs/service-onboarding.md)

## Deployment

Deployment should follow a repeatable procedure.

At minimum:

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Service-specific validation should then confirm:

* expected ports or routes
* application health
* authentication
* persistence
* external access where applicable
* monitoring coverage

## Operation

An operational service should have:

* known ownership
* known state location
* known dependencies
* known access paths
* known monitoring
* known backup coverage
* documented update instructions
* documented failure and recovery paths

Unmaintained services should not remain indefinitely merely because they continue running.

## Upgrade

Service upgrades should be documented where the process differs from the standard deployment procedure.

A runbook is required where an upgrade involves:

* schema migrations
* irreversible changes
* multi-step sequencing
* service downtime
* dependency coordination
* manual data transformation
* a non-obvious rollback process

## Migration

Moving a service between hosts should preserve:

* configuration
* state
* secrets
* access model
* monitoring
* backup coverage
* recovery documentation

The migration should include a rollback point and a validation procedure.

## Retirement

A service should be retired when it no longer provides enough value to justify its operational cost.

Retirement should include:

1. confirming that the service is no longer needed
2. exporting or preserving required data
3. removing external access
4. disabling monitoring
5. stopping and removing the deployment
6. archiving or deleting persistent state
7. removing obsolete secrets
8. updating architecture and repository documentation
9. recording significant replacement decisions where appropriate

Retirement is part of the service lifecycle, not an exceptional event.

## Documentation model

## Root README

The root README is the entry point to the platform.

It should contain:

* a concise platform description
* the current service inventory
* an architecture summary
* the repository structure
* basic deployment guidance
* known gaps
* the roadmap
* links to authoritative documentation

It describes the current platform rather than detailed policy.

## Platform handbook

The handbook defines stable engineering principles and operating standards.

It should not contain volatile service inventories, host specifications, or detailed runbook procedures.

## Service README

Each service README should explain:

* purpose
* host placement
* architecture
* configuration
* secrets
* persistent state
* exposure model
* deployment
* validation
* monitoring
* backup and restore
* updates
* troubleshooting
* known limitations

## Architecture documentation

Architecture documentation describes:

* system structure
* nodes
* networks
* trust boundaries
* service placement
* storage
* dependencies
* failure domains
* current and target state

Architecture documents explain how the platform fits together.

## Architecture Decision Record

An ADR records a significant decision and its consequences.

An ADR should be created when a change:

* establishes a platform-wide pattern
* introduces significant complexity
* changes a trust boundary
* changes the storage model
* changes the deployment model
* adds an important dependency
* constrains future choices
* replaces a previously documented decision

ADRs should capture:

* context
* decision
* alternatives considered
* consequences
* status

Accepted ADRs should not be rewritten to hide historical decisions. A later decision should supersede the earlier ADR.

## Runbook

A runbook documents an operational procedure.

A runbook is appropriate when a task:

* will be repeated
* is easy to perform incorrectly
* is performed infrequently enough to be forgotten
* is required during failure or time pressure
* has important validation or rollback steps

A runbook should include:

* purpose
* prerequisites
* procedure
* validation
* rollback
* failure handling
* related documentation

Runbooks should be executable, not merely descriptive.

## Definition of done

A new service is not considered complete until:

* its purpose is clear
* its host placement is intentional
* its configuration is version controlled
* its secrets are excluded from Git
* its persistent state is externalised
* its exposure model is deliberate
* its deployment is documented
* its validation is documented
* its monitoring requirements are documented
* its backup requirements are documented
* its recovery path is understood
* platform and architecture documentation are updated where necessary

Not every service requires the same operational depth, but every omission should be intentional.

## Operational expectations

## Validation

Changes should be validated at the lowest useful level.

Validation may include:

* configuration parsing
* container status
* health checks
* log inspection
* local connectivity
* remote connectivity
* authentication
* persistence across restart
* monitoring status
* backup status

A successful container start does not prove that the service is operational.

## Maintenance

Platform maintenance should include, where appropriate:

* reviewing container and dependency updates
* validating backups
* testing restores
* checking available disk capacity
* reviewing monitoring failures
* reviewing internet-exposed services
* rotating credentials
* removing unused services and resources
* updating documentation
* inspecting repository hygiene

Maintenance frequency should reflect the importance and rate of change of the platform.

## Failure handling

Failures should be treated as opportunities to improve the platform.

After a meaningful failure, consider:

* what failed
* why it was not prevented
* why it was not detected sooner
* whether recovery was straightforward
* what documentation was missing
* whether monitoring should change
* whether an ADR or runbook is required
* whether the architecture created unnecessary coupling

The objective is not to prevent every failure.

The objective is to make failures visible, bounded, and recoverable.

## Change management

Changes should remain small enough to understand and reverse.

Before a significant change:

* identify the problem
* define the expected outcome
* understand the affected services
* identify state and backup implications
* identify security implications
* define validation
* define rollback
* decide whether an ADR is required
* decide whether a runbook must be created or updated

A change that cannot be validated should not be considered complete.

## Guiding principle

The Personal Cloud is not a collection of containers.

It is a small engineering platform.

Every service, document, script, and architectural decision should make the platform easier to understand, safer to operate, simpler to recover, and more valuable over time.

