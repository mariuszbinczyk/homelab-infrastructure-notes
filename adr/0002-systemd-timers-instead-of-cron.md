---
title: "ADR-0002: systemd Timers Instead of Root Crontab for Scheduled Maintenance"
status: accepted
type: decision
owner: homelab
last_reviewed: 2026-04-28
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
  (`dosystemd timer documentation: `man systemd.timer`

The full active schedule as of the decision date:

| Timer | Frequency | Purpose |
|-------|-----------|---------|
| `homelab-diagnostics` | daily 02:00 | Health checks, Docker status, disk |
| `homelab-backup` | daily 02:30 | Full backup to NAS |
| `homelab-disk-cleanup` | daily 03:30 | Docker image and build-cache pruning |
| `homelab-cve-scan` | Sat 23:00 | Trivy CVE data collection |
| `homelab-audit` | Sun 02:00 | Consistency audit → Alertmanager |
| `homelab-cve-advisor` | Sun 18:00 | AI CVE analysis → notification |
| `homelab-backup-validation` | Mon 03:00 | Backup integrity verification |
| `homelab-update-tier3` | Mon 05:30 | Tier 3 container auto-update |
| `homelab-update-tier2` | Tue 03:30 | Tier 2 container auto-update |
| `homelab-nas-recovery` | every 10 min | NAS mount health + auto-remount |
| `homelab-ollama-warmup` | @reboot | Pre-load AI models after boot |

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

### Negative / Trade-offs

- Adding or modifying a timer requires `sudo systemctl daemon-reload` and
  root access; crontab edits needed only user access.
- Unit file syntax is more verbose than a crontab line.
- Existing scripts must be invoked from the `ExecStart=` line rather than
  being edited to understand timers.

### Neutral

- The cron syntax used for timer `OnCalendar=` entries is slightly different
  from POSIX cron (`DayOfWeek YYYY-MM-DD HH:MM:SS` vs `* * * * *`).
  Engineers familiar only with crontab syntax will need a brief lookup.

---

## References

- `systemd timer documentation: `man systemd.timer`
- `../../scripts/systemd/` — unit files (source of truth, git-tracked)
- systemd timer documentation: `man systemd.timer`
