---
title: "ADR-0003: All Containers Run as UID/GID 1000 with Documented Root Exceptions"
status: accepted
type: decision
owner: homelab
last_reviewed: 2026-04-28
---

# ADR-0003: All Containers Run as UID/GID 1000 with Documented Root Exceptions

**Date:** 2025-10
**Status:** Accepted

---

## Context

Docker containers run as root by default unless a `user:` directive is set.
In a homelab with bind-mounted host directories this creates two problems:

1. **File ownership drift** — files written by root-in-container are owned
   by root on the host, making manual inspection and backup restoration
   unnecessarily complex.
2. **Privilege escalation surface** — a container escape by a root-running
   process gives the attacker root on the host directly.

The homelab host runs all user processes under a single non-privileged account
with UID/GID 1000. Bind-mounted data directories are owned by this user.
Running containers as a different UID leads to permission errors on mounts,
which historically required workarounds that added complexity.

---

## Decision

Every container in the homelab **must** declare:

```yaml
user: '${PUID:-1000}:${PGID:-1000}'
```

where `PUID` and `PGID` default to `1000:1000` via the shared `.env` file.

**Root exceptions** are permitted only when the upstream image has a hard
requirement for `root` (e.g. it binds a privileged port at startup, performs
kernel-level operations, or does not support a `user:` override). Every
exception must be documented with an inline comment in the compose file:

```yaml
# ROOT EXCEPTION: <reason why root is required>
```

Known permanent exceptions as of the decision date:

| Service | Reason |
|---------|--------|
| `traefik` | Binds ports 80/443 on the host network |
| `adguard` | Binds UDP port 53 (privileged) |
| `mongodb` | Upstream image requires root for data directory initialisation |
| `uptime-kuma` | Upstream image does not support non-root; no data ownership conflict |

---

## Consequences

### Positive

- Host files created or written by containers are owned by UID 1000 and
  immediately accessible without `sudo chown`.
- Backup and restore operations work without privilege escalation.
- The principle of least privilege is applied consistently; deviations are
  visible and auditable in the compose file.

### Negative / Trade-offs

- Some upstream images assume root and require extra environment variables
  (`PUID`, `PGID`, or `RUN_AS_USER`) to work correctly; this must be verified
  when adding each new container.
- Images that do not support non-root at all must go through the exception
  process, which adds a comment-writing step to onboarding.

### Neutral

- The `NEW_CONTAINER_CHECKLIST.md` enforces this decision as a required
  checklist item, so it does not rely on individual memory.

---

## References

- `../docs/new-container-checklist.md` — enforces this rule during onboarding
- `../docs/permissions-strategy.md` — full permissions strategy document
- `../docs/permissions-strategy.md` — quick-reference card
