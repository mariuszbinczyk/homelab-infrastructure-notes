---
title: "ADR-0004: Prometheus + Loki + Uptime Kuma Hybrid Monitoring Stack"
status: accepted
type: decision
owner: homelab
last_reviewed: 2026-04-28
---

# ADR-0004: Prometheus + Loki + Uptime Kuma Hybrid Monitoring Stack

**Date:** 2025-10 (initial); enhanced 2026-02 with AI classification
**Status:** Accepted

---

## Context

A homelab serving a family needs monitoring that catches both "the service is
down" (availability) and "the service is degraded" (resource exhaustion, log
errors) without generating alert fatigue.

Early attempts with a single alerting system produced two recurring problems:

1. **Prometheus alone** fires on metrics thresholds but cannot detect that an
   HTTP endpoint returns 200 with wrong content, or that a container is healthy
   but unreachable from the LAN.
2. **Alert fatigue** — a set of naive threshold rules on memory, CPU, and disk
   produced constant low-priority noise, causing important alerts to be ignored.

Additionally, the monitoring stack itself needed to stay running even when the
containers it monitors are being restarted, requiring careful network isolation
(see ADR-0001).

---

## Decision

The homelab uses a **three-layer monitoring architecture**:

### Layer 1 — Availability (Uptime Kuma)

Uptime Kuma monitors HTTP/TCP endpoints from outside the monitored service's
network, providing fast failure detection (30 s interval). Notifications go
directly to the push channel (ntfy). This layer is independent of Prometheus
and continues working even if the metrics stack is degraded.

### Layer 2 — Metrics and Logs (Prometheus + Loki + Grafana)

- **Prometheus** scrapes metrics from cAdvisor, node-exporter, and
  service-specific exporters. All memory metrics use
  `container_memory_working_set_bytes` (not `usage_bytes`, which includes
  filesystem cache and overstates usage by 30–100%).
- **Loki + Promtail** aggregate container logs. Promtail uses the
  `containerlogs` job label (not `dockerlogs`) and strips the leading `/`
  from container names.
- **Grafana** provides dashboards and unified alert management.

### Layer 3 — AI Classification (Alertmanager → n8n → Ollama)

Alertmanager routes firing alerts to an n8n workflow that invokes a local LLM
(phi4-mini via Ollama) to classify alert severity, filter noise, and compose
a human-readable summary before dispatching to the notification channel. This
significantly reduces false-positive pages without requiring manual threshold
tuning.

### Critical metric rules

| Rule | Correct | Wrong |
|------|---------|-------|
| Memory | `container_memory_working_set_bytes` | `container_memory_usage_bytes` |
| Counters | `increase(counter[5m]) > 0` | `counter > 0` (fires forever) |
| Log job label | `containerlogs` | `dockerlogs` |

---

## Consequences

### Positive

- Uptime Kuma provides availability signals independently of Prometheus, so
  a Prometheus scrape failure does not silence availability alerts.
- AI classification eliminates the most common sources of alert fatigue
  without manual threshold maintenance.
- Loki centralises logs from all containers; no need to `docker logs` each
  service individually during an incident.
- The two-layer design means Layer 1 catches "service down" in ~30 s, while
  Layer 2 catches "service degraded" within the scrape interval (60 s).

### Negative / Trade-offs

- Four additional containers (Prometheus, Loki, Grafana, Alertmanager) add
  steady-state memory overhead (~500 MB combined).
- The AI classification step adds 5–20 s of alert dispatch latency. For
  critical alerts this is acceptable; for response times < 1 min it is not.
- The n8n + Ollama dependency in the alert path means a failure in n8n or
  Ollama will delay or drop alerts. Mitigated by Uptime Kuma's independent path.
- Maintaining correct Loki label conventions (job, container_name format)
  requires discipline across every new container onboarding.

### Neutral

- Grafana provisioning is file-based (dashboards and data sources in git).
  Alert rules are also managed in Prometheus rule files, not the Grafana UI,
  to keep them version-controlled.

---

## References

- `adr/0001-four-docker-networks.md` — why monitoring lives in an isolated network
