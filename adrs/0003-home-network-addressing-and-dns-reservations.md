# ADR-0003: Home Network Addressing and DNS Reservation Scheme

**Status:** Accepted
**Date:** 2026-08-16

## Context

The personal cloud uses the `192.168.0.0/24` LAN.

Infrastructure nodes require stable addresses for:

* DNS
* SSH
* monitoring
* firewall rules
* service configuration
* backup targets
* cluster configuration

Client devices do not require stable addresses unless an operational dependency is introduced.

Internal DNS uses the reserved `home.arpa` namespace.

## Decision

Use DHCP reservations for infrastructure and operator nodes.

Do not configure static addresses directly on hosts unless DHCP-independent bootstrap is required.

Use DNS names as the primary machine identity. IP addresses are infrastructure implementation details.

### Address allocation

| Range               | Purpose                                |
| ------------------- | -------------------------------------- |
| `192.168.0.1`       | Default gateway                        |
| `192.168.0.2-9`     | Network infrastructure                 |
| `192.168.0.10-19`   | Management infrastructure              |
| `192.168.0.20-39`   | Kubernetes nodes                       |
| `192.168.0.40-59`   | Non-Kubernetes servers and compute     |
| `192.168.0.60-79`   | Operator and trusted endpoints         |
| `192.168.0.80-99`   | Virtual IPs and service infrastructure |
| `192.168.0.100-199` | General DHCP clients                   |
| `192.168.0.200-239` | IoT and appliances                     |
| `192.168.0.240-254` | Experimental and reserved              |

### Initial reservations

| Address        | DNS name             | Role                          |
| -------------- | -------------------- | ----------------------------- |
| `192.168.0.10` | `mgt-1.home.arpa`    | Primary management node       |
| `192.168.0.40` | `mac-mini.home.arpa` | Non-Kubernetes compute/server |
| `192.168.0.60` | `x1.home.arpa`       | Operator endpoint             |

Future Kubernetes nodes will start at:

```text
192.168.0.20    k8s-1.home.arpa
192.168.0.21    k8s-2.home.arpa
192.168.0.22    k8s-3.home.arpa
```

### Client devices

Devices such as phones, tablets and e-readers remain in the general DHCP pool.

Examples:

```text
iPad
iPhone
Kobo
guest laptops
```

Reserve an address only when another system depends on that device having stable network identity.

### Virtual and service addresses

`192.168.0.80-99` is reserved for addresses not tied directly to physical hosts.

Expected uses include:

* Kubernetes `LoadBalancer` addresses
* MetalLB address pools
* HA virtual IPs
* reverse-proxy addresses
* service endpoints

Physical hosts must not consume this range.

## DNS

Internal names use:

```text
home.arpa
```

Examples:

```text
mgt-1.home.arpa
mac-mini.home.arpa
x1.home.arpa
k8s-1.home.arpa
```

Where supported, DHCP should distribute:

```text
search home.arpa
```

This permits short-name resolution:

```bash
ssh mgt-1
```

instead of:

```bash
ssh mgt-1.home.arpa
```

Configuration and documentation should prefer DNS names over literal IP addresses where practical.

## Naming

Prefer role-oriented hostnames where the role is expected to survive hardware replacement.

Examples:

```text
mgt-1
k8s-1
k8s-2
storage-1
```

Hardware-oriented names are acceptable for unique machines whose hardware identity is operationally meaningful:

```text
mac-mini
x1
```

## Consequences

* Infrastructure addresses remain stable while configuration stays centrally managed through DHCP.
* Address ranges communicate node role.
* Physical and virtual address spaces remain separate.
* Hosts can be replaced without redesigning the network.
* DNS becomes the stable interface for consumers of infrastructure.
* Ordinary client devices require no reservation management.

## Rule

Reserve an IP when another system is permitted to depend on that node's network identity.

Otherwise, use dynamic DHCP.

