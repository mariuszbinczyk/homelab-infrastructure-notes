# homelab-infrastructure-notes

Notes from running a self-hosted homelab — ~40 Docker containers on a single x86 server.
Mostly written for my own reference. Sharing in case you're solving something similar.

## Architecture decisions

Short records explaining *why* certain choices were made.

| | Decision |
|-|----------|
| [ADR-0001](adr/0001-four-docker-networks.md) | Four Docker bridge networks + container DNS instead of IP addresses |
| [ADR-0002](adr/0002-systemd-timers-instead-of-cron.md) | systemd timers instead of root crontab |
| [ADR-0003](adr/0003-puid-1000-with-root-exceptions.md) | Run containers as UID 1000, document every root exception |
| [ADR-0004](adr/0004-hybrid-monitoring-stack.md) | Prometheus + Uptime Kuma + AI alert classification to cut noise |

## Stack

Docker · Traefik · Prometheus · Loki · Grafana · Home Assistant · n8n · Ollama · AdGuard

---

*Questions or issues: open an issue.*
