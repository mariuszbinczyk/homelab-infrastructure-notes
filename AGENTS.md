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

Use the Python script below — it resolves every relative link **relative to the file that contains it**, not from the repository root. Links in `adr/` files that reference `docs/` files are resolved correctly.

```python
#!/usr/bin/env python3
"""Check local markdown links relative to their source file.
Run from the repository root: python3 check_links.py
Skips fenced code blocks so example paths inside ``` do not trigger false positives.
"""
import os, re, sys

REPO_ROOT = os.path.dirname(os.path.abspath(__file__))
LINK_RE = re.compile(r'\[(?:[^\]]*)\]\(((?:\.\.?/)[^)#\s]+)\)')
TICK_RE = re.compile(r'`((?:\.\.?/)[^`\s]+)`')

errors = []
for dirpath, dirnames, filenames in os.walk(REPO_ROOT):
    dirnames[:] = [d for d in dirnames if d != '.git']
    for filename in filenames:
        if not filename.endswith('.md'):
            continue
        filepath = os.path.join(dirpath, filename)
        in_fence = False
        with open(filepath, encoding='utf-8') as f:
            for lineno, line in enumerate(f, 1):
                if line.startswith('```'):
                    in_fence = not in_fence
                    continue
                if in_fence:
                    continue
                for pattern in (LINK_RE, TICK_RE):
                    for m in pattern.finditer(line):
                        ref = m.group(1)
                        resolved = os.path.normpath(
                            os.path.join(os.path.dirname(filepath), ref)
                        )
                        if not os.path.exists(resolved):
                            src = os.path.relpath(filepath, REPO_ROOT)
                            errors.append(f"{src}:{lineno}: BROKEN → {ref}")

if errors:
    print('\n'.join(errors))
    sys.exit(1)
print(f"All local links OK ({REPO_ROOT})")
```

> **Note on the template:** `adr/0000-template.md` uses placeholder text like `` `NNNN-superseding-adr.md` `` (backtick, not a link) intentionally — the link checker will not flag it.

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
