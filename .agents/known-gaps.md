# Known Gaps and Repository Drift

This file makes current limitations and observed inconsistencies visible to AI
tools. It is not authorization to fix them outside the scope of a user request.

## Platform limitations

- Backups are not fully automated.
- Restore testing is incomplete.
- Secrets are decentralized.
- Host provisioning is not reproducible.
- Deployments are primarily manual.
- Monitoring focuses mostly on availability.
- Storage-capacity and backup-freshness monitoring are incomplete.
- Some runtime state has not been moved fully outside the repository boundary.
- Image version and update policy is inconsistent.
- The Mac mini is a significant application failure domain.
- Storage architecture is evolving.
- Reusable Compose patterns and host-bootstrap automation do not yet exist.
- Independent verification of Uptime Kuma is not represented in repository
  configuration.

Treat these as visible constraints, not as implicit permission to redesign the
platform.

## Tracked environment files

Git currently tracks:

- `compose/calibre-web/.env`
- `compose/openweb-ui/.env`

This conflicts with the handbook and service documentation, which require real
environment files to remain outside Git.

Do not read, display, summarize, modify, or propagate their contents casually.
Treat them as potentially sensitive. Flag their tracked status before work that
touches environment configuration. Do not delete or untrack them unless that
change is explicitly in scope.

## Missing example configuration and ignore rules

- No service currently has a tracked `.env.example`, although the service
  documentation says one should be committed.
- No root `.gitignore` is tracked, despite the handbook defining expected
  exclusions.

This increases the risk of committing local configuration, secrets, generated
state, and backups.

## Persistent-state drift

### Calibre-Web

The service README recommends external application configuration, but Compose
currently mounts:

```text
./config:/config
```

This places generated application state under the repository service directory.
The library also uses a fixed host path instead of the documented
environment-driven target convention.

### Uptime Kuma

Compose currently mounts:

```text
/Users/me/code/data/uptime-kuma:/app/data
```

This conflicts with both the documented
`~/personal-cloud-data/uptime-kuma/data/` convention and the declared Fedora
deployment host because it is a macOS-style, user-specific path.

Do not silently normalize either discrepancy. Treat Compose as evidence of
current repository configuration and the documented layout as target state.

## Configuration-variable drift

- Open WebUI documentation refers to `OPENWEBUI_PORT` and `OPENWEBUI_DATA`,
  while Compose uses `OPEN_WEBUI_PORT` and `OPENWEBUI_DATA_DIR`.
- Calibre-Web documentation describes configurable configuration and library
  paths, while Compose uses fixed volume paths.
- Uptime Kuma documentation describes configurable port, data, and timezone
  values, while Compose hard-codes its port and data path and does not configure
  timezone.

Validate variable names against the actual Compose file before proposing
configuration examples.

## Image-version drift

- Open WebUI uses the floating `main` tag.
- Calibre-Web uses the floating `latest` tag.
- Uptime Kuma uses the broad major tag `1`.

The handbook and service READMEs prefer deliberate, bounded image versions when
unexpected upgrades would create unacceptable risk.

## Health-check drift

- Calibre-Web defines a container health check.
- Open WebUI and Uptime Kuma do not define container health checks.
- Their READMEs nevertheless call for checking that containers are healthy.

For services without a Compose health check, distinguish `running` from
`healthy` and use application-aware validation.

