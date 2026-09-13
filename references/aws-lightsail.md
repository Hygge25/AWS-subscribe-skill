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

## Verification and diagnosis

1. Check Xray and Hysteria2 service status and confirm separate TCP/UDP 443 listeners.
2. Parse the generated YAML with the same Mihomo binary used by Clash Verge.
3. Launch a temporary Mihomo process with a different mixed port. Fetch a small test endpoint through each named node individually; never switch the user's active profile just to test.
4. Verify the subscription with regular HTTPS validation, then ensure the fetched YAML has the expected hash and parses successfully.
5. For speed diagnosis, compare a fixed-size direct download, the same download through the temporary proxy, and server egress. Record test duration and endpoint. Only persist a bandwidth-hint change if it improves the proxy result.

Common outcomes:

- **TCP timeout:** check Lightsail TCP rule before Xray settings.
- **Reality EOF / invalid handshake:** confirm the public server address, current Reality client material, SNI, short ID, and Xray-version-specific field names. Test using the public address; server-private-path tests can be misleading.
- **Hysteria2 timeout:** check the distinct Lightsail UDP 443 rule, certificate hostname/IP validation, and password.
- **Slow but connected:** first verify the client is using the new profile/node. Then measure route limits before changing server parameters. The server's high egress benchmark does not guarantee the user's ISP-to-Tokyo route can match it.
