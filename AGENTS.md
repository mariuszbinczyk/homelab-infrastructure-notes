# homelab-infrastructure-notes — Agent Guidelines

This is a **public documentation repository** extracted from a private homelab setup.

## Purpose

Portfolio and community resource demonstrating:
- Architecture Decision Records (ADRs) for a self-hosted infrastructure
- Operational documentation following the Diátaxis framework
- Container security and monitoring practices

## Repository structure

```
adr/          ADRs — architecture decisions (Context → Decision → Consequences)
docs/         Operational documentation (how-to + explanation)
README.md     Entry point with stack overview
AGENTS.md     This file
```

## Content rules (CRITICAL)

This is a **public** repository. Before adding or editing any content:

- **NO** real IP addresses, subnets, or hostnames (use `<HOST-IP>`, `<homelab-domain>` placeholders)
- **NO** personal names, email addresses, or chat IDs
- **NO** credentials, tokens, API keys, or secrets
- **NO** references to private internal file paths (e.g. `/opt/docker/...`)
- Generic container and tool names (traefik, prometheus, etc.) are fine

## Document metadata schema

Every document must start with:

```yaml
---
title: "..."
status: active | historical | superseded | draft
type: tutorial | how-to | reference | explanation | decision
owner: homelab
last_reviewed: YYYY-MM-DD
---
```

## ADR format

```
# ADR-NNNN: Short title
Date + Status
## Context  (3–6 sentences, factual)
## Decision (what was decided + mandatory conventions)
## Consequences (Positive / Negative / Neutral)
## References
```

Use `adr/0000-template.md` as starting point for new ADRs.

## Pre-publish checklist (run before every commit)

Scan every changed file for the following. Any hit = BLOCK, do not publish.

### Sensitive patterns to grep for

```bash
# Private paths
grep -rn "/opt/docker\|/srv/docker\|/mnt/jellyfin\|/mnt/backup\|/etc/systemd/system" --include="*.md"

# Private network topology
grep -rn "192\.168\.\|172\.\(1[6-9]\|2[0-9]\|3[01]\)\.\|10\.\|\.lan[^>-]" --include="*.md"

# Personal / credentials
grep -rn "@[a-z].*\.[a-z]\{2,\}\|password.*=\|token.*=\|secret.*=\|api.key\|chat_id\|PAT\b" --include="*.md"

# Personal names or hostnames (adjust to your context)
grep -rn "mariusz\|atlas\b\|tetes\.pl\|OpenNAS" --include="*.md"
```

Allowed exceptions (do NOT flag these):
- `<HOST-IP>`, `<homelab-domain>`, `<data-dir>`, `<backup-dir>`, `<media-dir>`, `<repo-root>` — these are explicit placeholders
- `alerts@example.org`, `192.0.2.x` — standard documentation placeholder patterns
- `password` / `secret` in generic YAML examples (e.g. `db_password:` in Docker Secrets how-to)
- Tool names: traefik, prometheus, grafana, loki, etc. — public open-source software

### Link consistency check

```bash
# Find markdown links pointing to non-existent local files
grep -rn "\](\.\./" --include="*.md" | sed "s/.*](\(\.\.\/[^)]*\)).*/\1/" | sort -u | while read p; do
  [ -f "$p" ] || echo "BROKEN: $p"
done
```

### Frontmatter check

Every `.md` file (except `AGENTS.md` and `README.md`) must have:

```yaml
---
title: "..."
status: active | historical | superseded | draft
type: tutorial | how-to | reference | explanation | decision
owner: homelab
last_reviewed: YYYY-MM-DD
---
```

## When reviewing

1. Run the pre-publish checklist above before flagging anything as ready
2. Verify cross-references between documents are consistent and not broken
3. Ensure frontmatter is present and correct on every document
4. ADRs should be concise — if a section exceeds 10 lines, consider trimming
5. Do NOT add new technical content — only sanitization and structural fixes
