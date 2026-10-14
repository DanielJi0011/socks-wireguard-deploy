# Four-Protocol Deployment And Migration Runbook

Read this reference for builds, repairs that require persistent changes, protocol upgrades, port migrations, new devices, or Clash/Mihomo subscription distribution. The default target is a four-node bundle: VLESS Reality, VLESS XHTTP, TUIC, and Hysteria2.

## 1. Discover Before Editing

Collect or discover:

- VPS SSH endpoint, operating system, public address model, and outbound interface.
- Existing SSH authentication, proxy/VPN services, listening TCP/UDP ports, provider security-group rules, host firewall, forwarding, and NAT rules.
- Existing Xray, sing-box, Nginx, certificate, subscription, SOCKS5, and optional WireGuard configuration.
- Requested SOCKS5 source allowlist, authentication requirement, port, and expected destinations.
- Every concurrently used client device, its compatible Clash/Mihomo client, and whether that client version supports Reality, XHTTP, TUIC, and Hysteria2.
- Desired routing policy: full tunnel or China direct / foreign proxy.

Use key-only SSH for ongoing automation. Before disabling passwords, prove the new key works in a separate non-interactive connection and confirm `sshd -T` reports the intended effective settings.

Create root-only backups of every persistent file that will change. Record paths in the handoff or deployment log without copying private values into chat.

## 2. SOCKS5 With Dante

For an authenticated Dante service:

- Bind the requested port explicitly and select the actual egress interface discovered from the default route.
- Use username authentication when the client supports it; generate a unique high-entropy password.
- Block loopback, RFC1918, link-local, and other internal destinations unless the use case explicitly needs them. This reduces SSRF and internal-network exposure.
- Allow only required protocols. UDP requires Dante UDP support, firewall UDP allowance, and a client that implements SOCKS5 UDP ASSOCIATE.
- Enforce the fixed source IP in both UFW/nftables and Dante client rules when practical. Do not rely on password authentication alone for a server intended for one upstream host.
- Keep credentials out of process arguments, shell history, and command output.

Validate from an allowed source, not only from the VPS itself:

- Port reachability and authentication.
- Exit IP through SOCKS5.
- One real target used by the workload.
- Denial from a non-allowlisted source when a second vantage point is available.

An IP-check success does not prove ChatGPT or another specific upstream is reachable. Preserve target-specific evidence.

## 3. Default Four-Protocol Server

Deploy the following unless the user explicitly requests a different set:

### VLESS Reality

- Use Xray or another implementation whose Reality support matches the client.
- Generate server Reality keys and a short ID without printing private material into chat or routine logs.
- Give every device a distinct VLESS UUID.
- Choose a credible TLS destination and server name that the VPS can reach reliably. Validate the effective Reality handshake, not merely the TCP listener.
- Prefer TCP 443 when available. If TCP 443 is shared with XHTTP or subscription HTTPS, use an explicit SNI/fallback or layer-4 routing design and verify each route independently.

### VLESS XHTTP

- Use XHTTP rather than legacy TLS-WebSocket for new deployments.
- Give every device a distinct VLESS UUID and use a high-entropy path that does not overlap subscription paths.
- Serve it behind valid TLS with a hostname that resolves to the VPS. Configure Nginx/Xray routing so ordinary subscription HTTPS and XHTTP can coexist without a catch-all exposing secret files.
- Match the server and Mihomo/Xray client XHTTP mode and transport fields. Validate with the intended client generation because older clients may not support XHTTP.

### TUIC

- Use sing-box or another current implementation and a valid TLS certificate/hostname.
- Give every device an independent UUID/password credential.
- Bind only an allowed UDP port. QUIC uses UDP; opening the same number only for TCP is insufficient.
- Use sane congestion-control and UDP relay defaults, then tune only after a real transfer test indicates a problem.

### Hysteria2

- Use sing-box or another current implementation and a valid TLS certificate/hostname.
- Give every device an independent high-entropy password.
- Bind a different UDP address/port from TUIC unless one process intentionally demultiplexes them and the implementation supports it.
- Avoid advertising unrealistic bandwidth values. Start conservatively and tune from measured VPS and network capacity.

### Compact Port Mapping

When the provider security group cannot be edited but already permits TCP 443, UDP 3389, and UDP 22, a known workable mapping is:

- VLESS Reality: TCP 443.
- VLESS XHTTP and subscription HTTPS: TCP 443 through tested SNI/fallback/reverse-proxy routing.
- TUIC: UDP 3389.
- Hysteria2: UDP 22.

This mapping is a fallback for constrained security groups, not a universal default. UDP 22 does not collide with SSH on TCP 22. Inspect listeners and the actual provider rules before applying it.

Keep WireGuard disabled unless the user explicitly requests it. If requested, use one peer/private key/preshared key/tunnel IP per device and follow safe migration rules; do not replace the four default nodes silently.

## 4. Clash/Mihomo Per-Device Configuration

Generate a separate YAML per device containing exactly one device-specific entry for each default protocol: Reality, XHTTP, TUIC, and Hysteria2. Each device subscription must use its own protocol credentials and randomized HTTPS URL.

Create one selection group containing all four nodes. Do not add a selectable `DIRECT` entry unless requested. Default the group to a conservative broadly compatible node such as Reality, while allowing the user to select XHTTP, TUIC, or Hysteria2.

For China-direct split routing, keep the ordering explicit:

1. Loopback and private networks to `DIRECT`.
2. Service-specific rules that must override geography.
3. China domains to `DIRECT`.
4. China IPs to `DIRECT`.
5. Final `MATCH` to the foreign proxy group.

DNS must not deadlock startup:

- Reality endpoints should preferably be IP-based when appropriate; TLS-based XHTTP, TUIC, and Hysteria2 still need the configured hostname/SNI and a directly reachable bootstrap resolver.
- Provide a directly reachable bootstrap/default resolver.
- Route foreign DoH through the foreign proxy only after the proxy can be established without that DoH lookup.
- Disable IPv6 unless the server, tunnel, DNS, routing rules, and client network have all been tested for IPv6. Check the effective runtime config because Clash frontends may override the subscription.

Validate every YAML using the same Mihomo generation family used by the client. Do not assume older Clash clients support XHTTP, TUIC, or Hysteria2. If compatibility is missing, explain the required client upgrade rather than silently omitting nodes.

## 5. HTTPS Subscription Distribution

Serve each device config at a different cryptographically random, exact HTTPS path. The URL grants use of the node because the body contains that device's private key.

Recommended Nginx behavior:

- Exact `location = /<random>.yaml` or an exact randomized directory plus a friendly filename.
- `application/yaml` content type.
- `Cache-Control: no-store, private` and `X-Content-Type-Options: nosniff`.
- Access logging disabled for these paths.
- All unknown paths return 404.
- HTTP redirects to HTTPS if certificates require port 80.

Use root ownership and grant the Nginx worker only the read access it needs. Certificate renewal must be enabled and tested.

## 6. Safe Apply Order

Use this order for a migration:

1. Back up persistent files with root-only permissions.
2. Generate new per-device secrets in a `mktemp -d` directory with `umask 077`.
3. Prepare and syntax-check candidate Xray, sing-box, Nginx, and client configs.
4. Open only the required TCP/UDP firewall rules and confirm provider security-group coverage.
5. Apply Xray, sing-box, and Nginx changes without interrupting SSH or unrelated services.
6. Publish subscriptions and reload Nginx only after all syntax checks succeed.
7. Refresh and activate one client through the client UI or supported local API.
8. Verify every protocol separately: connection, transfer, exit IP, split routing, and target sites.
9. Activate and verify additional physical devices when available.
10. Close the old port only after replacement success.
11. Re-test SOCKS5 and unrelated services.
12. Remove temporary keys and deployment scripts. Keep a root-only rollback backup until the user accepts the result.

If a step fails, stop expanding the change. Preserve the old path or restore the last known-good files before retrying.

## 7. Acceptance Tests

Server checks:

- Xray, sing-box, Nginx, and other scoped services are active.
- `ss` shows the intended protocol and port; distinguish TCP from UDP and confirm no unintended bind conflicts.
- Service logs show successful Reality/XHTTP/TUIC/Hysteria2 sessions during deliberate tests without exposing credentials.
- Required firewall rules are present and unrelated rules remain intact.
- Direct VPS egress reaches an IP-check service and the user's real upstream.

Subscription checks:

- HTTPS returns 200 with successful certificate verification and YAML content type.
- Downloaded content hash matches the published file.
- A random wrong path returns 404.
- Phone and computer configs have distinct VLESS, TUIC, and Hysteria2 credential fingerprints and distinct subscription paths.
- Each config contains Reality, XHTTP, TUIC, and Hysteria2 exactly once in the intended selection group.
- Mihomo syntax validation passes.

Client checks:

- The intended subscription is current, not merely present.
- The foreign group selects one of the four nodes, not an unintended bypass.
- System proxy or TUN is enabled according to the requested coverage.
- Foreign IP equals the VPS public IP.
- A domestic IP-check destination shows the local ISP when China-direct routing is requested.
- ChatGPT or another real foreign target completes TLS; an HTTP 403 can still prove successful TLS and routing.
- A domestic site remains fast and reachable.

Do not claim that a phone or computer works until that physical device activates the profile or an equivalent external client test exercises its credentials. It is acceptable to report that syntax, credentials, server registration, and public test clients passed while the physical device remains untested.
