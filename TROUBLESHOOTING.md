# Troubleshooting Log

Issues encountered while setting up the reverse proxy.

---

## Issue 1: Subdomains Not Resolving to the Proxy Internally

**Symptom:**

After setting up the reverse proxy and configuring proxy hosts for each service, subdomains were not loading correctly from inside the network. The browser either timed out or reached the wrong destination.

**What I checked:**

The dynamic DNS domain and subdomains were configured correctly and resolving fine from outside the network. The proxy hosts were set up in the reverse proxy with the correct internal service addresses and ports. The certificates had issued successfully.

**Root cause:**

The DNS A records for each subdomain were pointing to the public WAN IP address. From outside the network that works fine since traffic hits the router's public IP and gets forwarded. But from inside the network, requests to the WAN IP either fail, loop back incorrectly, or depend on hairpin NAT being configured on the router.

The fix required adding local DNS A records in the router pointing each subdomain directly to the internal IP address of the proxy container. This way, any device on the internal network resolves the subdomain straight to the proxy rather than going out to the internet and back in.

The mistake was setting the A records to the proxy IP rather than the service IPs. Since the reverse proxy is the single entry point for all services, all subdomains need to point to the proxy container IP. The proxy handles routing from there based on the hostname. Setting individual A records to each service IP bypasses the proxy entirely.

**Fix:**

Updated all DNS A records in the router to point to the proxy container's static internal IP address. After that, every subdomain resolved correctly from inside the network, certificates worked, and HTTPS loaded without issues.

**Takeaway:**

When using a reverse proxy for internal services, every subdomain needs a local DNS entry pointing to the proxy IP, not the individual service IP. The proxy's job is to receive all hostname-based requests and route them to the right place. Pointing DNS directly at services bypasses that entirely and breaks the whole setup.

This also means the proxy container needs a static IP. If the IP changes, every DNS record breaks simultaneously. Assign a fixed address to any infrastructure component that other things depend on.

---

## General Notes

Outside of the DNS routing issue the setup was straightforward. The main things worth noting for anyone doing something similar:

- Get the LXC container IP set as static before configuring anything else. Everything else depends on it.
- Set up local DNS overrides in the router before testing anything from inside the network. Without them you will get misleading results.
- Use a DNS challenge for certificate issuance on internal-only services. An HTTP challenge requires public reachability which defeats the purpose of keeping services internal.
