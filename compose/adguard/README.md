# AdGuard Home

AdGuard Home provides DNS filtering for devices on the Tailscale network.

This deployment runs via Docker Compose and uses host-specific `.env` files so the same Compose definition can be reused across multiple devices.

## Scope

Initial scope is Tailscale only.

```text
Tailnet clients
      │
      ▼
AdGuard Home
      │
      ▼
Encrypted upstream DNS
```

The home LAN does not depend on this service.

## Repository Layout

```text
adguard/
├── docker-compose.yml
├── .env.example
├── .env.mgt-1
├── .env.mac-mini
├── .gitignore
└── README.md
```

Commit:

* `docker-compose.yml`
* `.env.example`
* `.gitignore`
* `README.md`

Do not commit host-specific `.env` files.

## Environment Files

The Compose file expects host-specific values from an env file.

Example:

```dotenv
TAILSCALE_IP=100.x.x.x
DATA_DIR=/home/damien/personal-cloud-data/adguard
```

For another host:

```dotenv
TAILSCALE_IP=100.y.y.y
DATA_DIR=/Users/damien/personal-cloud-data/adguard
```

Use `.env.example` as the template:

```dotenv
TAILSCALE_IP=
DATA_DIR=
```

Create a host-specific file:

```bash
cp .env.example .env.mgt-1
```

Then populate the values for that device.

## Git Ignore

Use:

```gitignore
.env
.env.*
!.env.example
```

This keeps host-specific configuration out of source control while retaining the template.

## Paths

Persistent data is stored outside the container.

Example on `mgt-1`:

```text
/home/damien/personal-cloud-data/adguard/
├── conf/
└── work/
```

Container paths:

```text
/opt/adguardhome/conf
/opt/adguardhome/work
```

The host path is provided through `DATA_DIR`.

## Network

Steady-state ports:

```text
53/tcp     DNS
53/udp     DNS
8080/tcp   Admin UI
```

All ports are bound to the host's Tailscale IP.

Port `3000/tcp` is required only for initial setup.

Get the local Tailscale IP:

```bash
tailscale ip -4
```

Check for an existing DNS listener:

```bash
sudo ss -lntup | grep ':53 '
```

Resolve any port `53` conflict before starting AdGuard Home.

## Compose Variables

The Compose file should reference environment variables explicitly:

```yaml
ports:
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:53:53/tcp"
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:53:53/udp"
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:3000:3000/tcp"
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:8080:80/tcp"

volumes:
  - "${DATA_DIR:?DATA_DIR is required}/work:/opt/adguardhome/work"
  - "${DATA_DIR:?DATA_DIR is required}/conf:/opt/adguardhome/conf"
```

Required-value syntax causes Compose to fail immediately if a variable is missing.

## Setup

Create the persistent directories for the target host:

```bash
mkdir -p "${DATA_DIR}/"{conf,work}
```

If running manually from a shell, either export `DATA_DIR` first or create the directories using the path from the selected env file.

For example:

```bash
mkdir -p /home/damien/personal-cloud-data/adguard/{conf,work}
```

## Start

Run Compose with the env file for the target host:

```bash
docker compose --env-file .env.mgt-1 up -d
```

For another host:

```bash
docker compose --env-file .env.mac-mini up -d
```

Check status:

```bash
docker compose --env-file .env.mgt-1 ps
```

View logs:

```bash
docker compose --env-file .env.mgt-1 logs -f
```

Always use the same env file for follow-up Compose operations on that host.

## Initial Configuration

Open:

```text
http://<TAILSCALE_IP>:3000
```

Configure:

```text
Admin interface: 0.0.0.0:80
DNS interface:   0.0.0.0:53
```

Docker controls host-level interface exposure.

After setup, the admin UI is available at:

```text
http://<TAILSCALE_IP>:8080
```

Remove the temporary setup port from `docker-compose.yml`:

```yaml
- "${TAILSCALE_IP:?TAILSCALE_IP is required}:3000:3000/tcp"
```

Then reconcile:

```bash
docker compose --env-file .env.mgt-1 up -d
```

## Expected Port Mapping

Steady state:

```yaml
ports:
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:53:53/tcp"
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:53:53/udp"
  - "${TAILSCALE_IP:?TAILSCALE_IP is required}:8080:80/tcp"
```

Do not bind DNS or the admin UI to all host interfaces unless intentional.

## Upstream DNS

Use encrypted upstream DNS.

Recommended options:

* Quad9
* Mullvad DNS
* Cloudflare

Prefer DNS-over-HTTPS or DNS-over-TLS.

Enable DNSSEC.

Keep blocklists conservative. Avoid multiple overlapping lists unless there is a clear reason.

## Validation

Test AdGuard directly before changing Tailscale DNS:

```bash
dig @<TAILSCALE_IP> example.com
```

or:

```bash
nslookup example.com <TAILSCALE_IP>
```

Verify:

* DNS queries resolve
* queries appear in AdGuard Home
* upstream DNS works
* blocked domains are blocked
* allowed domains resolve normally

Only then configure Tailscale to use the resolver.

## Tailscale

Configure the host's Tailscale IP as a tailnet DNS server.

Do not configure a public resolver as a secondary DNS server on clients. Clients may use it directly and bypass AdGuard.

Redundancy should come from a second controlled resolver.

## Multi-Host Deployment

The same Compose file is reused across hosts.

Example:

```text
mgt-1
  └── .env.mgt-1

Mac mini
  └── .env.mac-mini
```

Each host provides its own:

* `TAILSCALE_IP`
* `DATA_DIR`

The service definition remains unchanged.

This keeps host-specific configuration out of the Compose file and makes additional deployments predictable.

## Operations

Check service state:

```bash
docker compose --env-file .env.mgt-1 ps
```

View recent logs:

```bash
docker compose --env-file .env.mgt-1 logs --tail=100
```

Restart:

```bash
docker compose --env-file .env.mgt-1 restart
```

Stop:

```bash
docker compose --env-file .env.mgt-1 down
```

Start:

```bash
docker compose --env-file .env.mgt-1 up -d
```

## Update

Pull the configured image version:

```bash
docker compose --env-file .env.mgt-1 pull
```

Recreate:

```bash
docker compose --env-file .env.mgt-1 up -d
```

Validate:

```bash
docker compose --env-file .env.mgt-1 ps
docker compose --env-file .env.mgt-1 logs --tail=100
```

Use explicit image versions rather than `latest`.

## Backup

Back up the host-specific `DATA_DIR`.

For example:

```text
/home/damien/personal-cloud-data/adguard/conf/
/home/damien/personal-cloud-data/adguard/work/
```

The container is disposable. Persistent state is not.

The `.env` file should also be recoverable, but it contains only host-specific deployment values.

## Failure Recovery

If tailnet DNS stops working:

1. Check the container:

```bash
docker compose --env-file .env.mgt-1 ps
```

2. Check logs:

```bash
docker compose --env-file .env.mgt-1 logs --tail=100
```

3. Test AdGuard directly:

```bash
dig @<TAILSCALE_IP> example.com
```

4. Check upstream DNS configuration.

5. If necessary, remove or disable the AdGuard DNS configuration in Tailscale until the service is restored.

## Design Decisions

* One Compose definition for all hosts.
* Host-specific values supplied through env files.
* `.env.example` committed; real env files ignored.
* Tailscale-only initially to limit blast radius.
* Encrypted upstream DNS.
* No Unbound.
* Explicit Tailscale interface binding.
* No public client-side DNS fallback.
* Persistent state stored outside the container.
* Image versions pinned for reproducible upgrades.

## Future

Likely next steps:

* second AdGuard Home instance on the Mac mini
* configuration backup automation
* local DNS rewrites
* Uptime Kuma monitoring
* controlled expansion to LAN clients

