# AWS Lightsail Reality + Hysteria2 runbook

Use this reference for a personal Debian Lightsail instance. Substitute all values in angle brackets; do not save real credentials in a skill or a repository.

## Architecture and firewall

| Service | Transport | Port | Lightsail inbound rule |
| --- | --- | --- | --- |
| Xray VLESS + Reality | TCP | 443 | TCP 443 from intended clients (or public Internet if required) |
| Hysteria2 | UDP | 443 | UDP 443 from intended clients (or public Internet if required) |
| HTTPS subscription / ACME validation | TCP | 80, or another explicitly opened HTTPS port | TCP 80 if using HTTP-01 or serving HTTPS there |
| SSH | TCP | 22 | Restrict to the user's fixed IP where feasible |

Cloud firewall rules and OS listeners are independent. A process listening on the server does not mean the Lightsail firewall admits the traffic. A successful TCP socket connection only proves TCP reachability; it does not validate a Reality or Hysteria2 handshake.

## Reality server shape

Install Xray from its official release mechanism. Generate a new UUID, an X25519 key pair, and a short ID per user/device. A server configuration needs:

```json
{
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "<UUID>", "flow": "xtls-rprx-vision"}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "raw",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "target": "<TLS_1.3_TARGET>:443",
        "serverNames": ["<TLS_1.3_TARGET>"],
        "privateKey": "<SERVER_PRIVATE_KEY>",
        "shortIds": ["<SHORT_ID>"]
      }
    }
  }],
  "outbounds": [{"protocol": "freedom"}]
}
```

Use a target that the server can reach over TLS 1.3 and whose certificate matches the selected server name. The Xray project has renamed some client-side Reality fields over time; generate or validate against the installed Xray version instead of relying on old templates.

Mihomo profile shape:

```yaml
- name: <NAME>
  type: vless
  server: <SERVER_IP_OR_HOST>
  port: 443
  uuid: <UUID>
  network: tcp
  udp: true
  tls: true
  flow: xtls-rprx-vision
  servername: <TLS_1.3_TARGET>
  client-fingerprint: chrome
  reality-opts:
    public-key: <REALITY_CLIENT_MATERIAL>
    short-id: <SHORT_ID>
```

## Hysteria2 server shape

Install Hysteria2 via its official current installer or release. It must have a valid TLS certificate readable by its service user and a long random password.

```yaml
listen: :443
tls:
  cert: /etc/hysteria/tls/cert.pem
  key: /etc/hysteria/tls/key.pem
auth:
  type: password
  password: <HY2_PASSWORD>
masquerade:
  type: proxy
  proxy:
    url: https://<BENIGN_HTTPS_SITE>/
    rewriteHost: true
```

Client profile shape:

```yaml
- name: <NAME>
  type: hysteria2
  server: <SERVER_IP_OR_HOST>
  port: 443
  password: <HY2_PASSWORD>
  sni: <CERTIFICATE_NAME_OR_IP>
  skip-cert-verify: false
  up: "<MEASURED_UPLOAD> Mbps"
  down: "<MEASURED_DOWNLOAD> Mbps"
```

Do not set `skip-cert-verify: true` merely to avoid certificate work. Hysteria2's `up`/`down` are client congestion-control hints, not server capacity. Start conservatively and tune only using a repeatable measurement from the user's network.

## HTTPS subscription

The subscription URL is a bearer secret because it contains node credentials. Use a long random path, restrict the served directory permissions, return 404 elsewhere, and set `Cache-Control: no-store`.

Prefer a domain with a conventional HTTPS endpoint. If no domain is available, current Let's Encrypt supports short-lived IP-address certificates; verify current Certbot and Let's Encrypt documentation before relying on this. IP certificates have short lifetimes, so renewal automation is mandatory. If a service such as Hysteria2 runs under an unprivileged user, copy renewed certificate/key files to a tightly permissioned service-readable directory and restart it via a renewal post-hook.

Avoid binding a subscription web server to TCP 443 if Xray already owns that port. A nonstandard HTTPS port can work in a URL, but its matching Lightsail TCP rule must be added. If using TCP 80 for HTTPS after ACME issuance, renewal must briefly free port 80 or use a suitable webroot mechanism.

## Public IP changes and IP certificates

An ordinary Lightsail public IPv4 address can change after stop/start. Attaching a Lightsail static IP prevents accidental endpoint changes, but do that only after confirming the chosen address has acceptable destination-site reputation.

Treat an IP change as a coordinated migration:

1. Confirm SSH reaches the same intended instance at the new address and re-check listeners and service health.
2. Test important destinations directly from the server before changing proxy configuration. This separates server-egress or IP-reputation failures from transport failures.
3. Issue a new trusted certificate whose SAN contains the new IP. With a current Certbot that supports IP certificates, use the current short-lived IP-certificate profile and standalone validation. Verify the exact command against current official documentation.
4. Update the Nginx `server_name` and certificate paths, the Hysteria2 certificate copy/reload hook, and every client `server`/IP `sni` field. Reality credentials and its camouflage SNI usually do not need rotation merely because the public IP changed.
5. Disable or archive the old IP's renewal configuration so unattended renewal does not repeatedly fail. Keep a reversible backup until the new endpoint is proven.
6. Validate Nginx, Xray, and Hysteria2 configuration before restarting. Fetch the subscription with normal TLS verification and parse it with the user's Mihomo binary.
7. Run `certbot renew --cert-name <NEW_IP> --dry-run --run-deploy-hooks`, then verify Nginx and Hysteria2 are active and the subscription remains reachable.
8. Ensure a persistent timer checks the short-lived certificate at least daily. The post-hook must copy the renewed certificate/key to the Hysteria-readable directory with restrictive permissions, restart Hysteria2, and restore Nginx after standalone validation.

Do not reuse the old IP certificate for a new address, set `skip-cert-verify: true`, or leave both old and new renewal jobs active as shortcuts.

## Destination blocking and Google `automated queries`

When Reality works but HY2 does not, first force each named node through a temporary Mihomo instance and test the same URL. If both fail only for one destination, test that destination directly from the VPS. If direct server egress receives the same rejection, the cause is the AWS exit IP or destination policy—not HY2, Reality, DNS routing, or a missing Clash rule.

For Google Scholar, test both the homepage and a small search request. A fresh IP may remove an `automated queries` rejection, but this is not guaranteed and the reputation can change again. Do not silently route Scholar, X, or Telegram through a third-party provider to hide an AWS reputation problem; add external routing only when the user explicitly requests it.

## Persistent monthly traffic label

Use `vnStat` on the public interface for lightweight monthly accounting. State its scope accurately: interface totals include proxy traffic plus small amounts of SSH, certificate renewal, package downloads, and subscription fetches. It is not per-user or per-protocol accounting.

For a Clash Verge-visible label:

1. Enable `vnstat.service` and identify the public interface rather than assuming its name.
2. Read the current calendar month's RX + TX bytes from `vnstat --json`. If accounting begins mid-month immediately after a reboot, optionally store a one-time, month-keyed offset from the interface counters so already-observed boot traffic is not lost. Never carry that offset into the next month.
3. Keep the ordinary HY2 proxy name stable, for example `AWS-JP-HY2`.
4. Duplicate its complete, working Hysteria2 definition and name the duplicate `📊 本月累计 <VALUE> GB`. Use the same server, certificate validation, authentication, and obfuscation settings; do not create a dead or fake endpoint merely for display.
5. Add the traffic-label node to the manual selector, but not to the automatic fallback group. Selecting it should still provide a normal HY2/UDP connection.
6. Update both the duplicate proxy name and its selector reference atomically every 15 minutes. Use a systemd oneshot service and persistent timer. Validate that exactly one proxy definition and one selector reference carry the label.
7. Because the server-side YAML changes do not rewrite a profile already cached by Clash Verge, tell the user to refresh the subscription to see the latest label.

Renaming the main HY2 node on every counter update can reset client selection. A separate working duplicate avoids that disruption while still appearing as a Hysteria2/UDP node.

## Verification and diagnosis

1. Check Xray and Hysteria2 service status and confirm separate TCP/UDP 443 listeners.
2. Parse the generated YAML with the same Mihomo binary used by Clash Verge.
3. Launch a temporary Mihomo process with a different mixed port. Fetch a small test endpoint through each named node individually; never switch the user's active profile just to test.
4. Verify the subscription with regular HTTPS validation, then ensure the fetched YAML has the expected hash and parses successfully.
5. For speed diagnosis, compare a fixed-size direct download, the same download through the temporary proxy, and server egress. Record test duration and endpoint. Only persist a bandwidth-hint change if it improves the proxy result.
6. If a dynamic traffic-label node exists, force one request through that exact node name and confirm it reaches a small HTTPS endpoint. Syntax validation alone does not prove its duplicated credentials work.

Common outcomes:

- **TCP timeout:** check Lightsail TCP rule before Xray settings.
- **Reality EOF / invalid handshake:** confirm the public server address, current Reality client material, SNI, short ID, and Xray-version-specific field names. Test using the public address; server-private-path tests can be misleading.
- **Hysteria2 timeout:** check the distinct Lightsail UDP 443 rule, certificate hostname/IP validation, and password.
- **Slow but connected:** first verify the client is using the new profile/node. Then measure route limits before changing server parameters. The server's high egress benchmark does not guarantee the user's ISP-to-Tokyo route can match it.
