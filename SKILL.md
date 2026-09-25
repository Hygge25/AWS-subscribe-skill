---
name: aws-lightsail-proxy-subscription
description: Deploy, migrate, and troubleshoot a personal VLESS Reality and/or Hysteria2 Clash subscription on an AWS Lightsail Debian server, including IP-certificate renewal and monthly traffic labels. Use for a secure, updateable Clash Verge profile on Lightsail; do not use for unrelated proxy providers or generic AWS administration.
---

# AWS Lightsail Clash Subscription

Deploy a personal, secure Clash-compatible profile while preserving the user's existing active profile. Read [the AWS/Lightsail runbook](references/aws-lightsail.md) before changing a server. Its IP-migration and traffic-label sections apply to maintenance requests as well as new deployments.

## Choose the transport

- Prefer **VLESS + Reality + XTLS Vision over TCP 443** when the user has no domain, needs strong compatibility, or may be on a network that limits UDP.
- Add **Hysteria2 over UDP 443** when the user wants its QUIC/UDP performance characteristics. TCP 443 and UDP 443 can coexist on the same IP with no port conflict.
- Do not claim either transport is universally fastest. Measure from the user's network after deployment; Hysteria2 is often helpful on lossy paths, while Reality is the reliable fallback.

## Essential operating rules

- Treat SSH keys, client UUIDs, Hysteria2 passwords, Reality client material, and subscription paths as secrets. Never paste a private SSH key, server private key, or credentials into logs or user-visible artifacts beyond the intended private subscription/profile.
- Get the user's authorization before connecting to or changing a VPS. Make only the requested server and subscription changes.
- Use current official Xray, Hysteria2, Certbot, and AWS documentation rather than version-specific memory. Validate configuration with the installed binary before restarting a service.
- Preserve the user's active Clash Verge configuration. Creating or refreshing a separate profile is safe; do not activate it while diagnosing unless the user asks.
- A raw `vless://` URI or local YAML is useful for bootstrap, but an updateable subscription must be served over authenticated HTTPS. Do not expose credential-bearing YAML over plain HTTP.

## Workflow

1. Confirm SSH access, Debian version, sudo availability, TCP/UDP listeners, and Lightsail firewall state. Keep SSH access intact.
2. Install and configure only the requested transports. Use distinct random client credentials for Reality and Hysteria2. Keep Reality on TCP 443 and Hysteria2 on UDP 443 when both are requested.
3. Create a Clash/Mihomo YAML that includes explicit transport settings, secure DNS as appropriate, and a selectable group containing both nodes. Put the user's preferred node first only after it has passed a test.
4. Serve the YAML through a private HTTPS URL. Use a trusted certificate and a high-entropy, unguessable path. Configure automatic certificate renewal and ensure any Hysteria2 certificate copy/reload step happens after renewal.
5. Validate without touching the active Clash profile: syntax-test the YAML with the installed Mihomo binary, run a temporary local proxy on a non-conflicting port, fetch a small external endpoint, and verify country/region and TLS. Test the subscription URL with normal certificate validation.
6. If speed is disappointing, compare the user's direct connection, temporary proxy connection, and server egress before changing bandwidth hints. Update only parameters with measured benefit and re-test.
7. When the public IP changes, treat it as a certificate and client-endpoint migration—not a text-only substitution. Follow the runbook's IP migration checklist and perform a renewal dry run.
8. When monthly usage is requested, use persistent interface accounting and expose it as a duplicate, working HY2 node label. Keep the main HY2 node name stable so profile refreshes do not disturb the selected node.
9. For domestic-direct routing, keep Mihomo in `rule` mode, put private and China rules before the final proxy fallback, align DNS policy with those rules, and verify actual rule hits rather than syntax alone.

## Handoff

Report the active transports and ports, firewall rules required, verified (not assumed) connectivity result, maintenance timers, and a private subscription URL. State the limits clearly: physical latency, ISP routing, and destination-site IP reputation cannot be eliminated by server tuning.
