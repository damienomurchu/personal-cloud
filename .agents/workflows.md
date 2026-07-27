# Change and Validation Workflows

This file summarizes repository workflows for AI tools. `HANDBOOK.md` and
`docs/service-onboarding.md` are authoritative.

## Before a change

1. Read `README.md` and the relevant parts of `HANDBOOK.md`.
2. Read the affected architecture document and ADR.
3. Read the affected service README and Compose definition.
4. Inspect `git status` and preserve unrelated user changes.
5. Classify the request as service-specific or platform-wide.
6. Identify effects on state, secrets, security, backups, recovery, monitoring,
   documentation, validation, and rollback.
7. Decide whether an ADR or runbook is required.
8. Distinguish current repository configuration from documented target state.

## Standard repository checks

```bash
git status
git diff
git diff --cached
```

Before a commit, verify that no secrets, generated state, databases, logs,
backups, or large runtime files are staged.

## Compose validation

The standard command vocabulary is:

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Apply these constraints:

- `docker compose pull` and `docker compose up -d` change runtime state. Run them
  only when deployment is requested and the relevant host is in scope.
- `docker compose config` may interpolate secrets. Do not print its complete
  output in responses or durable logs. Report success or sanitized errors.
- Running state is not application health.
- Match validation depth to the risk and behavior changed.
- Test only the exposure paths assigned to the service.
- Test both authorized and unauthorized access for identity-protected services.
- For stateful changes, validate persistence and the restore path where
  proportionate.

## Service onboarding

A service is onboarded only when it has:

- a clear purpose, users, and owner
- intentional host placement
- `compose/<service>/docker-compose.yml`
- a safe `.env.example`
- secrets excluded from Git
- explicitly mounted persistent state outside the repository
- an assigned exposure class
- appropriate application and platform authentication
- application-aware health validation
- a documented monitoring decision
- documented backup and restore requirements
- documented update, troubleshooting, and removal procedures
- a service README
- relevant platform inventory and architecture updates
- a deployed configuration that matches the repository

Use the service README structure defined in `docs/service-onboarding.md`.

## Service update

1. Review upstream release notes.
2. Identify breaking changes, migrations, and compatibility constraints.
3. Confirm backup availability for non-reproducible state.
4. Record the current version and rollback point.
5. Change the image version or configuration deliberately.
6. Validate the Compose definition without exposing interpolated secrets.
7. Deploy only when explicitly requested.
8. Check container state and sanitized logs.
9. Test local, private, and public access paths as applicable.
10. Test actual application behavior, not just an HTTP response.
11. Update affected documentation.
12. Confirm rollback remains possible.

## Service retirement

Retirement must explicitly handle:

1. Required data export, retention, or secure deletion
2. External Cloudflare routes and DNS
3. Cloudflare Access applications or policies
4. Monitoring and alerts
5. Obsolete secrets and credentials
6. Container deployment
7. Persistent state
8. Repository inventory and architecture documentation

Do not delete persistent data merely because a container was removed.

## Documentation placement

| Information | Destination |
| --- | --- |
| Current inventory, concise architecture summary, known gaps, roadmap | `README.md` |
| Stable platform-wide standards | `HANDBOOK.md` |
| Nodes, networks, trust boundaries, placement, storage, and architecture | `architecture/README.md` |
| Significant durable decision and alternatives | `adrs/` |
| Repeatable operational procedure | `docs/` |
| Service configuration, state, access, deployment, validation, recovery, and troubleshooting | Service README |
| AI-specific navigation and behavior | `AGENTS.md` and `.agents/` |

Link to platform-wide rationale instead of duplicating it in every service
README.

## ADR triggers

Propose an ADR when a change:

- establishes or replaces a platform-wide pattern
- changes a trust boundary
- introduces direct public ingress or router port forwarding
- changes the storage or recovery model
- introduces a shared database or major dependency
- changes orchestration or deployment
- introduces a secret-management mechanism
- creates a new host role
- adds substantial operational complexity
- is expensive to reverse or materially constrains future choices

Do not rewrite an accepted ADR to hide history. Add a superseding ADR.

