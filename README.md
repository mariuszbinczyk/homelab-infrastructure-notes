# Homelab Infrastructure Notes

Architecture decisions and operational documentation from a self-hosted homelab
(~40 Docker containers, x86 server, Ubuntu 24.04).

Built and maintained over ~18 months. The goal of this repository is to show
that good infrastructure documentation is achievable in a personal project —
not just in enterprise teams.

Documentation follows the [Diátaxis](https://diataxis.fr/) framework.
Architecture decisions are captured as [ADRs](https://adr.github.io/) — short,
durable records of *why* something was decided, not just *what*.

## Stack

| Area | Technology |
|------|------------|
| Reverse proxy | Traefik v3 |
| Monitoring | Prometheus · Loki · Grafana · Uptime Kuma |
| Alert routing | Alertmanager → n8n → Ollama (AI classification) |
| Log aggregation | Loki + Promtail |
| Automation | systemd timers (backup · audit · CVE scan · tier updates) |
| Security scanning | Trivy (weekly CVE baseline) |
| Container runtime | Docker + Docker Compose |
| Home automation | Home Assistant |
| DNS / ad-blocking | AdGuard Home |

## Architecture Decision Records

ADRs explain the *reasoning* behind key technical decisions. Each covers: the
context that forced a choice, the decision made, and the tradeoffs accepted.

| # | Decision | Date |
|---|----------|------|
| [ADR-0001](adr/0001-four-docker-networks.md) | Four dedicated Docker bridge networks + FQDN-only inter-container routing | 2025-09 |
| [ADR-0002](adr/0002-systemd-timers-instead-of-cron.md) | Replace root crontab with version-controlled systemd timers | 2026-03 |
| [ADR-0003](adr/0003-puid-1000-with-root-exceptions.md) | All containers run as UID/GID 1000 — root exceptions require inline documentation | 2025-10 |
| [ADR-0004](adr/0004-hybrid-monitoring-stack.md) | Prometheus + Loki + Uptime Kuma hybrid monitoring with AI-enhanced alert classification | 2025-10 |

## Operations documentation

| Document | Type | What it covers |
|----------|------|----------------|
| [New Container Checklist](docs/new-container-checklist.md) | How-to | End-to-end onboarding: compose file → network → monitoring → DNS → git |
| [Container Permissions Strategy](docs/permissions-strategy.md) | Explanation | UID policy, filesystem ACL, root exception protocol |
| [Alerting Strategy](docs/alerting-strategy.md) | Explanation | Dual-layer alerting: Uptime Kuma + Prometheus + AI noise reduction |
| [Health Check Strategy](docs/healthcheck-strategy.md) | Explanation | Docker healthchecks vs Prometheus scraping — decision matrix + full container audit |

## Document metadata

Every document carries a YAML frontmatter block:

```yaml
---
status: active          # active | historical | superseded | draft
type: explanation       # tutorial | how-to | reference | explanation | decision
owner: homelab
last_reviewed: YYYY-MM-DD
---
```

This makes it straightforward to find stale documentation and distinguish
active guides from historical records.

> **Note:** Some operational documents contain inline notes in Polish
> (native language of the author). Architectural documentation and ADRs
> are in English.

---

*Questions or feedback: open an issue.*
