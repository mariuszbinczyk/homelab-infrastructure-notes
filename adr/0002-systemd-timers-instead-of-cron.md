---
title: "ADR-0002: systemd Timers Instead of Root Crontab for Scheduled Maintenance"
status: accepted
type: decision
owner: homelab
last_reviewed: 2026-05-02
---

# ADR-0002: systemd Timers Instead of Root Crontab for Scheduled Maintenance

**Date:** 2026-03-15
**Status:** Accepted

---

## Context

The homelab runs a set of recurring maintenance tasks: nightly backups, weekly
CVE scans, automated Tier 2/3 container updates, NAS health recovery, and
scheduled AI alert analysis. Until March 2026 all of these were defined as
entries in the root user's crontab.

Several operational problems accumulated over time:

1. **Invisible failures** — cron jobs write to mail or `/dev/null` by default;
   a silently failing backup or update went unnoticed until a manual check.
2. **No version control** — the crontab lived only on the host; it was not
   tracked in git and could drift from the documented schedule.
3. **No dependency management** — tasks that needed the Docker daemon, NAS
   mount, or network to be ready had no way to express those prerequisites.
4. **Inconsistent log access** — some scripts wrote to custom log files,
   others to journald, making it hard to correlate events.

---

## Decision

All homelab scheduled tasks are managed as **systemd timer + service unit
pairs**. Unit files live in `scripts/systemd/` inside the git repository and
are copied to `/etc/systemd/system/` during setup.

Key conventions:

- Every recurring task is a `.service` unit paired with a `.timer` unit.
- Timer names follow the pattern `homelab-<task>.timer`.
- Logs are written exclusively to **journald**; no custom log files.
  Access pattern: `journalctl -u homelab-<task> -n 50`.
- The `scripts/systemd/` directory is the **source of truth**; the system
  copy under `/etc/systemd/system/` is a deployment artefact.
- New tasks must be added to the schedule reference doc
  (`docs/operations/SCHEDULES.md).

The full active schedule (updated 2026-05-02):

| Timer | Frequency | Purpose |
|-------|-----------|---------|
| `homelab-diagnostics` | daily 01:47 | Health checks, Docker status, disk |
| `homelab-backup` | daily 02:23 | Full backup to NAS |
| `homelab-disk-cleanup` | daily 03:19 | Docker image and build-cache pruning |
| `homelab-ollama-pull` | daily 03:53 | Nightly AI model updates |
| `homelab-cve-scan` | Sat 22:43 | Trivy CVE data collection |
| `homelab-audit` | Sun 01:23 | Consistency audit → Alertmanager |
| `homelab-cve-advisor` | Sun 18:11 | AI CVE analysis → notification |
| `homelab-backup-validation` | Mon 03:47 | Backup integrity verification |
| `homelab-update-tier3` | Mon 05:17 | Tier 3 container auto-update |
| `homelab-update-tier2` | Tue 03:41 | Tier 2 container auto-update |
| `homelab-nas-recovery` | every 10 min (boot-relative) | NAS mount health + auto-remount |
| `homelab-ollama-warmup` | @reboot | Pre-load AI models after boot |

---

## Amendment — Timer Staggering (2026-05-02)

All `OnCalendar=` timers were updated to use non-round-minute times and two
additional directives:

**`RandomizedDelaySec`** — systemd picks a uniform random offset in `[0, N]`
before firing the unit. This prevents thundering-herd behaviour after a reboot
(when all Persistent timers with elapsed schedules would otherwise fire
simultaneously) and distributes I/O jitter across the nightly window.

| Job class | RandomizedDelaySec | Rationale |
|-----------|-------------------|-----------|
| Chained (diagnostics→backup→disk-cleanup) | 60 s | small jitter, chain has 50+ min buffers |
| Independent daily (ollama-pull) | 120 s | no downstream dependency |
| Weekly (cve-scan, updates, cve-advisor) | 120 s | no downstream dependency |
| nas-recovery | 30 s | fires every 10 min, low stakes |

**`AccuracySec=1s`** — overrides the default 1-minute coalescing window for
I/O-heavy jobs (backup, disk-cleanup, cve-scan, cve-advisor, ollama-pull,
backup-validation). Without this, systemd may delay a timer up to 60 seconds
to batch it with other expiring timers, which can push chained jobs past their
safety buffers.

**`homelab-nas-recovery`** was changed from clock-aligned `OnCalendar=*:0/10`
to `OnBootSec=3min` + `OnUnitActiveSec=10min`. This decouples it from the wall
clock so multiple reboots do not converge the recovery timer on the same
10-minute mark system-wide.

---

## Consequences

### Positive

- All task output lands in journald, enabling unified log queries and
  persistent history across reboots.
- Unit files are version-controlled; the scheduled task definition and the
  running system can be diffed.
- systemd can express `After=`, `Requires=`, and `ConditionPathExists=`
  guards, removing the need for defensive checks inside scripts.
- `systemctl list-timers 'homelab-*'` gives an immediate view of all upcoming
  and last-run homelab tasks.
- `RandomizedDelaySec` eliminates post-reboot thundering-herd and spreads
  nightly I/O without requiring per-script sleep hacks.

### Negative / Trade-offs

- Adding or modifying a timer requires `sudo systemctl daemon-reload` and
  root access; crontab edits needed only user access.
- Unit file syntax is more verbose than a crontab line.
- Existing scripts must be invoked from the `ExecStart=` line rather than
  being edited to understand timers.
- `RandomizedDelaySec` means the actual fire time is non-deterministic within
  a small window; log timestamps will vary slightly each day.

### Neutral

- The cron syntax used for timer `OnCalendar=` entries is slightly different
  from POSIX cron (`DayOfWeek YYYY-MM-DD HH:MM:SS` vs `* * * * *`).
  Engineers familiar only with crontab syntax will need a brief lookup.

---

## References

- `docs/operations/SCHEDULES.md` — current schedule reference
- `scripts/systemd/` — unit files (source of truth, git-tracked)
- `man systemd.timer` — systemd timer documentation
