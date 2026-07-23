# Uptime Kuma

## Purpose

Uptime Kuma provides availability monitoring for personal-cloud hosts, services, and external endpoints.

It is used to:

* detect service outages
* validate externally reachable application paths
* monitor host reachability
* provide a central operational status view
* separate monitoring from the primary compute node

Uptime Kuma is part of the platform management plane rather than a user workload.

Platform-wide exposure decisions are documented in [`../../adrs/0001-service-exposure-model.md`](../../adrs/0001-service-exposure-model.md).

## Host

Uptime Kuma runs on the Fedora-based ThinkPad X1 Carbon management node using Docker.

It is deliberately hosted separately from the Mac mini compute node so that it can detect:

* Mac mini outages
* application failures
* maintenance events
* Cloudflare Tunnel failures affecting compute-hosted services

The management node should remain powered, network-connected, and accessible even when the Mac mini is unavailable.

## Exposure

Uptime Kuma is browser-accessible at:

```text
https://status.damienmurphy.net
```

The external access path is:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> Uptime Kuma
```

Administrative host access remains private through Tailscale or the local network.

The service port should normally bind only to localhost when the Cloudflare Tunnel runs on the same host.

The status interface is private unless a separate public status page is intentionally configured.

## Dependencies

Uptime Kuma depends on:

* Docker
* Docker Compose
* persistent local storage
* Cloudflare Tunnel
* Cloudflare DNS
* Cloudflare Access
* Tailscale
* reliable power and network connectivity on the management node

Individual monitors may also depend on:

* DNS resolution
* public internet connectivity
* tailnet connectivity
* application credentials
* notification providers

## Configuration

Service configuration is defined in:

```text
compose/uptime-kuma/docker-compose.yml
```

Host-specific configuration should be supplied through a local `.env` file.

Typical configuration includes:

```dotenv
UPTIME_KUMA_PORT=3001
UPTIME_KUMA_DATA=/path/to/personal-cloud-data/uptime-kuma
TZ=Europe/Dublin
```

The actual variable names must match the Compose definition.

A safe `.env.example` should be committed.

The real `.env` file must not be committed.

Most monitor configuration is stored in the Uptime Kuma application database rather than the Compose file.

## Persistent data

Persistent Uptime Kuma data should live outside the repository.

Recommended location:

```text
~/personal-cloud-data/uptime-kuma/
└── data/
```

This data includes:

* monitors
* monitor history
* user accounts
* notification settings
* status pages
* application settings
* certificates or credentials stored by the application

Because most service configuration lives in the application database, this directory is required to rebuild the monitoring service without manual reconfiguration.

## Secrets

Potential secrets include:

* Uptime Kuma administrative credentials
* notification service tokens
* SMTP credentials
* ntfy credentials
* API keys
* monitor authentication credentials
* Cloudflare Tunnel credentials

Secrets stored through the Uptime Kuma interface become part of the persistent application data and must be protected through backup encryption and host access controls.

The Compose configuration should not contain plaintext credentials.

Cloudflare Tunnel credentials must remain outside Git.

## Deployment

From the service directory:

```bash
cd compose/uptime-kuma
docker compose config
docker compose pull
docker compose up -d
```

Inspect the deployment:

```bash
docker compose ps
docker compose logs --tail=100
```

On first deployment, complete the initial account setup through the local or Cloudflare-protected interface.

## Validation

Validate the following:

1. The Compose configuration renders successfully.
2. The container starts and remains healthy.
3. The service responds locally.
4. Existing monitors load correctly.
5. Monitor checks are running.
6. `https://status.damienmurphy.net` resolves correctly.
7. Cloudflare Access blocks unauthorised users.
8. Authorised users can access the dashboard.
9. Notifications work where configured.
10. The monitoring node can reach both public and private targets.

Useful commands:

```bash
docker compose config
docker compose ps
docker compose logs --tail=100
```

Example local test:

```bash
curl -I http://127.0.0.1:3001
```

Adjust the port to match the deployed configuration.

Trigger a controlled test where practical, such as temporarily monitoring an unused endpoint, to confirm alert delivery.

## Monitoring

Uptime Kuma is the monitoring service, but it should not be treated as self-verifying.

Its own availability should be checked through at least one independent mechanism where practical.

Possible approaches include:

* Cloudflare Tunnel health
* an external HTTP monitor
* a Tailscale host check
* a scheduled systemd or cron health check
* host-level service monitoring
* external notification when the container stops

Recommended monitor categories inside Uptime Kuma include:

### Public endpoint monitors

Examples:

```text
https://ai.damienmurphy.net
https://books.damienmurphy.net
https://status.damienmurphy.net
```

These validate the user-facing path.

### Private origin monitors

These validate the local application independently of Cloudflare.

### Host monitors

Monitor:

* Mac mini reachability
* ThinkPad reachability
* tailnet connectivity where useful

### Infrastructure monitors

Potential checks include:

* DNS resolution
* certificate expiry
* storage capacity
* Cloudflare Tunnel health
* backup freshness

Public and private checks should be named clearly so failures are easy to interpret.

## Backup and restore

Back up the Uptime Kuma persistent-data directory:

```text
~/personal-cloud-data/uptime-kuma/data/
```

This backup preserves:

* monitor definitions
* history
* users
* notification settings
* status pages
* application configuration

Before restoring:

1. Stop Uptime Kuma.
2. Restore the persistent-data directory.
3. Confirm ownership and permissions.
4. Start the service.
5. Verify monitors and notification integrations.
6. Confirm monitor checks resume normally.

Example:

```bash
docker compose down
# Restore data directory
docker compose up -d
docker compose logs --tail=100
```

A successful restore test should confirm:

* monitors are present
* historical data is readable
* notifications can be sent
* Cloudflare access works
* public and private targets are reachable

Because Uptime Kuma contains operational configuration, restoring it should be part of the platform disaster-recovery process.

## Updating

Before updating:

1. Review Uptime Kuma release notes.
2. Check for database or storage migrations.
3. Confirm a recent backup exists.
4. Record the current image version.

Update the image version in `docker-compose.yml`, then run:

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Validate:

* dashboard login
* monitor history
* active checks
* notification delivery
* Cloudflare access

Prefer explicit image versions over `latest` where practical.

## Troubleshooting

### Service does not start

Inspect:

```bash
docker compose logs --tail=200
```

Check:

* persistent-data path
* filesystem ownership
* environment variables
* port conflicts
* Docker service state
* available disk space

### Dashboard loads but monitors fail

Check:

* DNS resolution
* internet connectivity
* Tailscale connectivity
* firewall rules
* target authentication
* proxy settings
* target service availability

Distinguish between failure of the monitored service and failure of the monitoring node's network path.

### Public endpoint fails but origin monitor succeeds

Likely causes include:

* Cloudflare Tunnel failure
* DNS misconfiguration
* Cloudflare Access policy
* TLS or certificate issues
* public routing failure

### Origin monitor fails but public endpoint succeeds

Possible causes include:

* monitor using the wrong private address
* Tailscale routing issue
* local DNS differences
* host firewall rules
* the public endpoint being served through a different path

### Notifications do not arrive

Check:

* notification provider credentials
* destination settings
* provider availability
* rate limits
* firewall and DNS access
* Uptime Kuma logs
* test-notification result

### Uptime Kuma appears reset

Check whether the correct persistent-data directory is mounted.

An incorrect or empty mount may cause a fresh application database to be created.

Stop the service before replacing or restoring the database.

### Monitoring node becomes unreachable

Check:

* Fedora power-management settings
* lid-close behaviour
* network adapter state
* Tailscale service
* Docker service
* Cloudflare Tunnel service
* host power and internet connectivity

The management node should not suspend while operating as the monitoring plane.

## Removal

Before removal:

1. Export or document critical monitor definitions if required.
2. Back up the Uptime Kuma data directory.
3. Identify any alerts or operational processes that depend on it.
4. Stop the service.
5. Remove the Cloudflare Tunnel route.
6. Remove the Cloudflare Access application or policy.
7. Remove the DNS record.
8. Remove notification credentials.
9. Replace any monitoring coverage that is still required.
10. Retain or securely delete persistent data.

Stop and remove the service:

```bash
docker compose down
```

Do not remove Uptime Kuma without first deciding how remaining services will be monitored.

