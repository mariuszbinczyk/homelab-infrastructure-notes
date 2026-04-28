---
title: "Alerting Strategy — Dual-Layer Model"
status: active
type: explanation
owner: homelab
last_reviewed: 2025-11-26
---

# Homelab Alerting Strategy

**Version:** 1.0
**Last Updated:** 2025-11-26
**Status:** Production

---

## 🎯 Overview

This homelab uses **dual alerting systems** working in parallel:
1. **Uptime-Kuma** - HTTP/TCP availability monitoring
2. **Prometheus + Alertmanager + n8n AI** - Metrics-based monitoring with AI classification

Both systems complement each other and provide comprehensive coverage.

---

## 📊 System Architecture

### **System 1: Uptime-Kuma (Simple Uptime)**

```
Uptime-Kuma → ntfy (direct)
```

**Purpose:** Fast detection of service unavailability

**What it monitors:**
- HTTP endpoints (200 OK checks)
- TCP port availability
- Response time monitoring

**Notification channels:**
- ntfy push notifications (configured in Uptime-Kuma UI)
- Prefix: `[Uptime-Kuma]`

**Use case:** "Is the service alive?"

**Advantages:**
- ✅ Extremely fast detection (seconds)
- ✅ Simple setup
- ✅ Immediate notifications

**Disadvantages:**
- ❌ No context (just UP/DOWN)
- ❌ No intelligent filtering
- ❌ Can be noisy (temporary network blips)

---

### **System 2: Prometheus + AI (Intelligent Monitoring)**

```
Prometheus → Alertmanager → n8n AI Workflow → ntfy + Email
                    ↓
                  Email (critical alerts)
```

**Purpose:** Deep metrics analysis with AI-enhanced context

**What it monitors:**
- System metrics (CPU, RAM, Disk)
- Container metrics (via cAdvisor)
- Service performance (query time, latency)
- Custom exporters (AdGuard, backups)

**AI Workflow (n8n):**
1. **Webhook** - Receives alert from Alertmanager
2. **Parse Alertmanager JSON** - Extracts alert fields
3. **Prepare AI Prompt** - Builds classification prompt
4. **Call Ollama API** - llama3.2:3b (2GB model) classifies alert
5. **Parse AI Response** - Extracts classification & enhanced message
6. **Send to ntfy** - Publishes AI-enhanced notification

**AI Classification Levels:**
- `CRITICAL` - Service down, data loss (immediate action)
- `HIGH` - Performance degraded, potential impact
- `MEDIUM` - Warning, non-urgent issue
- `LOW` - Informational, no action needed
- `SUPPRESS` - Noise, test alerts (can be filtered)

**Notification channels:**
- **ntfy** - All alerts (AI-enhanced)
- **Email** - Critical alerts + warnings

**Use case:** "Why does the service have a problem and what should I do?"

**Advantages:**
- ✅ Intelligent context and root cause hints
- ✅ Suggested actions
- ✅ Noise reduction via AI classification
- ✅ Rich metrics data

**Disadvantages:**
- ❌ Slower (AI inference ~15-20 seconds)
- ❌ More complex setup
- ❌ Requires Ollama resources

---

## 🔄 How They Work Together

### **Example: AdGuard DNS Issue**

**Uptime-Kuma detects:**
```
🔴 AdGuard DNS Down [Uptime-Kuma]
Request timeout
```
↳ Fast, simple, immediate

**Prometheus + AI analyzes:**
```
🚨 [HIGH] AdGuardDNSQueriesSlow
Instance: adguard:53

DNS query time >500ms. Likely upstream resolver issue or rate limiting.

Action: Review AdGuard query logs and check upstream DNS connectivity.

---
Original severity: warning
AI reasoning: Performance degradation indicates upstream issues.
Not CRITICAL as DNS still responds, but needs prompt investigation.
```
↳ Detailed, contextualized, actionable

### **Complementary Coverage:**

| Scenario | Uptime-Kuma | Prometheus AI |
|----------|-------------|---------------|
| Service completely down | ✅ Immediate alert | ✅ Context + impact |
| Slow performance | ❌ Not detected | ✅ Detected + analyzed |
| Temporary network blip | ⚠️ False positive | ⚠️ May classify as SUPPRESS |
| Memory leak building up | ❌ Not detected | ✅ Early warning |

---

## 📝 Alertmanager Routing Strategy

**File:** `<repo-root>/monitoring/alertmanager/config/alertmanager.yml`

### **Routing Logic:**

```yaml
route:
  receiver: 'homelab-email'  # Default

  routes:
    # All alerts go to n8n AI first
    - receiver: 'n8n-webhook'
      continue: true  # Also send to other receivers

    # Critical alerts - email + n8n
    - match:
        severity: critical
      receiver: 'homelab-email-critical'
      group_wait: 5s
      repeat_interval: 1h

    # Warning alerts - email + n8n
    - match:
        severity: warning
      receiver: 'homelab-email'
```

### **Receivers:**

1. **n8n-webhook** (`http://n8n:5678/webhook/alertmanager`)
   - All alerts
   - AI processing
   - Sends to ntfy with enhanced context

2. **homelab-email** (`alerts@example.org`)
   - Warning + info alerts
   - Subject: `[Homelab] {severity}: {alertname}`

3. **homelab-email-critical** (`alerts@example.org`)
   - Critical alerts only
   - Subject: `🚨 [CRITICAL] {alertname} - Immediate Action Required`
   - Faster grouping (5s vs 30s)

---

## 🤖 AI Features (Ollama llama3.2:3b)

### **Why llama3.2:3b?**
- ✅ Small (2GB) - fits in 7.5GB RAM system
- ✅ Fast inference (~15-20s per alert)
- ✅ Good at classification tasks
- ✅ Understands homelab context

### **AI Prompt Structure:**

```
You are a homelab alert classifier.

ALERT: Name={alertname}, Severity={severity}, Instance={instance},
Summary={summary}, Description={description}

TASK:
1) Classify urgency: CRITICAL, HIGH, MEDIUM, LOW, SUPPRESS
2) Provide brief enhanced message with root cause and action
3) Keep response concise (2-3 sentences)

RESPOND JSON: {classification:text, enhanced_message:text, reasoning:text}
```

### **AI Output Example:**

**Input:** ContainerMemoryHigh, warning, jellyfin using 90% RAM

**AI Response:**
```json
{
  "classification": "MEDIUM",
  "enhanced_message": "Container 'jellyfin' is using 90% of its allocated memory limit (1.8GB out of 2GB), which could indicate a potential memory leak or performance issue. Consider increasing the container's memory limit to ensure stability.",
  "reasoning": "Memory usage at 90% indicates significant allocation, but within allowed limit. If left unchecked, may lead to performance issues. Increasing memory limit can help mitigate this concern."
}
```

---

## 🔕 Noise Reduction Strategy

### **Inhibition Rules** (Alertmanager)

Prevent duplicate/related alerts:

```yaml
inhibit_rules:
  # If service is down, don't alert about high CPU/memory
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      alertname: 'HighCPUUsage|HighMemoryUsage'
    equal: ['instance']

  # If critical disk space, suppress warning
  - source_match:
      alertname: 'SrvDiskSpaceCritical'
    target_match:
      alertname: 'SrvDiskSpaceWarning'
    equal: ['mountpoint']
```

### **AI Classification** (n8n)

AI can classify alerts as `SUPPRESS` for:
- Test alerts
- Flapping services
- Known non-issues
- Temporary conditions

*(Currently not filtered, but can add IF node to suppress)*

### **Grouping** (Alertmanager)

```yaml
group_by: ['alertname', 'severity', 'category']
group_wait: 30s
group_interval: 5m
repeat_interval: 4h
```

Reduces notification spam by batching related alerts.

---

## 📱 Notification Format

### **ntfy Message Structure:**

```
🚨 [{CLASSIFICATION}] {alertname}
📍 Instance: {instance}
📌 Message: {AI-enhanced message with context and action}

---
🔹 Original severity: {severity}
🤖 AI reasoning: {why this classification}
```

### **Example:**

```
🚨 [HIGH] BackupJobDelayed
📍 Instance: backup-server:22
📌 Message: Daily backup job is delayed by 36 hours and last successful backup was on 2025-11-25. Check cron schedule and disk space to resolve the issue.

---
🔹 Original severity: warning
🤖 AI reasoning: Job delay is a warning condition, indicating potential data loss or system impact. However, it's not critical as no immediate action is required, but prompt investigation and resolution are necessary to prevent further delays.
```

---

## 🔧 Configuration Files

| Component | Config File | Purpose |
|-----------|-------------|---------|
| Prometheus | `<repo-root>/monitoring/prometheus/config/prometheus.yml` | Scrape targets, rules |
| Alertmanager | `<repo-root>/monitoring/alertmanager/config/alertmanager.yml` | Routing, receivers |
| n8n Workflow | n8n web UI | AI processing workflow |
| Uptime-Kuma | SQLite DB (`<data-dir>/uptime-kuma/data/kuma.db`) | Monitors, notifications |

---

## 📊 Monitoring the Monitors

**Grafana Dashboard:** http://grafana.<homelab-domain>
- Prometheus targets health
- Alertmanager stats
- n8n workflow executions (manual check)
- Uptime-Kuma status (UI only)

**Self-monitoring alerts:**
- PrometheusDown
- AlertmanagerDown
- ServiceDown (includes Uptime-Kuma)

---

## 🚀 Future Enhancements (TODOs)

### **1. Daily AI Health Report Workflow**
- Scheduled n8n workflow (cron: daily 08:00)
- Ollama analyzes logs from all services
- Generates summary: issues, trends, recommendations
- Sends digest to ntfy/email
- **Status:** TODO - Next session priority

### **2. SUPPRESS Alert Filtering**
- Add IF node in n8n workflow
- Don't send alerts classified as SUPPRESS
- Log suppressed alerts for review
- **Status:** Optional - can add if too much noise

### **3. Uptime-Kuma → n8n Integration**
- Send Uptime-Kuma alerts to n8n webhook
- AI context for uptime alerts
- Unified notification format
- **Status:** Nice-to-have

### **4. Polish Language Support**
- Test if llama3.2:3b can respond in Polish
- Update prompt: "Respond in Polish"
- **Status:** Optional - English works fine

---

## Related Documentation

- [ADR-0004: Hybrid Monitoring Stack](../adr/0004-hybrid-monitoring-stack.md)


**Maintained by:** homelab operator
**Review Schedule:** Monthly or after major changes
