# VPS Audit

A repeatable process for inspecting your VPS without mixing observation with changes. Run this when you're not sure what's running, after a long break, or before any major maintenance.

---

## Principle

Audit first, change second. Never run a maintenance task without understanding the current state. Every time you "just fix it quickly" without checking the state first, you risk breaking something that was working.

---

## Checklist

### 1. Disk

```bash
df -h /
docker system df
```

Expect:
- Root filesystem under 70%
- Identify which Docker component (images, volumes, build cache) is the largest if disk is high

---

### 2. Running Containers

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Record:
- Which containers are running
- Which have been `Restarting` — this means they are crash-looping
- Which ports are exposed to `0.0.0.0` vs localhost vs Tailscale IP

---

### 3. Services (systemd)

```bash
systemctl list-units --type=service --state=running
```

Record any persistent services not managed by Docker (e.g. Coolify itself, custom APIs, scheduled jobs). Every persistent process should be a systemd unit.

---

### 4. Service Bindings

For every service listening on a port:

```bash
ss -tlnp
```

Check:
- Internal services (databases, admin panels) should be bound to `127.0.0.1` or a private IP (Tailscale), not `0.0.0.0`
- Compare what you see in `ss -tlnp` against the config files that will be loaded on next restart

---

### 5. UFW Rules

```bash
ufw status numbered
```

For each open port, verify something is listening:

```bash
ss -tlnp | grep <port>
```

Delete any rule with no listener.

---

### 6. Logs — Recent Errors

```bash
# System errors in the last hour
journalctl -p err --since "1 hour ago"

# Specific service logs
journalctl -u <service> --since "1 hour ago" --no-pager

# Docker container logs
docker logs <container> --tail 50
```

---

### 7. Scheduled Jobs

```bash
# Cron jobs for current user
crontab -l

# Systemd timers
systemctl list-timers
```

Record what runs automatically and when.

---

### 8. Backup State

```bash
# Find recent backup files
find /tmp /srv /home -name "*.sql.gz" -o -name "*.tar.gz" 2>/dev/null | sort -t: -k2 | tail -10
ls -lh ~/srv/backups/ 2>/dev/null
```

Record: latest backup date, destination (local/remote), and whether it was verified.

---

### 9. Security Baseline

See [security-hardening.md](security-hardening.md) for a full pass. At minimum:

```bash
# SSH config
sshd -T | grep -E "^(permitrootlogin|passwordauthentication)"
# Expected: prohibit-password, no

# fail2ban
fail2ban-client status sshd

# Docker daemon on TCP (should be empty)
ss -tlnp | grep 2375
```

---

## Output Format

After each audit, document:

```
## Audit — [date]

### Verified Facts
- [container/service]: [state] — [what it does]
- ...

### Gaps (could not verify)
- [item]: [why]

### Issues Found
- 🔴 [item]: [what's wrong] — [fix needed]
- 🟠 [item]: ...

### Actions Taken
- [what was changed during this session]

### Next Scheduled Audit
[date or trigger]
```

Save this as `AUDITS/[date]-vps-audit.md` in your control plane or a private maintenance repo.
