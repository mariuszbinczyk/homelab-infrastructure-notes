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

## When reviewing

1. Check for sensitive data (IPs, emails, personal info, internal paths)
2. Verify cross-references between documents are consistent
3. Ensure frontmatter is present and correct on every document
4. ADRs should be concise — if a section exceeds 10 lines, consider trimming
