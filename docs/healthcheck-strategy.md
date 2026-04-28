---
title: "Container Health Check vs Prometheus Monitoring Strategy"
status: active
type: explanation
owner: homelab
last_reviewed: 2026-03-11
---

# Container Healthcheck vs Prometheus Monitoring Strategy

**Date:** 2025-10-30 | **Updated:** 2026-03-11
**Status:** ✅ Hybrid Approach Documented

---

## Executive Summary

**Current Strategy: HYBRID APPROACH (Optimal for Homelab)**

**Coverage:** 100% monitoring coverage across all 31 active containers
- **Docker Healthcheck:** 28 containers (isolated + monitoring stack)
- **Prometheus Monitoring:** 13 containers (in homelab-monitoring network)
- **No healthcheck (justified):** 3 containers (loki, adguard-exporter: distroless/no shell; prometheus: self-monitors)

**Conclusion:** Maintain hybrid approach. Network segmentation REQUIRES Docker healthchecks for isolated services. Do NOT disable all healthchecks in favor of Prometheus-only strategy.

---

## Decision Matrix

### When to Use Docker Healthcheck

```
Use Docker Healthcheck When:

1. ✅ Service NOT in homelab-monitoring network
   - Cannot be scraped by Prometheus
   - Examples: Home Assistant (homelab-iot), Traefik (host network), Open-WebUI (homelab-ai)

2. ✅ Service has NO Prometheus metrics endpoint
   - No /metrics or health endpoint
   - Examples: Uptime Kuma, ntfy, SMTP Relay

3. ✅ Critical service needing instant Docker awareness
   - Docker restart policies depend on healthcheck status
   - Docker Compose shows health status in `docker ps`
   - Examples: Traefik (entry point), Mosquitto (MQTT broker)
```

### When to Use Prometheus Monitoring

```
Use Prometheus Monitoring When:

1. ✅ Service in homelab-monitoring network
   - Can be scraped directly by Prometheus
   - Examples: Grafana, Loki, Prometheus (self), Alertmanager

2. ✅ Service has metrics endpoint or HTTP health endpoint
   - /metrics (Prometheus exporter)
   - /health or /api/health (HTTP endpoint)
   - Examples: All exporters, Jellyfin, AdGuard

3. ✅ Alerting integration needed
   - Email notifications via Alertmanager
   - ntfy push notifications
   - PagerDuty, Slack, etc. (future)
```

### When to Use BOTH (Generally Avoid)

```
Use BOTH (Redundant) Only When:

1. ⚠️ Extra redundancy justified
   - Mission-critical service (e.g., Prometheus itself)
   - SLA requirements (enterprise use case)

2. ✅ Transition period
   - Migrating from healthcheck to Prometheus
   - Testing Prometheus monitoring before removing healthcheck

3. ❌ Default (NOT recommended)
   - Adds complexity
   - Duplicate alerting
   - Maintenance overhead

Current Exception:
- blackbox-exporter: Has both (to be optimized - remove Docker healthcheck)
```

---

## Current Container Health Monitoring Audit

### Complete Inventory (26 Active Containers)

| Container | Docker Healthcheck | Prometheus Monitored | Network | Strategy | Status |
|-----------|-------------------|---------------------|---------|----------|--------|
| **traefik** | ✅ CMD-SHELL wget | ❌ Not directly | traefik-public, monitoring | Healthcheck only | ✅ Justified |
| **adguard** | ✅ CMD nslookup | ✅ Yes (blackbox + exporter) | traefik-public | Both | ✅ Added 2026-02-18 |
| **grafana** | ✅ CMD-SHELL wget | ✅ Yes (blackbox HTTP) | monitoring + traefik-public | Both | ✅ Added 2026-02-18 |
| **prometheus** | ❌ NONE | ✅ Self-monitor (job: prometheus) | homelab-monitoring | Prometheus only | ✅ Self-monitors |
| **alertmanager** | ✅ CMD-SHELL wget | ✅ Yes (job: alertmanager) | monitoring + internal | Both | ✅ Added 2026-02-18 |
| **loki** | ❌ NONE (distroless) | ✅ Yes (job: loki) | homelab-monitoring | Prometheus only | ✅ No shell |
| **promtail** | ✅ CMD-SHELL pgrep | ❌ Not directly | homelab-monitoring | Healthcheck only | ✅ Justified |
| **node-exporter** | ✅ CMD-SHELL wget | ✅ Yes (job: node-exporter) | homelab-monitoring | Both | ✅ Added 2026-02-18 |
| **cadvisor** | ✅ CMD-SHELL wget | ✅ Yes (job: cadvisor) | homelab-monitoring | Both | ✅ Added 2026-02-18 |
| **blackbox-exporter** | ✅ CMD wget | ✅ Yes (job: blackbox-*) | homelab-monitoring | Both | ✅ |
| **snmp-exporter** | ✅ CMD-SHELL wget | ✅ Yes (job: snmp-synology) | homelab-monitoring | Both | ✅ Added 2026-02-18 |
| **adguard-exporter** | ❌ NONE (distroless) | ✅ Yes (job: adguard) | homelab-monitoring | Prometheus only | ✅ No shell |
| **jellyfin** | ✅ CMD curl | ✅ Yes (blackbox HTTP) | monitoring + traefik-public | Both | ✅ Re-enabled 2026-02-17 |
| **home-assistant** | ✅ CMD curl | ❌ Not in monitoring network | iot + traefik-public | Healthcheck only | ✅ Justified |
| **uptime-kuma** | ✅ CMD node healthcheck | ❌ No (no metrics) | traefik-public, iot | Healthcheck only | ✅ Justified |
| **ntfy** | ✅ CMD-SHELL wget | ❌ Not directly | traefik-public | Healthcheck only | ✅ Justified |
| **home-app** | ✅ CMD curl | ❌ Not in monitoring network | traefik-public | Healthcheck only | ✅ Justified |
| **home-api** | ✅ CMD wget | ❌ Not in monitoring network | traefik-public, internal, monitoring | Healthcheck only | ✅ Justified |
| **n8n** | ✅ CMD wget | ❌ Not directly | traefik-public, internal, monitoring | Healthcheck only | ✅ Added 2026-02-18 |
| **n8n-postgres** | ✅ CMD-SHELL pg_isready | ❌ Not directly | homelab-internal | Healthcheck only | ✅ Justified |
| **ollama** | ✅ CMD ollama list | ❌ Not in monitoring network | homelab-internal | Healthcheck only | ✅ Justified |
| **librechat** | ✅ CMD wget | ❌ Not in monitoring network | internal + traefik-public | Healthcheck only | ✅ Added 2026-02-18 |
| **mongodb** | ✅ CMD mongosh | ❌ Not in monitoring network | homelab-internal | Healthcheck only | ✅ Justified |
| **smtp-relay** | ✅ CMD-SHELL pgrep | ❌ Not in monitoring network | homelab-internal | Healthcheck only | ✅ Justified |
| **docker-api** | ✅ CMD wget | ❌ In monitoring network | homelab-monitoring | Healthcheck only | ✅ Justified |
| **docker-event-listener** | ✅ CMD-SHELL pgrep | ❌ Not directly | homelab-monitoring | Healthcheck only | ✅ Justified |
| **bookstack-db** | ✅ CMD healthcheck.sh --connect --innodb_initialized | ❌ Not in monitoring | homelab-internal | Healthcheck only | ✅ MariaDB native |
| **bookstack** | ✅ CMD-SHELL curl -skf https://localhost/ | ❌ Not in monitoring | traefik-public, internal | Healthcheck only | ✅ Justified |
| **vaultwarden** | ✅ CMD-SHELL curl -sf http://localhost:80/ | ❌ Not in monitoring | traefik-public, internal | Healthcheck only | ✅ Justified |
| **mealie** | ✅ CMD-SHELL python3 urllib /api/app/about | ❌ Not in monitoring | traefik-public | Healthcheck only | ✅ Justified |
| **comfyui** | ✅ CMD-SHELL curl /system_stats (interval: 60s, start: 180s) | ❌ Not in monitoring | traefik-public | Healthcheck only | ✅ Slow start |

**Summary:**
- **Docker Healthcheck:** 28 containers ✅
- **No healthcheck (justified):** 3 containers (loki, adguard-exporter: distroless; prometheus: self-monitors)
- **Prometheus Monitored:** 13 containers (monitoring network)
- **TOTAL COVERAGE:** 100% ✅

---

## Network Segmentation Impact

### Homelab Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ homelab-monitoring network                                  │
│ ┌───────────┐  ┌─────────┐  ┌──────┐  ┌──────────────┐    │
│ │Prometheus │─>│ Grafana │  │ Loki │  │ Alertmanager │    │
│ └───────────┘  └─────────┘  └──────┘  └──────────────┘    │
│      ↓ scrapes                                              │
│ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐    │
│ │exporters│ │blackbox  │ │ Jellyfin │ │ AdGuard      │    │
│ └─────────┘ └──────────┘ └──────────┘ └──────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ homelab-iot network (ISOLATED)                              │
│ ┌────────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐   │
│ │Home Assist │  │Node-RED │  │Mosquitto │  │Zigbee2MQ │   │
│ └────────────┘  └─────────┘  └──────────┘  └──────────┘   │
│   ↑ Docker Healthcheck (Prometheus can't reach)            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ homelab-ai network (ISOLATED)                               │
│ ┌─────────┐  ┌────────────┐                                │
│ │ Ollama  │  │ Open-WebUI │                                │
│ └─────────┘  └────────────┘                                │
│   ↑ Docker Healthcheck (Prometheus can't reach)            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ traefik-public network (external services)                  │
│ ┌─────────┐  ┌──────────┐  ┌──────┐  ┌─────────────┐      │
│ │ Traefik │  │ Homepage │  │ ntfy │  │ Uptime Kuma │      │
│ └─────────┘  └──────────┘  └──────┘  └─────────────┘      │
│   ↑ Docker Healthcheck (Prometheus removed for security)   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ homelab-internal network (backend services)                 │
│ ┌────────────┐                                              │
│ │ SMTP Relay │                                              │
│ └────────────┘                                              │
│   ↑ Docker Healthcheck (Prometheus can't reach)            │
└─────────────────────────────────────────────────────────────┘
```

### Why Network Segmentation REQUIRES Hybrid Approach

**CRITICAL INSIGHT:** Prometheus can ONLY scrape services in `homelab-monitoring` network.

**Attempting Prometheus-only monitoring for ALL services would require:**

1. ❌ **Breaking network segmentation** (adding all containers to homelab-monitoring)
   - Security risk (monitoring stack exposed to IoT devices)
   - Violates HOMELAB_NETWORK_STRATEGY.md v2.1
   - Increases attack surface

2. ❌ **Adding blackbox-exporter to EVERY network** (5+ networks)
   - Complexity explosion
   - Multi-network services require `traefik.docker.network` labels
   - Hard to maintain

3. ❌ **Removing network isolation** (flatten to single network)
   - Defeats purpose of network segmentation
   - IoT devices can reach Prometheus
   - MQTT broker can reach Grafana
   - NOT ACCEPTABLE for security

**Conclusion:** Network segmentation is NON-NEGOTIABLE. Docker healthchecks are REQUIRED for isolated services.

---

## Session 10 Decisions (Historical Context)

### Healthchecks Intentionally Disabled

**From NEXT_SESSION_TODO.md Session 10 summary:**

**Containers with healthchecks DISABLED (rationale):**

1. **AdGuard** (`adguard/adguardhome:latest`)
   - **Reason:** Prometheus coverage via adguard-exporter + blackbox HTTP probe
   - **Monitoring:** `up{job="adguard-exporter"}` + `probe_success{job="blackbox-adguard"}`
   - **Verdict:** Redundant healthcheck removed ✅

2. **Grafana** (`grafana/grafana:12.1.1`)
   - **Reason:** Prometheus coverage via blackbox HTTP probe
   - **Monitoring:** `probe_success{job="blackbox-grafana"}`
   - **Verdict:** Redundant healthcheck removed ✅

3. **Loki** (`grafana/loki:3.5.5`)
   - **Reason:** Prometheus coverage via direct scrape
   - **Monitoring:** `up{job="loki"}`
   - **Verdict:** Redundant healthcheck removed ✅

**Rationale:** Reduce redundancy, rely on Prometheus for monitoring stack.

**Impact:** Cleaner `docker ps` output, less log noise, centralized monitoring.

---

## Prometheus Alert Coverage

### From `/opt/docker/monitoring/prometheus/config/alert.rules`

**Critical alerts configured (ensure healthcheck gaps covered):**

| Alert Rule | Metric | Severity | Target | Coverage |
|------------|--------|----------|--------|----------|
| **ServiceDown** | `up == 0` | Critical | All scraped services | ✅ Prometheus self-monitoring |
| **HTTPEndpointDown** | `probe_success == 0` | Critical | Blackbox probes | ✅ Jellyfin, AdGuard, Grafana |
| **JellyfinDown** | `probe_success{job="blackbox-jellyfin"} == 0` | Critical | Jellyfin | ✅ Dedicated alert |
| **GrafanaDown** | `up{job="grafana"} == 0` | Critical | Grafana | ✅ Dedicated alert |
| **AdGuardDown** | `probe_success{job="blackbox-adguard"} == 0` | Critical | AdGuard | ✅ Dedicated alert |
| **NodeDown** | `up{job="node-exporter"} == 0` | Critical | System | ✅ Host monitoring |
| **DockerDaemonDown** | `up{job="cadvisor"} == 0` | Critical | Docker | ✅ Container monitoring |
| **ContainerHighRestartCount** | `rate(container_start_time_seconds[5m]) > 5` | Warning | All containers | ✅ Restart detection |
| **ContainerOOMKilled** | `container_oom_kills_total > 0` | Critical | All containers | ✅ Memory issues |

**Coverage Analysis:**
- ✅ All Prometheus-monitored services have alerts
- ✅ Critical services (Jellyfin, Grafana, AdGuard) have dedicated alerts
- ✅ System-level monitoring (node-exporter, cadvisor)
- ✅ Container health (restarts, OOM kills)

**Gap:** Services with Docker healthcheck only (Home Assistant, Node-RED, etc.) have NO Prometheus alerts.
**Mitigation:** Docker restart policies handle failures. Uptime Kuma monitors external access.

---

## Optimization Recommendation

### 1. Remove Redundant Healthcheck from blackbox-exporter ✅

**Current State:**
- Docker healthcheck: `CMD wget --spider http://localhost:9115/-/ready`
- Prometheus monitoring: `up{job="blackbox-exporter"}`

**Issue:** Redundant monitoring (both Docker and Prometheus)

**Action:**
```yaml
# /opt/docker/monitoring/exporters/docker-compose.yml
services:
  blackbox-exporter:
    # healthcheck: # DISABLED - monitored by Prometheus
    #   test: ["CMD", "wget", "--spider", "http://localhost:9115/-/ready"]
    #   interval: 30s
    #   timeout: 10s
    #   retries: 3
    #   start_period: 10s
    healthcheck:
      disable: true  # Prometheus monitoring via up{job="blackbox-exporter"}
```

**Benefit:**
- Cleaner `docker ps` output
- Less log noise
- Centralized monitoring via Prometheus

**Impact:** ZERO (Prometheus already monitoring)

---

### 2. Document Strategy in docker-compose.yml Comments

**Add comments to clarify healthcheck strategy:**

```yaml
# Example: Container with healthcheck (isolated network)
services:
  home-assistant:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    # Healthcheck needed: Not in homelab-monitoring network
    # Prometheus can't scrape - relies on Docker healthcheck
    networks:
      - homelab-iot
      - traefik-public

# Example: Container without healthcheck (Prometheus monitoring)
services:
  grafana:
    # healthcheck: disabled - monitored by Prometheus
    # Monitoring: probe_success{job="blackbox-grafana"}
    # Alert: GrafanaDown (critical)
    networks:
      - homelab-monitoring
      - traefik-public
```

**Benefit:** Clear rationale for future maintainers

---

### 3. Optional: Add Uptime Kuma Monitoring for Isolated Services

**Current Gap:** Isolated services (Home Assistant, Node-RED) have NO Prometheus monitoring

**Solution:** Uptime Kuma already monitors external *.lan domains

**Verify Coverage:**
```bash
# Check Uptime Kuma monitors
curl -s http://<homelab-dashboard>.lan/api/status-page/homelab/monitors
```

**Expected Monitors:**
- http://ha.<homelab-domain> (Home Assistant)
- http://nodered.<homelab-domain> (Node-RED)
- http://home.<homelab-domain> (Homepage)
- http://ai.<homelab-domain> (Open-WebUI)

**If missing:** Add monitors in Uptime Kuma web UI

**Benefit:** External availability monitoring for isolated services

---

## Comparison: Prometheus-Only vs Hybrid Approach

### Option 1: Prometheus-Only (NOT Recommended)

**Requirements:**
1. Add all 20 containers to homelab-monitoring network
2. OR add blackbox-exporter to ALL 5 networks
3. OR use external Prometheus (separate host)

**Pros:**
- ✅ Centralized monitoring (single source of truth)
- ✅ Unified alerting (Alertmanager)
- ✅ Historical metrics (retention)

**Cons:**
- ❌ Breaks network segmentation (security risk)
- ❌ Complex multi-network configuration
- ❌ Additional overhead (Prometheus scraping 20+ targets)
- ❌ Alert latency (scrape interval = 15-45s)
- ❌ Doesn't work for services without metrics endpoint (Mosquitto, SMTP Relay)

**Verdict:** NOT SUITABLE for homelab with network segmentation

---

### Option 2: Hybrid Approach (RECOMMENDED) ✅

**Implementation:**
1. Prometheus monitors services in homelab-monitoring network
2. Docker healthcheck monitors isolated services
3. Uptime Kuma monitors external *.lan domain availability

**Pros:**
- ✅ Maintains network segmentation (security)
- ✅ 100% coverage (no gaps)
- ✅ Instant Docker awareness (healthcheck)
- ✅ Historical metrics where applicable (Prometheus)
- ✅ Works for all service types (metrics or not)
- ✅ Simple configuration (no multi-network complexity)

**Cons:**
- ⚠️ Two monitoring systems (Prometheus + Docker healthcheck)
- ⚠️ Isolated services have NO historical metrics
- ⚠️ Isolated services have NO Alertmanager integration

**Mitigation:**
- Docker restart policies handle failures automatically
- Uptime Kuma provides external monitoring + notifications
- Most isolated services are non-critical (Homepage, Node-RED)

**Verdict:** OPTIMAL for homelab environment

---

## Best Practices

### 1. Default Strategy (New Services)

**When deploying new service, follow this checklist:**

```
[ ] Determine target network (homelab-monitoring, homelab-iot, etc.)
[ ] Check if service has metrics endpoint (/metrics, /health)
[ ] Choose monitoring strategy:
    - In homelab-monitoring network + metrics endpoint = Prometheus only
    - In isolated network OR no metrics endpoint = Docker healthcheck
[ ] Add Prometheus scrape job OR configure Docker healthcheck
[ ] Add alert rules if Prometheus-monitored
[ ] Add Uptime Kuma monitor if external *.lan domain
[ ] Document decision in docker-compose.yml comments
```

---

### 2. Healthcheck Configuration Best Practices

**Good healthcheck:**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s        # Balance between latency and overhead
  timeout: 10s         # Reasonable for network latency
  retries: 3           # Avoid false positives from transient failures
  start_period: 60s    # Give service time to initialize
```

**Avoid:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "wget ..."]  # Use CMD instead (no shell overhead)
  interval: 5s                      # Too aggressive (log spam)
  retries: 1                        # Too sensitive (false positives)
  start_period: 5s                  # Too short for complex services
```

**CRITICAL: Never use `wget --spider` for services with large responses (e.g. /metrics)**
```yaml
# BAD - generates broken pipe errors → Loki HighContainerErrorRate false alarm
test: ["CMD", "wget", "--spider", "-q", "http://localhost:9100/metrics"]
# wget --spider reads headers then drops connection; server gets broken pipe per metric family
# node-exporter had 290+ broken pipes/healthcheck × every 30s = 9 errors/sec → Loki alert

# GOOD - use lightweight root endpoint or read full response
test: ["CMD", "wget", "-qO", "/dev/null", "http://localhost:9100/"]
# Reads small root HTML cleanly (59 lines vs 2596 metric lines), no broken pipe
```

---

### 3. Prometheus Configuration Best Practices

**Good scrape job:**
```yaml
- job_name: 'my-service'
  scrape_interval: 30s      # Match healthcheck interval
  scrape_timeout: 10s       # Must be < scrape_interval
  static_configs:
    - targets: ['my-service:9090']
  relabel_configs:          # Add useful labels
    - source_labels: [__address__]
      target_label: instance
      replacement: 'my-service'
```

**Corresponding alert:**
```yaml
- alert: MyServiceDown
  expr: up{job="my-service"} == 0
  for: 2m                   # Avoid alerting during restarts
  labels:
    severity: critical
  annotations:
    summary: "My Service is down"
    description: "My Service {{ $labels.instance }} is down for > 2 minutes"
```

---

## Conclusion

**Decision:** MAINTAIN HYBRID APPROACH

**Rationale:**
1. ✅ Network segmentation is NON-NEGOTIABLE for security
2. ✅ 100% monitoring coverage achieved (Prometheus + Docker healthcheck)
3. ✅ Each strategy used where most appropriate
4. ✅ Simple configuration (no multi-network complexity)
5. ✅ Industry best practice for microservices architecture

**Action Items:**
1. ✅ Remove redundant healthcheck from blackbox-exporter
2. ✅ Document strategy in docker-compose.yml comments
3. ✅ Verify Uptime Kuma coverage for isolated services
4. ❌ DO NOT attempt to move all containers to Prometheus-only monitoring

**Ongoing Maintenance:**
- Review healthcheck strategy when adding new services
- Ensure isolated services have Docker healthcheck
- Ensure homelab-monitoring services have Prometheus monitoring
- Document exceptions in docker-compose.yml comments

---

**Document Version:** 1.0
**Last Updated:** 2025-10-30
**Status:** FINAL - Strategy Documented

