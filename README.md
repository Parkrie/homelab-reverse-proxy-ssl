# Internal Reverse Proxy with SSL and Dynamic DNS

Setting up an internal reverse proxy to give self-hosted services friendly hostnames and valid HTTPS certificates, accessible only through a VPN. This project covers reverse proxy configuration, dynamic DNS, certificate management, and internal DNS routing.

**Skills demonstrated:** Reverse proxy configuration, SSL/TLS certificate management, DNS management, containerization, LXC administration, zero-trust access, network security

---

## Why I Did This

Running self-hosted services by IP address and port number works but does not scale well. The goal here was to give every service its own subdomain with a valid HTTPS certificate, all while keeping everything completely internal with no ports open to the internet.

Rather than exposing services through port forwarding, all external access requires connecting through a VPN first. The reverse proxy never sees traffic from the public internet.

---

## Architecture

The reverse proxy runs inside an LXC container on a dedicated virtualization host, sitting on the server VLAN. It acts as the single entry point for all internal service traffic. Each service gets its own subdomain, and the router resolves every subdomain to the proxy container's internal IP. The proxy then routes requests to the correct service based on the hostname.

```

```

![Proxy Flow](https://raw.githubusercontent.com/Parkrie/homelab-reverse-proxy-ssl/main/diagrams/proxy-flow.png)

---

## Certificate Management

SSL certificates are issued automatically using a DNS challenge through the dynamic DNS provider. Because services are internal only and never directly reachable from the internet, a DNS challenge is required -- an HTTP challenge would fail since there is no public route to the proxy.

Certificates auto-renew before expiration. All services are served over HTTPS with automatic HTTP to HTTPS redirects.

---

## DNS Configuration

Each service subdomain follows the pattern `service.domain.example`. DNS A records are configured in the router's local DNS, pointing every subdomain to the proxy container's static internal IP.

Without these local overrides, subdomains resolve to the public WAN IP, which either fails or requires hairpin NAT. Pointing them to the proxy IP internally means resolution is fast and never touches the public internet.

---

## Access Control

No services are exposed to the internet. There is no port forwarding on the WAN interface. External access requires connecting through a mesh VPN first, after which services are reachable exactly as they are from inside the network.

All VPN users currently have access to all services. A self-hosted SSO solution is planned as a next step to add per-service authentication on top of the proxy.

---

## Container Setup

LXC was chosen over a full VM because the proxy is a lightweight service that does not need a fully virtualized OS. The container shares the host kernel while remaining isolated from other services, and has a static IP on the server VLAN that all DNS records point to.

---

## Security Design Rationale

**Confidentiality** is protected by ensuring no service is reachable without first authenticating to the VPN. All traffic is encrypted over HTTPS regardless of whether the request originates inside the house or through the VPN tunnel.

**Integrity** is supported by valid TLS certificates on every service. Certificate issuance is automated, eliminating the risk of services falling back to HTTP due to an expired cert.

**Availability** is addressed by centralizing routing through a single proxy. Adding or removing a service requires one proxy host entry and one DNS record rather than individual firewall and port forward changes on the router.

**Least privilege** is applied by keeping the proxy container on the server VLAN with no unnecessary outbound access. No service ports are exposed on the WAN or directly on the host.

**Threat model:** The primary threats addressed are unauthorized access to self-hosted services, unencrypted service traffic, and accidental exposure of internal services to the internet. VPN-only access, HTTPS enforcement, and internal DNS resolution directly mitigate all three.

---

## What I Learned

- Router-level DNS A records pointing to the proxy IP are essential for internal HTTPS to work. Without them subdomains resolve to the WAN IP and either fail or loop.
- A DNS challenge is the correct certificate issuance method for internal-only services. HTTP challenges require public reachability.
- LXC containers are a good fit for lightweight network services like a proxy. Lower overhead than a VM and easier to manage for something that just needs to forward traffic.
- Centralizing all service traffic through one proxy makes the setup easier to audit and manage over time.

---

## Planned Improvements

- Self-hosted SSO for centralized authentication across all services
- Access lists on sensitive services as an interim layer before SSO is in place
- Proxy access logging for visibility into service usage

---

## See Also

- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - DNS routing issue encountered during setup
