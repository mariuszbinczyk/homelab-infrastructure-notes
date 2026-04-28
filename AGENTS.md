# homelab-infrastructure-notes — Agent Guidelines

Personal homelab notes, public. ADRs only — no internal runbooks.

## Structure

```
adr/      Architecture Decision Records (Context → Decision → Consequences)
README.md Entry point
AGENTS.md This file
```

## Content rules

Public repo. Before adding or editing anything:

- No real IPs, subnets, or hostnames — use `<HOST-IP>`, `<homelab-domain>`
- No personal names, emails, or credentials
- No private file paths (`/opt/docker`, `/srv/docker`, `/mnt/...`)
- Open-source tool names (traefik, prometheus, etc.) are fine

## ADR format

Use `adr/0000-template.md`. Each ADR covers:
- **Context** — what situation forced a decision (3–6 sentences)
- **Decision** — what was chosen and any mandatory conventions
- **Consequences** — positive, negative, neutral trade-offs
- **References** — links to related ADRs or docs in this repo

Keep ADRs short. If a section runs over 10 lines, trim it.

## Pre-publish checklist

Run before committing:

```bash
# Private paths
grep -rn "/opt/docker\|/srv/docker\|/mnt/" --include="*.md"

# Private network topology
grep -rn "192\.168\.\|172\.\(1[6-9]\|2[0-9]\|3[01]\)\.\|\.lan[^>-]" --include="*.md"

# Credentials or personal data
grep -rn "password.*=\|token.*=\|secret.*=\|@.*\.\(com\|pl\|org\|ch\)" --include="*.md"
```

## Link checker

```python
#!/usr/bin/env python3
"""Run from repo root: python3 check_links.py"""
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
                for pat in (LINK_RE, TICK_RE):
                    for m in pat.finditer(line):
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
print(f"All links OK")
```
