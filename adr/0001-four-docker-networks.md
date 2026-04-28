---
title: "ADR-0001: Four Docker Bridge Networks and FQDN-Based Routing"
status: accepted
type: decision
owner: homelab
last_reviewed: 2026-04-28
---

# ADR-0001: Four Docker Bridge Networks and FQDN-Based Routing

**Date:** 2025-09
**Status:** Accepted

---

## Context

The homelab runs ~40 containerised services with very different trust and
visibility requirements: public web UIs exposed through a reverse proxy,
internal monitoring daemons, backend APIs and databases, and IoT device bridges.

Initially all services shared a single default Docker bridge network. This
created two problems:

1. **Blast radius** — any compromised container could reach every other container
   by IP, with no network-level isolation.
2. **Routing ambiguity** — multi-network services caused Traefik to pick the
   wrong network interface for routing, leading to intermittent 502 errors.

A secondary problem emerged over time: container IP addresses change after
every recreation, making hardcoded IPs in config files a maintenance burden.

---

## Decision

The homelab uses **four dedicated Docker bridge networks**, each with a single,
well-defined purpose:

| Network | Purpose |
|---------|---------|
| `traefik-public` | All services with a public web UI accessible via Traefik |
| `homelab-monitoring` | Internal monitoring stack (Prometheus, Loki, Grafana, Alertmanager, etc.) |
| `homelab-internal` | Backend services, databases, and APIs not exposed publicly |
| `homelab-iot` | IoT device bridges (MQTT broker, Zigbee coordinator, Node-RED) |

**FQDN rule:** all inter-container communication must use the Docker DNS name
`<container-name>.<network-name>` (e.g. `prometheus.homelab-monitoring:9090`),
never a raw IP address. IPs change on container recreation; FQDNs do not.

**Multi-network label rule:** any container attached to more than one network
and exposed through Traefik **must** carry the label
`traefik.docker.network=traefik-public`. Without it, Traefik may route through
the wrong interface.

These conventions are enforced via the new-container checklist
(`../docs/new-container-checklist.md`) and audited periodically.

---

## Consequences

### Positive

- Clear separation of concerns: monitoring traffic never crosses the public
  network; IoT traffic is isolated from everything else.
- Traefik routing is unambiguous for multi-network services.
- Network-level containment reduces lateral movement risk if a container is
  compromised.
- Configuration files referencing other containers use stable DNS names rather
  than volatile IPs.

### Negative / Trade-offs

- Services that need to talk across network boundaries must be explicitly joined
  to both networks, which increases cognitive overhead when adding a service.
- Every new container requires a deliberate network placement decision.
- The IoT network (`homelab-iot`) is largely dormant because the primary IoT
  services (Mosquitto, Zigbee2MQTT, Node-RED) are currently disabled.

### Neutral

- The number of networks (four) is a convention, not a technical limit.
  A fifth network may be introduced for a specific need (e.g. a dedicated
  database network) via a follow-up ADR.

---

## References

- `../docs/new-container-checklist.md` — checklist that enforces this decision
- `adr/0004-hybrid-monitoring-stack.md` — why monitoring lives in an isolated network
