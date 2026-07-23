# Calibre-Web

## Purpose

Calibre-Web provides a private browser-based interface for browsing, managing, and accessing the personal ebook library.

It is used to:

* access the ebook library across trusted devices
* browse books and metadata through a web interface
* retain ownership of the underlying library
* avoid depending on a third-party hosted library service

Platform-wide exposure decisions are documented in [`../../adrs/0001-service-exposure-model.md`](../../adrs/0001-service-exposure-model.md).

## Host

Calibre-Web runs on the Mac mini compute node using OrbStack.

The Mac mini currently hosts the application and provides access to the Calibre library and application configuration.

The service may later move or mount data from a dedicated storage node.

## Exposure

Calibre-Web is browser-accessible at:

```text
https://books.damienmurphy.net
```

The external access path is:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> Calibre-Web
```

Administrative access to the Mac mini remains private through Tailscale or the local network.

The application port should normally bind only to localhost when the Cloudflare Tunnel runs on the same host.

## Dependencies

Calibre-Web depends on:

* OrbStack
* Docker Compose-compatible tooling
* the Calibre ebook library
* persistent Calibre-Web configuration
* Cloudflare Tunnel
* Cloudflare DNS
* Cloudflare Access
* Tailscale for host administration

Optional integrations may introduce additional dependencies, such as Google Drive OAuth credentials.

## Configuration

Service configuration is defined in:

```text
compose/calibre-web/docker-compose.yml
```

Host-specific configuration should be supplied through a local `.env` file.

Typical configuration includes:

```dotenv
CALIBRE_WEB_PORT=8083
CALIBRE_WEB_CONFIG=/path/to/personal-cloud-data/calibre-web/config
CALIBRE_LIBRARY=/path/to/personal-cloud-data/calibre-web/library
PUID=1000
PGID=1000
TZ=Europe/Dublin
```

The actual variables must match the current Compose definition.

A safe `.env.example` should be committed.

The real `.env` file must not be committed.

Application-generated configuration must not remain inside the repository.

## Persistent data

Calibre-Web has two important persistent-data categories.

### Application configuration

Recommended location:

```text
~/personal-cloud-data/calibre-web/config/
```

This may include:

* `app.db`
* `gdrive.db`
* application settings
* user accounts
* metadata specific to Calibre-Web

### Calibre library

Recommended location:

```text
~/personal-cloud-data/calibre-web/library/
```

This includes:

* `metadata.db`
* ebook files
* cover images
* author and title directories
* library metadata

The library is the primary durable asset and must not be stored inside the Git repository.

Generated logs should also remain outside Git or be written to container output.

## Secrets

Potential secrets include:

* Calibre-Web administrative credentials
* OAuth client credentials
* `client_secrets.json`
* Google Drive tokens
* Cloudflare Tunnel credentials

Files such as the following must not be committed:

```text
client_secrets.json
app.db
gdrive.db
*.log
```

Secrets should be provided through local files or environment variables with restrictive permissions.

OAuth credentials should be backed up securely if recreating them would be difficult.

## Deployment

From the service directory:

```bash
cd compose/calibre-web
docker compose config
docker compose pull
docker compose up -d
```

Inspect the deployed service:

```bash
docker compose ps
docker compose logs --tail=100
```

Confirm that the configured library path contains a valid Calibre library before completing application setup.

## Validation

Validate the following:

1. The Compose configuration renders successfully.
2. The container starts and remains healthy.
3. The service responds locally.
4. The configured Calibre library is detected.
5. Books and metadata are visible.
6. A book can be opened or downloaded.
7. `https://books.damienmurphy.net` resolves correctly.
8. Cloudflare Access blocks unauthorised users.
9. Authorised users can access the application.
10. The origin is not directly reachable from the public internet.

Useful commands:

```bash
docker compose config
docker compose ps
docker compose logs --tail=100
```

Example local test:

```bash
curl -I http://127.0.0.1:8083
```

Adjust the port to match the deployed configuration.

## Monitoring

Calibre-Web should be monitored from Uptime Kuma.

Recommended checks:

* HTTPS check for `https://books.damienmurphy.net`
* local or Tailscale-based origin check where practical
* host reachability check for the Mac mini

A successful HTTP response does not confirm that the library is mounted correctly.

Periodic manual validation should include:

* opening the library
* selecting a book
* verifying metadata
* testing a download or browser read

## Backup and restore

Calibre-Web requires backup of both application configuration and the ebook library.

Back up:

```text
~/personal-cloud-data/calibre-web/config/
~/personal-cloud-data/calibre-web/library/
```

The Calibre library is the higher-value data set.

A consistent backup should avoid copying databases during active writes where practical.

Before restoring:

1. Stop Calibre-Web.
2. Restore the library directory.
3. Restore the application configuration directory.
4. Confirm file ownership and permissions.
5. Start the service.
6. Validate users, settings, metadata, and book access.

Example:

```bash
docker compose down
# Restore config and library data
docker compose up -d
docker compose logs --tail=100
```

A restore test should confirm that:

* `metadata.db` opens successfully
* books are visible
* files can be accessed
* application users and settings are intact

## Updating

Before updating:

1. Review upstream release notes.
2. Check for database migrations.
3. Confirm recent backups of both the library and configuration.
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

* login
* library visibility
* metadata
* book access
* OAuth integrations, if used
* Cloudflare access

Prefer explicit container image versions over `latest` where practical.

## Troubleshooting

### Service does not start

Inspect:

```bash
docker compose logs --tail=200
```

Check:

* volume paths
* file ownership
* environment variables
* port conflicts
* image compatibility

### Library is empty or unavailable

Check:

* the host library path
* the container mount path
* presence of `metadata.db`
* directory permissions
* whether an empty directory was mounted accidentally
* application library-path configuration

Do not create a new library until the original path has been verified.

### Books appear but cannot be opened

Check:

* ebook file presence
* library directory permissions
* path consistency
* file corruption
* browser or proxy size limits
* application logs

### Google Drive integration fails

Check:

* `client_secrets.json`
* OAuth redirect URLs
* token validity
* `gdrive.db`
* file permissions
* Cloudflare hostname configuration

OAuth files must remain outside Git.

### Local access works but the public hostname fails

Check:

* Cloudflare Tunnel status
* tunnel route
* DNS record
* Access policy
* origin address and port
* local service binding

### Application data appears reset

Check whether the correct configuration directory is mounted.

An empty or incorrect mount can cause Calibre-Web to initialise a new `app.db`.

Stop the service before making changes to database files.

## Removal

Before removal:

1. Back up the ebook library.
2. Back up application configuration if users or settings must be retained.
3. Stop the service.
4. Remove the Cloudflare Tunnel route.
5. Remove the Cloudflare Access application or policy.
6. Remove the DNS record.
7. Remove Uptime Kuma monitors.
8. Remove OAuth credentials that are no longer required.
9. Remove unused secrets.
10. Retain or securely delete the persistent data.

Stop and remove the service:

```bash
docker compose down
```

Do not delete the library as part of routine container removal.

