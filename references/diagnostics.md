# Diagnostics And Failure Matrix

Read this reference when a proxy shows Timeout, a subscription cannot be imported, one of the four default protocols fails, a phone and computer behave differently, or UI state disagrees with traffic.

## Evidence Order

Diagnose from outside in:

1. Subscription HTTPS availability and content freshness.
2. Effective client profile, selected proxy, system proxy/TUN state, and runtime port.
3. Raw IP traffic through the local proxy to separate tunnel failure from DNS failure.
4. Protocol-specific packet arrival and service logs: Reality/XHTTP on TCP and TUIC/Hysteria2 on UDP.
5. Server listeners, TLS/SNI routing, firewall, route, MTU/QUIC behavior, and direct egress.
6. Target-specific reachability and IP reputation.

Do not change multiple layers before identifying the failing boundary.

## Common Signatures

### Subscription unavailable

Symptoms: non-200 response, TLS failure, stale client file, or import error.

Check Nginx exact path, file permissions, certificate, DNS, HTTP redirect, content type, YAML syntax, client feature support, and client download logs. This is configuration distribution failure, not proof that any node protocol is down.

### Profile loaded but traffic bypasses the node

Symptoms: no server-side test session and foreign IP remains local, while the client UI is running.

Check the actual current profile, proxy group selection (`DIRECT` versus node), system proxy, TUN, Android VPN state, and per-application exclusions. Read runtime-generated config rather than only subscription metadata.

### Raw IP and domain both time out

If a request to an IP such as `1.1.1.1` through the local proxy also times out, DNS is not the primary cause. Inspect the selected protocol's transport path and server logs.

### Domain fails but raw IP works

This points to DNS policy, poisoned results, unavailable DoH, IPv6 selection, or a DNS bootstrap loop. Check logs for `dns resolve failed`, test the configured resolvers directly, and verify whether the frontend merged `ipv6: true` over the subscription.

### Both node and DIRECT latency show Timeout

Do not conclude that both network paths are dead. Clash may test both against the same blocked foreign URL, and DNS for that URL may itself be routed through the failed node. Test a raw IP, a domestic site, and a foreign site separately.

### Reality fails while TCP 443 is reachable

Check the selected profile, UUID, Reality public key, short ID, server name, fingerprint, time synchronization, and whether TCP 443 routing sends that SNI to the Reality inbound. A reachable TCP port does not prove a valid Reality handshake.

### XHTTP fails while subscription HTTPS works

Check XHTTP path and mode, host/SNI, TLS certificate, reverse-proxy route, HTTP version behavior, and client support. A 200 response for the YAML proves only subscription delivery; it does not prove the XHTTP transport path.

### Server sees no TUIC or Hysteria2 UDP packets

Check wrong endpoint/port, stale subscription, client not selected, local firewall, upstream UDP filtering, VPS provider security group, and public-IP/NAT mapping. Confirm the server is listening on UDP, not merely TCP with the same number. Diagnose TUIC and Hysteria2 ports separately.

### UDP packets arrive but TUIC or Hysteria2 does not authenticate

Check the exact device credential, TLS hostname/SNI, certificate chain and validity, ALPN/protocol settings, server/client clocks, and whether the packets reach the intended UDP listener. Then test the same device on cellular and Wi-Fi to distinguish credential/configuration errors from return-path UDP filtering. Do not change MTU first when authentication itself has not completed.

### Protocol authenticates but no Internet traffic passes

Verify the server can reach the target directly, inspect outbound routing and DNS, and confirm the client actually selected the tested node. If small requests work but large transfers stall, investigate MTU, QUIC path quality, congestion control, and provider shaping without applying one protocol's tuning blindly to the others.

### Exit-IP site works but ChatGPT/TikTok fails

The tunnel is up. Check target-specific routing, ASN/datacenter IP reputation, destination filtering, account/device region checks, and TLS logs. For TikTok App, distinguish browser behavior from App behavior and consider SIM MCC, account region, location, app source, and cached region state.

For a SOCKS5 path, successful authentication and an IP-check result do not prove target reachability. Preserve errors such as TTL expired, EOF, or upstream timeout and test the target from the proxy server itself.

## Multi-Device Credential Isolation

Phone and computer should have separate VLESS UUIDs, TUIC credentials, Hysteria2 passwords, and subscription URLs. Shared credentials make revocation and diagnosis ambiguous even when the protocol technically permits concurrent use. Test one device credential at a time when attributing server logs.

## Packet Capture Discipline

Use a bounded capture such as a packet count or short timeout on the exact TCP or UDP port. Correlate timestamps with one deliberate client request and one named protocol. Avoid simultaneous tests that make log attribution ambiguous.

After every temporary diagnostic process, confirm its local port and process are gone.

## Fix Validation

After a fix, require all relevant evidence again:

- A fresh successful session for each changed protocol.
- Increasing traffic or a completed real request through that protocol.
- Raw IP request through the client proxy.
- Foreign exit IP equals the VPS.
- Domestic direct IP remains local when requested.
- Real target TLS or HTTP result.
- All four default nodes, existing HTTPS subscription service, SSH, and unrelated services still work.

Only then close old firewall ports or clean rollback material.
