# Forgejo

## Purpose

Forgejo provides private Git repository hosting and code-collaboration
capabilities for the personal cloud.

It is intended to:

* host personally owned source repositories
* provide issues, pull requests, releases, and repository administration
* reduce dependence on third-party Git hosting
* keep repository data and operational control on owned infrastructure

Forgejo is a stateful service. Its repositories, database, configuration, and
generated secrets must be treated as high-value durable data.

Platform-wide exposure decisions are documented in
[`../../adrs/0001-service-exposure-model.md`](../../adrs/0001-service-exposure-model.md).

## Host

Forgejo is prepared to run on the Mac mini compute node using OrbStack. It is
not a current workload until deployment and validation are complete.

The Mac mini is the initial placement because Forgejo is a user-facing
application rather than a management-plane service, and the host provides the
required compute and local persistent storage.

This placement makes Forgejo dependent on the Mac mini. Backups must therefore
be stored off-host so a Mac mini failure does not remove both the service and
its recovery data.

## Exposure

Forgejo uses separate web and Git SSH access paths.

### Browser access

The web interface is browser-accessible at:

```text
https://git.damienmurphy.net
```

The external path is:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> http://127.0.0.1:3002
  -> Forgejo
```

The HTTP origin binds to localhost because the Cloudflare Tunnel runs on the
same host.

Forgejo's own authentication remains enabled. Cloudflare Access is an
additional platform boundary rather than a replacement for application
authentication.

### Git SSH access

Git-over-SSH is private:

```text
Trusted device
  -> Tailscale
  -> Mac mini Tailscale address:2222
  -> Forgejo SSH
```

The SSH port must bind to the Mac mini's Tailscale address, not every host
interface. The advertised SSH hostname should be the Mac mini's stable
Tailscale MagicDNS name.

Git-over-HTTPS through the Cloudflare-protected hostname is not the normal clone
and push path. Command-line Git does not perform an interactive Cloudflare
Access login reliably. Machine access, webhooks, API clients, and package
clients require a separately designed authentication path before they are
enabled.

Administrative access to the host remains private through Tailscale or the
trusted local network.

## Dependencies

Forgejo depends on:

* OrbStack
* Docker Compose-compatible tooling
* persistent local storage
* Cloudflare Tunnel
* Cloudflare DNS
* Cloudflare Access
* Tailscale
* reliable off-host backup storage

The initial deployment does not include:

* a Forgejo Actions runner
* an external database
* a package or container registry access design
* anonymous public repositories
* inbound webhooks that bypass Cloudflare Access

These capabilities should be evaluated independently before being enabled.

## Configuration

Service configuration is defined in:

```text
compose/forgejo/docker-compose.yml
```

Host-specific configuration is supplied through a local `.env` file. Create it
from `.env.example` and replace every placeholder:

```bash
cp .env.example .env
```

The real `.env` file must not be committed.

Required settings include:

* UID and GID owning the persistent-data directory
* persistent-data path
* local HTTP port
* public web domain and root URL
* Tailscale SSH bind address
* advertised Tailscale SSH hostname and port
* registration and visibility policy

Forgejo also creates `/data/gitea/conf/app.ini`. Despite the historical `gitea`
path name, this is Forgejo's generated application configuration. It may contain
generated secrets and must remain outside Git.

The Compose definition uses SQLite because the expected personal workload is
small and does not justify a separate database service. Moving to PostgreSQL or
another database later would require a planned migration and recovery test.

## Persistent data

All persistent Forgejo data is mounted from:

```text
~/personal-cloud-data/forgejo/data/
```

to:

```text
/data
```

The directory contains or may contain:

* Git repositories
* the SQLite database
* `app.ini`
* generated application secrets
* SSH host keys and authorized-key data
* user avatars and attachments
* issue and pull-request data
* releases and release assets
* packages if the package registry is later enabled
* other application-generated state

The directory must be owned by the UID and GID configured through `PUID` and
`PGID`.

Do not place this directory inside the Git repository.

## Secrets

Potential secrets include:

* Forgejo administrator credentials
* generated application secret keys and internal tokens
* user access tokens
* OAuth or external-authentication credentials
* webhook secrets
* SMTP credentials
* SSH private host keys
* Cloudflare Tunnel credentials

Generated Forgejo secrets and SSH keys are stored within the persistent data
directory. Backups must therefore be encrypted and access-controlled.

Do not enable reverse-proxy header authentication. Forgejo application accounts
remain the identity source inside the application.

## Deployment

Before the first deployment:

1. Create `~/personal-cloud-data/forgejo/data/`.
2. Set its ownership to the configured `PUID` and `PGID`.
3. Create a local `.env` from `.env.example`.
4. Confirm the Tailscale bind address and MagicDNS hostname.
5. Confirm host ports `3002` and `2222` are available.
6. Confirm no router port-forwarding exposes the SSH port.

From the service directory:

```bash
cd compose/forgejo
docker compose config
docker compose pull
docker compose up -d
```

Inspect the deployment:

```bash
docker compose ps
docker compose logs --tail=100
```

Add `git.damienmurphy.net` to the Mac mini Cloudflare Tunnel and route it to:

```text
http://127.0.0.1:3002
```

Protect the hostname with a Cloudflare Access policy restricted to approved
identities.

Complete Forgejo's installation through the protected web interface and create
the initial administrator. Confirm that registration is disabled after the
administrator account exists.

## Validation

Validate the following:

1. The Compose configuration renders successfully without displaying secrets.
2. The container starts and reports healthy.
3. `http://127.0.0.1:3002/api/healthz` succeeds locally.
4. The first administrator can sign in.
5. New unauthorised registrations are disabled.
6. A private repository can be created.
7. The repository can be cloned over Tailscale SSH.
8. A commit can be pushed and fetched over Tailscale SSH.
9. `https://git.damienmurphy.net` resolves correctly.
10. Cloudflare Access blocks unauthorised users.
11. An authorised user can use the web interface.
12. The HTTP origin and SSH port are not exposed through the public internet.
13. Data persists across a controlled container restart.

Useful checks:

```bash
docker compose ps
docker compose logs --tail=100
curl -fsS http://127.0.0.1:3002/api/healthz
ssh -T -p 2222 git@mac-mini-tailnet-name
```

The SSH check is expected to authenticate through a Forgejo-managed user key,
not provide an interactive shell.

Verify that the selected container image contains `curl`, which the Compose
health check uses. If it does not, replace the health check with an available
HTTP client before considering the service operational.

## Monitoring

Add Forgejo to Uptime Kuma.

Recommended checks:

* private origin check for the Forgejo health endpoint
* HTTPS check for `https://git.damienmurphy.net`
* Mac mini host-reachability check

The public check validates DNS, Cloudflare Access, tunnel routing, and basic web
availability. It may validate the Cloudflare Access response rather than the
Forgejo application itself.

The private check validates Forgejo independently of Cloudflare.

Periodic functional validation should create or use a test repository and
perform an authenticated clone, push, and fetch. An HTTP response does not prove
that Git operations work.

## Backup and restore

Forgejo requires a full backup of:

```text
~/personal-cloud-data/forgejo/data/
```

The primary backup method is a consistent, encrypted, off-host copy of the
complete data directory.

For the initial SQLite deployment:

1. Stop Forgejo.
2. Snapshot or copy the complete data directory.
3. Verify the backup completed successfully.
4. Restart Forgejo.
5. Confirm the health endpoint and Git operations.

Stopping the service prevents repositories, the SQLite database, and other
application data from being captured at different points in time.

`forgejo dump` may be used as a supplementary export, but it is not the primary
recovery mechanism.

To restore:

1. Stop Forgejo.
2. Preserve or remove the current data directory.
3. Restore the complete backup to the configured path.
4. Confirm ownership and permissions.
5. Start Forgejo with the same or a compatible image version.
6. Run the Forgejo doctor checks.
7. Validate web login, repositories, issues, clone, fetch, and push.

Example diagnostic command:

```bash
docker compose exec --user git forgejo \
  doctor check --all --log-file /tmp/doctor.log
```

Restore tests should be performed periodically and before relying on the service
as the only copy of important repositories.

## Updating

The initial image is pinned to the Forgejo 15 LTS patch release:

```text
codeberg.org/forgejo/forgejo:15.0.5
```

Before updating:

1. Review Forgejo release notes and the upgrade guide.
2. Identify breaking changes and database migrations.
3. Confirm a recent, restorable full backup.
4. Record the current image version.
5. Update to an intended patch version.
6. Validate the Compose configuration.
7. Pull and deploy the image.
8. Inspect container health and logs.
9. Run `forgejo doctor check --all`.
10. Validate web login and Git clone, fetch, and push.

Major-version updates require manual review and validation. Do not use `latest`
or an unbounded major tag.

## Troubleshooting

### Service does not start

Check:

* persistent-data ownership and permissions
* `.env` variable names and values
* port conflicts
* OrbStack state
* image compatibility
* container logs

### Container is running but unhealthy

Test the health endpoint from the host. If the endpoint works, confirm that
`curl` is available inside the image and that the health-check command is valid.

### Web access works locally but not externally

Check:

* Cloudflare Tunnel status
* hostname route
* DNS
* Access policy
* `FORGEJO_ROOT_URL`
* origin address and port

### SSH clone or push fails

Check:

* Tailscale connectivity
* `FORGEJO_SSH_BIND_ADDRESS`
* host port `2222`
* advertised SSH hostname and port
* the user's registered public key
* local firewall rules
* Forgejo and SSH logs

Do not widen the SSH bind address merely to avoid diagnosing the private path.

### Generated links use the wrong hostname

Check `FORGEJO_DOMAIN`, `FORGEJO_ROOT_URL`, `FORGEJO_SSH_DOMAIN`, and
`FORGEJO_SSH_PORT`. Restart Forgejo after configuration changes.

### Repository or application data appears missing

Stop the service and verify the configured data mount before creating new
repositories or running repair commands. An empty or incorrect mount may cause
Forgejo to initialize fresh state.

## Removal

Before removal:

1. Confirm which repositories and application data must be retained.
2. Take and verify a final encrypted backup.
3. Stop the service.
4. Remove the Cloudflare Tunnel route.
5. Remove the Cloudflare Access application or policy.
6. Remove the DNS record.
7. Remove Uptime Kuma monitors.
8. Remove obsolete tokens and integration credentials.
9. Remove the container deployment.
10. Retain or securely delete the persistent data according to an explicit
    decision.

Stop and remove the container:

```bash
docker compose down
```

Do not delete Forgejo's persistent data as part of routine container removal.
