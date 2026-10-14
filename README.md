# socks-wireguard-deploy

Codex Skill for deploying, migrating, distributing, and troubleshooting self-hosted proxy gateways.

The default workflow covers VLESS Reality, VLESS XHTTP, TUIC, and Hysteria2, with separate Clash/Mihomo subscriptions per device. It includes discovery, root-only rollback backups, firewall and port checks, HTTPS subscription delivery, split-routing guidance, and end-to-end validation.

## Files

- `SKILL.md` — entry point and safety invariants
- `references/deployment-runbook.md` — deployment and migration workflow
- `references/diagnostics.md` — troubleshooting workflow
- `agents/openai.yaml` — Codex agent metadata

This repository contains no server addresses, private keys, API keys, subscription URLs, or generated proxy credentials.
