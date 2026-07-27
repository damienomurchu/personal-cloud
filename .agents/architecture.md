# Architecture Context

This file summarizes the current architecture for AI tools. The authoritative
sources are `architecture/README.md`, `HANDBOOK.md`, and accepted ADRs.

## Current nodes

| Role | Host | Runtime | Responsibilities |
| --- | --- | --- | --- |
| Compute | Mac mini | OrbStack and Compose-compatible tooling | Open WebUI, local LLM runtime, Calibre-Web, application compute, and locally attached application data |
| Management | Fedora ThinkPad X1 Carbon | Docker and Docker Compose | Uptime Kuma, monitoring, administration, troubleshooting, and recovery entry point |

Place services according to resource needs and failure boundaries, not merely
where capacity is available.

The management node is intentionally separate from the compute node. Monitoring
and recovery should remain available when the Mac mini fails or is under
maintenance.

## Connectivity and exposure

Use the narrowest exposure class that satisfies the requirement:

1. Localhost or an internal container network
2. Tailscale or the trusted local network
3. Cloudflare Tunnel plus Cloudflare Access for selected browser-facing services

Direct inbound router port forwarding is outside the normal architecture. An
exception requires explicit justification and a separate ADR.

Tailscale is the preferred remote administrative path. Administrative
interfaces should normally remain private. The local network is convenient but
not inherently trusted.

Browser-facing traffic normally follows:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare edge
  -> outbound Cloudflare Tunnel
  -> localhost-bound service origin
```

Cloudflare Access or Tailscale does not automatically replace useful
application-level authentication.

The default tunnel topology is one tunnel per host, routing multiple service
hostnames. A dedicated service tunnel requires a concrete isolation, lifecycle,
or administrative reason.

## Trust boundaries

Treat the following as distinct boundaries:

- public internet to Cloudflare identity and edge controls
- Cloudflare to the outbound tunnel
- tunnel to application origin
- Tailscale and local-network clients to hosts
- containers to host operating systems
- applications to persistent storage
- Git configuration to host settings and secrets

Access through one boundary does not grant unrestricted trust through the
others.

## Configuration and state

The repository defines how workloads should run. Hosts supply environment-
specific settings and secrets. External persistent storage contains durable
application data. Container instances are replaceable.

Target storage layout:

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

Do not assume the target layout matches every current Compose definition. Check
`.agents/known-gaps.md` and the affected service files.

## Failure and recovery model

Expected failures include stopped containers, failed upgrades, unavailable
hosts, full disks, disconnected tunnels, expired credentials, internet
outages, configuration drift, corrupt data, and machine replacement.

The intended recovery hierarchy is:

1. Restart the affected container.
2. Redeploy from the Compose definition.
3. Restore service configuration and data.
4. Rebuild the host and redeploy its services.
5. Move the service to another capable host.

This requires verified backups, recoverable secrets, host bootstrap knowledge,
and tested restore procedures. Several of these remain incomplete.

## Architectural direction

Likely future capabilities include dedicated storage, automated and
restore-tested backups, centralized notifications, host bootstrap automation,
shared Compose patterns, update automation, stronger metrics and logging, and
additional compute or storage nodes.

These are directions, not commitments. Introduce them only to solve a concrete
problem or create a useful capacity, failure, or ownership boundary.

