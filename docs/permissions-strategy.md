---
title: "Container Permissions Strategy"
status: active
type: explanation
owner: homelab
last_reviewed: 2026-03-10
---

# Homelab Permissions Strategy

**Version:** 1.0
**Last Updated:** 2026-03-10
**Author:** Homelab Infrastructure Team

---

## Table of Contents

1. [Overview](#overview)
2. [Global Solution](#global-solution)
3. [Exceptions (Root Required)](#exceptions-root-required)
4. [Best Practices](#best-practices)
5. [Implementation Checklist](#implementation-checklist)
6. [Troubleshooting](#troubleshooting)

---

## Overview

This document establishes a unified permissions strategy for the entire homelab infrastructure. The primary goal is to **minimize privilege escalation risks** while maintaining operational functionality.

**Core Principle:** All containers run as unprivileged user with UID=1000, GID=1000 (operator account) unless explicitly justified.

---

## Global Solution

### 1. Standard User Configuration

All containers use the following user configuration:

```bash
PUID=1000  # operator account UID
PGID=1000  # operator account GID
UMASK=002  # Default permissions: 775 for directories, 664 for files
```

### 2. Global Environment Variables

Add to `shared/.env.global`:

```bash
# User/Group Configuration
PUID=1000
PGID=1000
UMASK=002
```

### 3. Docker Compose Configuration

Every `docker-compose.yml` should include:

```yaml
services:
  service_name:
    user: '${PUID}:${PGID}'
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - UMASK=${UMASK}
    # ... rest of configuration
```

### 4. Filesystem ACL Configuration

Apply ACL to all mount points to ensure consistent permissions:

```bash
# Jellyfin media directory
sudo setfacl -R -m u:1000:rwx,d:u:1000:rwx <media-dir>

# Docker volumes and configs
sudo setfacl -R -m u:1000:rwx,d:u:1000:rwx <data-dir>

# Backup directory
sudo setfacl -R -m u:1000:rwx,d:u:1000:rwx <backup-dir>
```

**Explanation:**
- `-R`: Recursive
- `-m`: Modify ACL
- `u:1000:rwx`: Grant user 1000 read/write/execute
- `d:u:1000:rwx`: Default ACL for new files/directories

### 5. Sticky Bit on Shared Directories

Set group sticky bit to preserve group ownership:

```bash
# Apply to all service directories
sudo find <data-dir> -type d -exec chmod g+s {} \;

# Or individually:
sudo chmod g+s <data-dir>/jellyfin
sudo chmod g+s <data-dir>/grafana
sudo chmod g+s <data-dir>/prometheus
# ... etc
```

**Benefit:** New files inherit the directory's group ownership.

### 6. Never Run as Root (Unless Justified)

**Default stance:** No container runs as root.

Before granting root privileges, ask:
1. Does it need privileged ports (<1024)?
2. Does it need host system access?
3. Can capabilities solve the problem instead?

---

## Exceptions (Root Required)

These services have documented justification for root access:

### 1. Traefik
**Why:** Binds to privileged ports 80/443
**Mitigation:** Uses NET_BIND_SERVICE capability

```yaml
services:
  traefik:
    cap_add:
      - NET_BIND_SERVICE
    # Note: Still runs as root, consider using authbind in future
```

### 2. AdGuard Home
**Why:** Binds to privileged port 53 (DNS)
**Mitigation:** Network isolation, read-only root filesystem where possible

```yaml
services:
  adguard:
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    # Runs as root for DNS port access
```

### 3. MongoDB
**Why:** MongoDB startup requires root for data directory ownership and lock files
**Mitigation:** Network isolation (homelab-internal only, no external ports)

```yaml
services:
  mongodb:
    # ROOT EXCEPTION: MongoDB init requires root for data dir ownership
```

### 4. Uptime Kuma
**Why:** Uses `setpriv` which requires root privileges; fails with non-root (`setpriv: setgroups failed: Operation not permitted`)
**Mitigation:** Docker socket mounted read-only, resource limits enforced

```yaml
services:
  uptime-kuma:
    # ROOT EXCEPTION: setpriv requires root (louislam/uptime-kuma image limitation)
```

### 5. bookstack-db (MariaDB 11)
**Why:** MariaDB initialization scripts require root to set up data directory and user grants
**Mitigation:** Network isolation (homelab-internal only), dedicated DB user for BookStack

```yaml
services:
  bookstack-db:
    # ROOT EXCEPTION: MariaDB init requires root (standard for all MySQL/MariaDB images)
```

---

## Best Practices

### 1. Principle of Least Privilege

- Grant **minimum necessary permissions**
- Use capabilities instead of full root where possible
- Regularly audit permission requirements

### 2. Consistent UID/GID Across All Containers

**Benefits:**
- Simplified permission management
- Predictable file ownership
- Easier backup/restore operations
- Reduced attack surface

**Implementation:**
- Use `PUID=1000` and `PGID=1000` everywhere
- Verify with: `docker exec <container> id`

### 3. Docker Secrets for Sensitive Data

**Never store secrets in:**
- Plain text files
- Environment variables (visible in `docker inspect`)
- Git repositories

**Use instead:**
```yaml
services:
  service_name:
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### 4. Never Use chmod 777

**Why it's dangerous:**
- World-writable = anyone can modify
- Opens door to privilege escalation
- Indicates misconfigured permissions

**Alternatives:**
- Use ACL: `setfacl -m u:1000:rwx <path>`
- Fix ownership: `chown 1000:1000 <path>`
- Use `chmod 775` or `chmod 770` with correct group

### 5. Document Every Exception

When a container **must** run as root:
1. Document the reason in `docker-compose.yml` comments
2. Add to this document's Exceptions section
3. Implement mitigation strategies
4. Set a review date to reconsider

**Example:**
```yaml
services:
  special-service:
    # RUNS AS ROOT: Requires raw socket access for network monitoring
    # MITIGATION: Read-only filesystem, dropped capabilities
    # REVIEW DATE: 2025-12-01
    user: "0:0"
    cap_drop:
      - ALL
    cap_add:
      - NET_RAW
    read_only: true
```

### 6. Use Read-Only Filesystems Where Possible

```yaml
services:
  service_name:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

### 7. Drop Unnecessary Capabilities

```yaml
services:
  service_name:
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
```

---

## Implementation Checklist

### Phase 1: Global Configuration
- [ ] Add PUID/PGID/UMASK to `shared/.env.global`
- [ ] Verify operator user exists with UID=1000, GID=1000
- [ ] Document current root-running containers

### Phase 2: Filesystem Permissions
- [ ] Apply ACL to `<media-dir>`
- [ ] Apply ACL to `<data-dir>`
- [ ] Apply ACL to `<backup-dir>`
- [ ] Set sticky bit on all `<data-dir>/*` directories
- [ ] Verify permissions: `getfacl <path>`

### Phase 3: Container Migration
For each service:
- [ ] Add `user: '${PUID}:${PGID}'` to `docker-compose.yml`
- [ ] Add environment variables (PUID, PGID, UMASK)
- [ ] Test container startup
- [ ] Verify file permissions inside container
- [ ] Check application functionality
- [ ] Document any issues

### Phase 4: Validation
- [ ] Run: `docker ps --format "{{.Names}}: {{.User}}"` to check users
- [ ] Verify no unexpected `root` users
- [ ] Test backup/restore with new permissions
- [ ] Update documentation

### Phase 5: Monitoring
- [ ] Add Prometheus alert for containers running as root (optional)
- [ ] Schedule quarterly permission audit
- [ ] Create runbook for permission issues

---

## Troubleshooting

### Container Won't Start After Adding user: Directive

**Symptom:** Container exits immediately or shows permission errors

**Solutions:**
1. Check if directories are owned by UID 1000:
   ```bash
   ls -la <data-dir>/<service>/
   ```

2. Fix ownership:
   ```bash
   sudo chown -R 1000:1000 <data-dir>/<service>/
   ```

3. Check ACL:
   ```bash
   getfacl <data-dir>/<service>/
   ```

4. Verify container actually supports PUID/PGID (linuxserver.io images do)

### Permission Denied Errors

**Symptom:** Application logs show "permission denied" errors

**Solutions:**
1. Check umask inside container:
   ```bash
   docker exec <container> sh -c 'umask'
   ```

2. Verify UMASK environment variable is set:
   ```bash
   docker exec <container> env | grep UMASK
   ```

3. Check if app creates files with correct permissions:
   ```bash
   docker exec <container> ls -la /config/
   ```

### ACL Not Working

**Symptom:** Files created with wrong permissions despite ACL

**Solutions:**
1. Verify ACL support on filesystem:
   ```bash
   mount | grep /srv
   ```
   Should show `acl` in options

2. Remount with ACL support:
   ```bash
   sudo mount -o remount,acl /srv
   ```

3. Make permanent in `/etc/fstab`:
   ```
   UUID=xxx /srv ext4 defaults,acl 0 2
   ```

### Service Requires Root (New Discovery)

**Process:**
1. Document why root is needed in service's `docker-compose.yml`
2. Add to [Exceptions](#exceptions-root-required) section
3. Implement mitigation:
   - Drop unnecessary capabilities
   - Use read-only filesystem
   - Restrict network access
   - Use security profiles (AppArmor/SELinux)
4. Set review date

---

## References

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [ACL Documentation](https://linux.die.net/man/5/acl)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

---

## Changelog

| Date       | Version | Changes                          |
|------------|---------|----------------------------------|
| 2025-11-08 | 1.0     | Initial permissions strategy     |
| 2026-03-10 | 1.1     | Dodano wyjątki: MongoDB, Uptime Kuma, bookstack-db (MariaDB). Usunięto błędne wpisy Node Exporter i cAdvisor (nie są root). |

---

**Next Review Date:** 2026-02-08 (Quarterly)
