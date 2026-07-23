# Open WebUI

## Purpose

Open WebUI provides a private browser-based interface for interacting with locally hosted language models.

It is used to:

* access models running on the Mac mini
* provide a consistent web interface across trusted devices
* experiment with local AI-assisted workflows
* keep selected prompts, conversations, and model usage under personal control

Platform-wide exposure decisions are documented in [`../../adrs/0001-service-exposure-model.md`](../../adrs/0001-service-exposure-model.md).

## Host

Open WebUI runs on the Mac mini compute node using OrbStack.

The Mac mini is the appropriate host because it also provides the local AI runtime and has sufficient memory and compute capacity for local models.

## Exposure

Open WebUI is browser-accessible at:

```text
https://ai.damienmurphy.net
```

The external access path is:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> Open WebUI
```

Administrative access to the host remains private through Tailscale or the local network.

The service port should normally bind only to localhost when the Cloudflare Tunnel runs on the same host.

## Dependencies

Open WebUI depends on:

* OrbStack
* Docker Compose-compatible tooling
* a local model runtime, currently Ollama or an equivalent API-compatible service
* Cloudflare Tunnel
* Cloudflare DNS
* Cloudflare Access
* persistent local storage
* Tailscale for host administration

The local model runtime must be reachable from the Open WebUI container.

## Configuration

Service configuration is defined in:

```text
compose/openweb-ui/docker-compose.yml
```

Host-specific configuration should be supplied through a local `.env` file.

Typical configuration includes:

```dotenv
OPENWEBUI_PORT=3000
OPENWEBUI_DATA=/path/to/personal-cloud-data/openweb-ui
OLLAMA_BASE_URL=http://host.internal:11434
```

The actual variable names must match the current Compose definition.

A safe `.env.example` should be committed alongside the Compose file.

The real `.env` file must not be committed.

## Persistent data

Persistent Open WebUI state should live outside the repository.

Recommended location:

```text
~/personal-cloud-data/openweb-ui/
└── data/
```

This data may include:

* user accounts
* conversations
* application settings
* uploaded knowledge files
* model connection configuration
* application metadata

Model files themselves are managed separately by the local model runtime and should not be stored inside the Open WebUI service directory.

## Secrets

Potential secrets include:

* Open WebUI application secret keys
* OAuth credentials, if configured
* external model API keys
* Cloudflare Tunnel credentials
* administrative passwords

Secrets must be stored outside Git and supplied through:

* a local `.env` file
* mounted secret files
* the application's secure configuration mechanism

Cloudflare Tunnel credentials are managed separately from the Compose configuration.

## Deployment

From the service directory:

```bash
cd compose/openweb-ui
docker compose config
docker compose pull
docker compose up -d
```

Inspect the deployed service:

```bash
docker compose ps
docker compose logs --tail=100
```

Confirm that the local model runtime is running before testing model access.

## Validation

Validate the following:

1. The Compose configuration renders successfully.
2. The container is running and healthy.
3. The service responds locally.
4. The service can reach the local model runtime.
5. A model can be selected and queried successfully.
6. `https://ai.damienmurphy.net` resolves correctly.
7. Cloudflare Access blocks unauthorised users.
8. An authorised user can sign in and use the service.
9. The origin is not directly reachable from the public internet.

Useful commands:

```bash
docker compose config
docker compose ps
docker compose logs --tail=100
```

Example local test:

```bash
curl -I http://127.0.0.1:3000
```

Adjust the port to match the deployed configuration.

## Monitoring

Open WebUI should be monitored from Uptime Kuma.

Recommended checks:

* HTTPS check for `https://ai.damienmurphy.net`
* local or Tailscale-based origin check where practical
* host reachability check for the Mac mini

The public check validates:

* DNS
* Cloudflare Access
* Cloudflare Tunnel
* application availability

The private check validates the application independently of Cloudflare.

A simple HTTP status check may not prove that model inference is working. Periodic manual validation should include submitting a prompt to a local model.

## Backup and restore

Back up the persistent Open WebUI data directory if conversation history, users, settings, or uploaded knowledge files need to be retained.

Back up:

```text
~/personal-cloud-data/openweb-ui/data/
```

The local model files do not necessarily need to be backed up if they can be downloaded again.

Before restoring:

1. Stop the service.
2. Preserve or remove the existing data directory.
3. Restore the backup to the configured persistent-data path.
4. Confirm ownership and permissions.
5. Start the service.
6. Validate user access, settings, conversations, and model connectivity.

Example:

```bash
docker compose down
# Restore data
docker compose up -d
docker compose logs --tail=100
```

Restore procedures should be tested after significant application or database-version changes.

## Updating

Before updating:

1. Review the Open WebUI release notes.
2. Check for database migrations or breaking configuration changes.
3. Confirm a recent backup exists.
4. Record the current container image version.

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
* conversation history
* model discovery
* prompt execution
* Cloudflare access

Prefer an explicit version tag over `latest` where practical.

## Troubleshooting

### Service does not start

Inspect logs:

```bash
docker compose logs --tail=200
```

Check:

* environment variables
* mounted data paths
* filesystem permissions
* port conflicts
* image compatibility

### Models are unavailable

Check:

* the local model runtime is running
* the configured model API URL
* host-to-container networking
* OrbStack host aliases
* model availability in the runtime
* local firewall rules

Test the model API directly from the host before debugging Open WebUI.

### Local service works but the public hostname fails

Check:

* Cloudflare Tunnel process status
* tunnel route configuration
* DNS records
* Cloudflare Access policy
* origin port and protocol
* whether the service is bound to the expected address

### Cloudflare login succeeds but Open WebUI rejects access

Cloudflare Access and Open WebUI authentication are separate layers.

Check:

* Open WebUI user account state
* application authentication settings
* session cookies
* application logs

### Application data appears missing

Check:

* the configured volume path
* whether a new empty directory was mounted
* filesystem permissions
* whether the Compose project is using the intended `.env`

Do not initialise a fresh application database until the original data location has been identified.

## Removal

Before removal:

1. Decide whether conversations, settings, and uploaded files must be retained.
2. Back up or export required data.
3. Stop the service.
4. Remove the container.
5. Remove the Cloudflare Tunnel route.
6. Remove the Cloudflare Access application or policy.
7. Remove the DNS record.
8. Remove Uptime Kuma monitors.
9. Remove unused secrets.
10. Retain or securely delete persistent data.

Stop and remove the service:

```bash
docker compose down
```

Do not remove persistent data until its retention decision has been made.

