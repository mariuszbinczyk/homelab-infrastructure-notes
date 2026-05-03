---
title: "ADR-0005: Security Monitoring Stack — CrowdSec + Zeek + Falco"
status: active
type: decision
owner: homelab
last_reviewed: 2026-05-03
---

# ADR-0005: Security Monitoring Stack — CrowdSec + Zeek + Falco

**Date:** 2026-05-01
**Status:** Accepted

---

## Context

Homelab Docker (~40 containers) had a mature monitoring stack (Prometheus, Loki, Alertmanager → n8n → Dispatcher), but was missing a security layer:

- No WAF / reactive blocking for HTTP
- No passive network traffic visibility (NSM)
- No runtime security for containers and host
- No lateral movement or container drift detection

Additional context: **AdGuard is the DNS resolver for the entire LAN** (router points to AdGuard). This gives full DNS visibility for all devices (phones, laptops, IoT, smart home) through Zeek — increases NSM value but also DNS log volume (estimated 200–500 MB/day).

Previously evaluated but not deployed: fail2ban (no HTTP application layer), Suricata (signature IDS vs metadata NSM), Wazuh (too heavy, ~4 GB RAM budget).

## Decision

Deploy three open tools with integration into the existing alerts pipeline:

### 1. CrowdSec — reactive blocking + AppSec WAF

- **Engine**: `crowdsecurity/crowdsec:v1.6.4` in a dedicated container (LAPI)
- **Bouncer**: Traefik plugin (`maxlerebourg/crowdsec-bouncer-traefik-plugin v1.6.0`) — not sidecar (lower RAM, in-process, native `X-Real-Ip`)
- **AppSec WAF**: CrowdSec hub engine (virtual-patching + generic-rules collections) — blocks CVE exploits, path traversal, known payloads
- **Community Blocklist** (CAPI): dynamic malicious IP list, updated every few hours
- **Acquisition**: Docker socket for Traefik/Authentik + `/var/log/auth.log` (SSH host)
- **Whitelist**: specific admin IPs only (e.g., `ADMIN_DESKTOP/32`, `ADMIN_LAPTOP/32`) — NOT entire `/24` (compromised IoT devices must remain bannable)

### 2. Zeek — passive NSM (DNS-heavy)

- **Image**: `zeek/zeek:6.2.0` *(plan: `blacktop/zeek:lts`, changed — blacktop has no 6.2.0 tag)*
- **Network**: `network_mode: host`, `user: root` (libpcap requires root or CAP_NET_RAW)
- **Interface**: main Ethernet interface (passed via `-i <iface>` in `command:`)
- **Output**: JSON Lines in `/srv/docker/zeek/logs/*.log` (rotated to `*.2026-05-03-12-00-00.log`)
- **Internal rotation**: `redef Log::default_rotation_interval = 30mins;` in `local.zeek`
- **Retention**: 14 days (`.log.gz` after compression by sidecar `zeek-log-rotator`)
- **Logs to Loki**: Promtail job `zeek` — `*.log` pattern, JSON pipeline, Unix timestamp
- **Protocols**: conn, dns, http, ssl, ssh, notice, weird, x509 (enabled by default in 6.2.0)

### 3. Falco + Falcosidekick — runtime security

- **Image**: `falcosecurity/falco:0.39.2` *(plan: `falco-no-driver:0.39`, changed — deployed full image with modern_ebpf)*
- **eBPF config**: `engine.kind: modern_ebpf` in `falco.yaml` *(plan: `ebpf`, modern_ebpf is better for kernel 6.8+)*
- **Network**: `network_mode: host`, `pid: host` (visibility into all namespaces)
- **Falcosidekick**: `falcosecurity/falcosidekick:2.33.0` — outputs to Loki HTTP push + Prometheus metrics
- **Note**: Falco 0.39.2 does not expose `/metrics` on `:8765` (only `/healthz`) — monitoring via Falcosidekick
- **Key rules**: container drift, write below /etc, shell in container, sensitive file read, outbound to crypto miner pools
- **Local whitelist**: `falco/falco_rules.local.yaml` — specific containers whitelisted for false positives

### Alert Integration

All signals feed into the existing pipeline AM → n8n → Dispatcher:

- **Prometheus** (deployed): 6 rules in `homelab_security` group — `CrowdSecLAPIDown`, `CrowdSecHighBanRate`, `CrowdSecAppSecHighBlockRate`, `FalcosidekickDown`, `FalcosidekickOutputErrors`, `CrowdSecAppSecLatencyHigh`
- **Loki** (deployed): `monitoring/loki/rules/fake/security.yml` — `falco_security` (4 rules) + `crowdsec_security` (1 rule)
- **CrowdSec metrics**: prefix `cs_*` (not `crowdsec_*`) — `cs_active_decisions`, `cs_bucket_overflowed_total`, `cs_appsec_block_total`
- Alertmanager: no changes (existing routes and quiet hours)

## Consequences

### Positive

- Reactive blocking of HTTP attacks in real time
- Full DNS visibility for entire LAN (DGA, beaconing, IoT C2 detection)
- Runtime security with eBPF — zero performance overhead on containers
- Existing alert pipeline preserved with quiet hours and per-user routing
- Community blocklist — protection against known malicious IPs without configuration

### Costs and Risks

| Risk | Mitigation |
|---|---|
| +~2 GB RAM (CrowdSec 512M, Zeek 1G, Falco 384M, Sidekick 128M) | Comfortable margin on 32 GB system |
| Traefik restart during plugin install | Evening window, backup traefik.yml, `defaultDecision: bypass` |
| Admin self-ban | Whitelist specific admin IPs (not /24) |
| Falco false-positive on docker exec | `user_known_run_shell_activities` macro whitelist |
| DNS log volume from entire LAN | 30min rotation, 14d retention |
| Falco privileged + pid:host | Conscious trade-off; Tetragon alternative in Stage 6+ |
| Falco libcurl ~6h errors | Benign TCP keep-alive reset; Falco reconnects automatically |

## Alternatives Considered

- **Fail2ban**: no HTTP application layer, regex on logs only — rejected
- **Suricata**: signature IDS (vs Zeek metadata NSM) — kept as Stage 6+ supplement
- **ModSecurity**: standalone WAF hard to integrate with Traefik plugin model — rejected
- **Wazuh**: SIEM too heavy (~4 GB RAM for full stack) — rejected, possibly Stage 7+
- **Dedicated security network**: unnecessary — Zeek/Falco use host network, CrowdSec on existing networks

## Implementation

Deployment in 6 stages:

| Stage | Description | Status |
|-------|-------------|--------|
| 1 | CrowdSec engine + Prometheus scrape | ✅ Done |
| 2 | Traefik plugin + AppSec WAF | ✅ Done |
| 3 | Falco + Falcosidekick | ✅ Done |
| 4 | Zeek (host network, DNS-heavy) | ✅ Done |
| 5 | Prometheus/Loki alert rules | ✅ Done — 6 Prometheus + 5 Loki rules |
| 6 | Docs, ADR polish | ✅ Done |

---

## Amendment — Zeek Startup Pitfalls (2026-05-03)

Deploying Zeek 6.2.0 in standalone mode required debugging five non-obvious issues:

**1. Missing `command:` in docker-compose.yml**
The default entrypoint of `zeek/zeek:6.2.0` is `bash` (no arguments) — the container exited immediately and was restarted by `unless-stopped`. Fix: add `command: bash -c "zeek -i <iface> -C local"`.

**2. Wrong installation path `/opt/zeek` (actual: `/usr/local/zeek`)**
The assumed install path was `/opt/zeek/share/zeek/site/`, while the image uses `/usr/local/zeek/share/zeek/site/`. A volume mount to `/opt/zeek/...` did not override the system file — the stock `local.zeek` was loaded instead (no JSON output, no rotation).

**3. Non-existent policies in Zeek 6.2.0**
- `policy/protocols/dns/log-dns` — does not exist; DNS logging is built-in by default
- `policy/protocols/http/entities` — moved to `base/protocols/http/entities`
- `@load base/frameworks/logging` — redundant in non-bare mode (auto-loaded)

**4. Log rotation sidecar bug**
A sidecar `zeek-log-rotator` was searching for `*.json` (wrong glob) and compressing actively-written files (potentially corrupting logs). Correct behavior:
- `redef Log::default_rotation_interval = 30mins;` in `local.zeek` → Zeek creates `conn.2026-05-03-12-00-00.log` etc.
- Sidecar should only compress timestamped files (`*.20[0-9][0-9]-*.log`), not active `conn.log`

**5. Log rotation disabled by default in standalone mode**
Default `Log::default_rotation_interval = 0secs` (disabled). Without this setting, `dns.log` grows ~200 MB/day unbounded. Set in `local.zeek`: `redef Log::default_rotation_interval = 30mins;`.
