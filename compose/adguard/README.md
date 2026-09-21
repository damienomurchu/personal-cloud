# AdGuard Home

AdGuard Home provides DNS filtering for devices on the Tailscale network.

This instance runs on `mgt-1` via Docker Compose.

## Scope

Initial scope is Tailscale only.

```text
Tailnet clients
      │
      ▼
AdGuard Home
    mgt-1
      │
      ▼
Encrypted upstream DNS
```

The home LAN does not depend on this service.

## Paths

Persistent data:

```text
~/personal-cloud-data/adguard/
├── conf/
└── work/
```

Container paths:

```text
/opt/adguardhome/conf
/opt/adguardhome/work
```

## Network

Steady-state ports:

```text
53/tcp     DNS
53/udp     DNS
8080/tcp   Admin UI
```

All ports should be bound to the Tailscale IP of `mgt-1`.

Port `3000/tcp` is required only for initial setup.

Get the Tailscale IP:

```bash
tailscale ip -4
```

Check for an existing DNS listener:

```bash
sudo ss -lntup | grep ':53 '
```

Resolve any port `53` conflict before starting AdGuard Home.

## Setup

Create persistent directories:

```bash
mkdir -p ~/personal-cloud-data/adguard/{conf,work}
```

Start the service:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

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
- "<TAILSCALE_IP>:3000:3000/tcp"
```

Then reconcile:

```bash
docker compose up -d
```

## Expected Port Mapping

Steady state:

```yaml
ports:
  - "<TAILSCALE_IP>:53:53/tcp"
  - "<TAILSCALE_IP>:53:53/udp"
  - "<TAILSCALE_IP>:8080:80/tcp"
```

Do not bind DNS or the admin UI to all host interfaces unless that is intentional.

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

Only then configure Tailscale to use `mgt-1` as its DNS resolver.

## Tailscale

Configure the `mgt-1` Tailscale IP as the tailnet DNS server.

Do not configure a public resolver as a secondary DNS server on clients. Clients may use it directly and bypass AdGuard.

Redundancy should come from a second controlled resolver, not a public fallback.

## Operations

Check service state:

```bash
docker compose ps
```

View recent logs:

```bash
docker compose logs --tail=100
```

Restart:

```bash
docker compose restart
```

Stop:

```bash
docker compose down
```

Start:

```bash
docker compose up -d
```

## Update

Pull the configured image version:

```bash
docker compose pull
```

Recreate:

```bash
docker compose up -d
```

Validate:

```bash
docker compose ps
docker compose logs --tail=100
```

Use explicit image versions rather than `latest`.

## Backup

Back up:

```text
~/personal-cloud-data/adguard/conf/
~/personal-cloud-data/adguard/work/
```

The container is disposable. Persistent state is not.

## Failure Recovery

If tailnet DNS stops working:

1. Check the container:

```bash
docker compose ps
```

2. Check logs:

```bash
docker compose logs --tail=100
```

3. Test AdGuard directly:

```bash
dig @<TAILSCALE_IP> example.com
```

4. Check upstream DNS configuration.

5. If necessary, remove or disable the AdGuard DNS configuration in Tailscale until the service is restored.

## Design Decisions

* Tailscale-only initially to limit blast radius.
* Encrypted upstream DNS to reduce ISP visibility.
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
