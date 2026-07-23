# ADR 0001: Service Exposure Model

* **Status:** Accepted
* **Date:** 2026-07-23
* **Decision owners:** Damien Murphy
* **Related documentation:** [`../architecture/README.md`](../architecture/README.md), [`../docs/service-onboarding.md`](../docs/service-onboarding.md)

## Context

The personal cloud runs services on privately owned machines within a home network.

Some services need to be accessed:

* only by processes on the same host
* from trusted devices on the local network
* remotely for administration
* through a browser from devices that may not be connected to the private network

The platform therefore needs a consistent exposure model that balances:

* ease of access
* security
* operational simplicity
* recoverability
* independence from the physical home network
* minimal direct exposure of internal systems

Without a common model, each service could develop its own combination of:

* open host ports
* router port forwarding
* application passwords
* VPN access
* reverse proxies
* tunnel configuration
* external authentication

That would create inconsistent trust boundaries and increase the chance of accidental exposure.

## Decision

Services will use one of three exposure classes:

1. **Localhost only**
2. **Private network through Tailscale or the trusted local network**
3. **Browser-accessible through Cloudflare Tunnel and Cloudflare Access**

Services must use the narrowest class that satisfies their access requirements.

Direct inbound router port forwarding will not be part of the normal service exposure model.

## Exposure classes

### Localhost only

Services that do not need to be accessed from another machine will bind only to localhost or remain on an internal container network.

Examples include:

* internal databases
* local AI model APIs
* helper services
* application backends used by a colocated frontend

Typical path:

```text
Local process
  -> localhost
  -> service
```

This is the preferred class when no remote access is needed.

### Private network

Administrative and non-public services will be accessed through:

* Tailscale for remote private access
* the local network where appropriate

Examples include:

* SSH
* host administration
* internal dashboards
* metrics endpoints
* container management
* storage administration
* recovery tooling

Typical remote path:

```text
Trusted device
  -> Tailscale
  -> private host or service
```

Tailscale is the preferred remote administrative access mechanism.

Administrative interfaces should not be published through Cloudflare merely for convenience when private network access is sufficient.

### Browser-accessible

Selected user-facing services will be made reachable through:

```text
Browser
  -> Cloudflare Access
  -> Cloudflare edge
  -> Cloudflare Tunnel
  -> local service
```

Examples include:

* Open WebUI
* Calibre-Web
* Uptime Kuma

Cloudflare Tunnel provides the network path without requiring inbound router port forwarding.

Cloudflare Access provides an identity-aware policy layer before traffic reaches the service.

Internet reachability does not imply anonymous public availability. Browser-accessible services will normally require authentication through Cloudflare Access.

## Decision rules

The exposure class will be selected using the following order:

1. Can the service remain local to the host?
2. Can the requirement be met through Tailscale?
3. Does the service require convenient browser access from devices that may not use Tailscale?
4. Does the value of external accessibility justify the additional attack surface and dependency?

Cloudflare exposure is used only when it materially improves usability.

Tailscale remains the default for administration.

## Application authentication

Cloudflare Access and Tailscale do not automatically replace application-level authentication.

Application authentication should remain enabled when it provides useful protection against:

* incorrect access policies
* lateral access from another trusted device
* direct local access
* tunnel misconfiguration
* shared-device use
* privilege separation between application users

Authentication may be simplified or disabled only where the service-specific risk is understood and documented.

## Tunnel topology

The default topology is one Cloudflare Tunnel per host, with multiple service hostnames routed through it.

Example:

```text
Mac mini tunnel
├── ai.damienmurphy.net -> http://127.0.0.1:3000
└── books.damienmurphy.net -> http://127.0.0.1:8083

Management node tunnel
└── status.damienmurphy.net -> http://127.0.0.1:3001
```

This model is preferred because it:

* limits the number of tunnel agents
* keeps routing aligned with host ownership
* reduces credential and process sprawl
* remains easy to understand

A dedicated tunnel may be introduced where a service requires:

* stronger credential isolation
* a different lifecycle
* independent failure handling
* a different administrative boundary

## Port binding

Where a Cloudflare Tunnel runs on the same host as the service, the service should normally bind to localhost.

Example:

```yaml
ports:
  - "127.0.0.1:3000:8080"
```

This prevents the service from being unnecessarily reachable through every host interface.

Where Tailscale or local-network access is also required, the binding may be widened intentionally.

The selected binding must be documented in the service README.

## DNS

Browser-accessible services will use service-specific subdomains under `damienmurphy.net`.

Examples include:

* `ai.damienmurphy.net`
* `books.damienmurphy.net`
* `status.damienmurphy.net`

DNS will be managed through Cloudflare and associated with the relevant tunnel route.

Naming should describe the user-facing capability rather than the underlying implementation where practical.

## Trust model

This decision creates distinct trust boundaries.

### Cloudflare path

The platform trusts Cloudflare to provide:

* DNS
* TLS termination
* tunnel transport
* Access policy enforcement
* identity integration

Cloudflare is not treated as the only security control.

The application, host, and stored data remain separate layers.

### Tailscale path

The platform trusts Tailscale to provide:

* authenticated device membership
* encrypted private connectivity
* device identity
* policy enforcement where ACLs or grants are used

Membership of the tailnet does not grant unrestricted application or host privileges.

### Local network

The local network is considered convenient but not inherently trusted.

Sensitive services should still use authentication and narrow port exposure.

## Rationale

### Why Cloudflare Tunnel

Cloudflare Tunnel allows selected services to be accessed without:

* opening inbound router ports
* exposing the home IP as the direct application endpoint
* operating a public reverse proxy at the network edge
* managing inbound NAT rules
* issuing and renewing origin-facing public certificates manually

The tunnel uses an outbound connection from the host to Cloudflare.

This creates a simpler and narrower ingress model for the current platform scale.

### Why Cloudflare Access

Cloudflare Tunnel alone provides reachability, not user authorisation.

Cloudflare Access adds an identity-aware control before traffic reaches the local service.

This centralises common access policy and reduces dependence on the varying authentication quality of individual self-hosted applications.

### Why Tailscale

Tailscale provides private access that maps well to host administration and trusted-device workflows.

It avoids making operational interfaces internet-facing and removes the need to maintain a traditional manually configured VPN.

It also provides a consistent access path across locations and changing home-network conditions.

### Why not use only Tailscale

Tailscale would provide a strong private-access model for all services, but requiring every browser or device to join the tailnet would add friction.

This is undesirable for services intended to be readily accessible through a normal browser.

Cloudflare Access and Tunnel are therefore used for selected user-facing services, while Tailscale remains the administrative path.

### Why not use only Cloudflare

Cloudflare is well suited to HTTP and browser-oriented access, but not every service should be routed through an internet-facing identity proxy.

SSH, databases, host management, metrics, and recovery tooling are better kept on a private network.

Using Cloudflare for all access would also increase dependence on:

* Cloudflare availability
* correct Access policy configuration
* internet connectivity
* an external identity provider

### Why no direct router port forwarding

Direct port forwarding would:

* expose the home network edge directly
* require additional reverse-proxy and TLS management
* increase the impact of host or proxy misconfiguration
* create a less consistent authentication model
* expose the public IP as the application origin
* provide little benefit relative to the selected tools

It may be considered only where Cloudflare Tunnel and Tailscale cannot meet a specific requirement.

Such an exception requires its own ADR.

## Consequences

### Positive consequences

* No normal requirement for inbound router port forwarding
* Clear distinction between user-facing and administrative access
* Consistent identity controls for browser-accessible services
* Private remote administration
* Reduced direct exposure of host services
* Simple service-specific DNS names
* Exposure decisions become repeatable during onboarding
* Monitoring can validate both public and private access paths
* New services inherit a secure default

### Negative consequences

* External browser access depends on Cloudflare
* Private remote access depends on Tailscale
* Identity and tunnel configuration exist partly outside Git
* Misconfigured Cloudflare Access policies may expose or block services
* Some services may require two authentication steps
* Host tunnel credentials must be securely stored and recoverable
* Internet loss prevents externally routed access, even where the local service remains healthy
* Multiple access paths can complicate monitoring and troubleshooting

### Operational consequences

Service documentation must record:

* exposure class
* hostname
* host port binding
* tunnel ownership
* Access policy
* Tailscale access requirements
* application authentication
* health-check path

The onboarding workflow in [`../docs/service-onboarding.md`](../docs/service-onboarding.md) implements these requirements.

## Failure modes

### Cloudflare outage

Browser-accessible services may become unavailable externally.

Private access through Tailscale or the local network should remain possible where configured.

### Tailscale outage or device deauthorisation

Remote administration may become unavailable.

Local network access should remain available from a trusted machine on-site.

### Tunnel process failure

The application may remain healthy locally while the external hostname fails.

Monitoring should distinguish between:

* local service health
* tunnel health
* public endpoint health

### Access policy misconfiguration

A valid user may be denied access, or an overly broad policy may permit unintended access.

Policies should be tested from:

* an authorised identity
* an unauthorised or logged-out session

### Application authentication failure

Cloudflare Access may still permit traffic while the application rejects the user.

This should be treated as a separate application-layer failure.

### Home internet outage

External Cloudflare and Tailscale access may be unavailable.

Local access should continue where the application and local network remain operational.

## Monitoring implications

Where useful, browser-accessible services should have two checks:

1. A local or private check that validates the application origin
2. A public check that validates DNS, Cloudflare, Access behaviour, tunnel routing, and the application

These checks answer different questions.

A public endpoint monitor alone cannot distinguish an application failure from a tunnel or Cloudflare failure.

A local monitor alone cannot confirm that users can reach the service.

## Security implications

The model reduces direct ingress but does not eliminate risk.

Controls still required include:

* application patching
* container image updates
* secure credentials
* least-privilege containers
* local firewalling
* restricted host ports
* secret rotation
* backup and restore
* access policy review
* device lifecycle management

Cloudflare Tunnel is not a substitute for secure application operation.

Tailscale membership is not a substitute for host authorisation.

## Alternatives considered

### Router port forwarding with a reverse proxy

A reverse proxy such as Caddy, Traefik, or NGINX could terminate TLS and route public traffic to services.

This was rejected as the default because it requires direct inbound exposure and adds network-edge operations that Cloudflare Tunnel currently removes.

A local reverse proxy may still be introduced for internal routing or configuration reuse without changing the external exposure decision.

### Traditional VPN

A manually operated WireGuard or OpenVPN deployment could provide private access.

This was rejected because Tailscale provides the required capability with less key distribution, routing, and device-management overhead.

### Tailscale Funnel

Tailscale Funnel could expose selected services publicly through the Tailscale network.

It was not selected as the primary browser-access model because Cloudflare already provides DNS, Access policy, and existing domain integration for the platform.

It may be reconsidered for specific use cases.

### Application authentication only

Services could be exposed publicly and rely only on their native login mechanisms.

This was rejected because application authentication quality and security posture vary significantly.

A common identity-aware edge provides a stronger and more consistent default.

### Cloudflare Access only, without application authentication

This would reduce login friction.

It was rejected as a universal rule because it creates excessive dependence on a single policy layer and may not provide sufficient user or privilege separation inside the application.

## Exceptions

A service may deviate from this model where required by:

* a non-HTTP protocol
* inbound webhooks that cannot pass through the existing controls
* peer-to-peer networking
* application incompatibility with reverse proxies or identity gates
* public anonymous access
* performance or latency requirements
* third-party callback constraints

Exceptions must document:

* why the standard model is insufficient
* the new trust boundary
* additional controls
* monitoring requirements
* recovery implications
* how the exception could later be removed

Material exceptions require a separate ADR.

## Review triggers

This decision should be reviewed when:

* the platform gains multiple permanent locations
* a dedicated firewall or edge gateway is introduced
* Cloudflare or Tailscale costs or terms materially change
* services require substantial non-HTTP ingress
* a shared internal reverse proxy becomes necessary
* the platform adopts Kubernetes or another orchestrator
* identity management becomes centralised elsewhere
* external users beyond the owner require differentiated access
* a significant exposure-related incident occurs

## Resulting standard

The default service exposure hierarchy is:

```text
Localhost
  preferred where possible

Tailscale or trusted local network
  default for private and administrative access

Cloudflare Tunnel plus Cloudflare Access
  used for selected browser-accessible services

Direct inbound port forwarding
  exception requiring explicit justification
```

This hierarchy should be applied during every service onboarding decision.

