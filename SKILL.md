---
name: socks-wireguard-deploy
description: Deploy, migrate, distribute, or troubleshoot self-hosted proxy gateways. By default, build per-device Clash/Mihomo subscriptions containing VLESS Reality, VLESS XHTTP, TUIC, and Hysteria2 nodes, adapted to the VPS security-group ports; use SOCKS5 or WireGuard only when explicitly requested.
---

# Four-Protocol Proxy Deployment

Use this skill to turn a VPS into a resilient four-protocol proxy gateway and provide separate Clash/Mihomo subscriptions for the user's phone and computer.

## Default Deliverable

Unless the user explicitly requests a different topology, deploy all four node types in each requested device subscription:

1. **VLESS Reality** over TCP for a broadly compatible primary path.
2. **VLESS XHTTP** over TLS/TCP as the newer HTTP-based alternative to legacy TLS-WebSocket.
3. **TUIC** over QUIC/UDP for games, video, and mobile networks.
4. **Hysteria2** over UDP as an additional high-throughput, lossy-network path.

Generate a separate subscription for every concurrently used device. Phone and computer must have different randomized HTTPS paths and independent protocol credentials. Give nodes clear device-neutral names so each subscription shows four selectable nodes.

Do not add WireGuard, legacy TLS-WebSocket, or a selectable `DIRECT` node unless the user asks for it. China-direct routing rules may still send domestic traffic directly when requested.

## Port And Service Strategy

- Discover the provider security-group rules, host firewall, and current TCP/UDP listeners before choosing ports. TCP and UDP are separate namespaces and may reuse a number.
- Adapt to already-open ports when changing the provider console is unavailable. A proven compact example is Reality and XHTTP on TCP 443 through a tested routing/fallback arrangement, TUIC on UDP 3389, and Hysteria2 on UDP 22. Treat this as an example, not a universal hard-coded mapping.
- Never bind two independent processes to the same protocol/address/port. Sharing TCP 443 requires a deliberate SNI/fallback/reverse-proxy design that passes configuration and live tests.
- Preserve SSH reachability. Using UDP 22 does not conflict with SSH on TCP 22, but verify the distinction explicitly.
- Keep existing unrelated services working and do not reopen a protocol known to be unusable for this VPS unless the user requests a retest.

## Scope And Authority

- A request to inspect or diagnose authorizes read-only checks, packet capture, and temporary local test processes, but not persistent fixes.
- A request to deploy, migrate, repair, upgrade, or "make it work" authorizes the minimum related server, firewall, subscription, and client changes needed for the stated outcome.
- Preserve unrelated services and maintain a root-only rollback backup before persistent changes.
- Do not disable SSH password access, rotate unrelated credentials, or close a working transition port until replacement access passes live validation unless the user explicitly requests immediate revocation.
- Treat VLESS UUIDs, Reality private keys, TUIC/Hysteria2 passwords, subscription URLs, SOCKS credentials, and WireGuard private material as passwords. Never expose server-side private values or include them in routine logs.

## Non-Negotiable Invariants

1. Phone and computer receive separate credentials and separate randomized subscription URLs. Do not copy one device's secret-bearing subscription to another unless the user explicitly accepts shared credentials.
2. Update both runtime state and persistent configuration, then verify services survive a restart or an equivalent configuration reload check.
3. Validate the effective client configuration after Clash/Mihomo merges and overrides. Downloaded YAML alone does not prove proxy selection, TUN/system-proxy state, DNS, or IPv6 behavior.
4. A green UI badge or open port is not an end-to-end test. Test each of the four protocols through the VPS and verify the observed exit IP.
5. Make firewall, routing, and service changes idempotent. Detect existing rules and listeners before adding or replacing them.
6. Serve subscriptions only on exact randomized HTTPS paths. Return 404 elsewhere, disable access logs for secret paths, and send `Cache-Control: no-store, private`.
7. Remove temporary secret copies after validation. Keep only intended client subscriptions and root-only rollback material.

## Workflow Routing

- For deployment, migration, protocol upgrades, additional devices, or subscription distribution, read [references/deployment-runbook.md](references/deployment-runbook.md) completely before changing state.
- For timeouts, failed imports, DNS errors, one-protocol failures, destination-specific failures, or ambiguous UI status, read [references/diagnostics.md](references/diagnostics.md) completely before deciding on a fix.
- If a request includes both repair and a new deployment, read both references and identify the failing boundary before expanding changes.

## Completion Standard

A deployment is complete only when the requested device subscriptions contain all four nodes with independent credentials, server and client configuration syntax passes, exact HTTPS subscription delivery passes, each protocol completes a real proxied request with the VPS public exit IP, requested China-direct behavior works, and pre-existing services still pass their checks.

The handoff must label the phone and computer links, list the four node types, state exactly what was tested, disclose anything still requiring the user's physical device, and warn that each subscription link is a private credential.
