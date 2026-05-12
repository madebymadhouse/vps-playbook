# Security Hardening

A complete, repeatable security baseline check for a production VPS. Run this after any infrastructure change, or any time you want to verify the server is in a known-good state.

---

## The Persistent Config Rule

Before anything else: a fix is only done when it survives a restart.

Checking `ss -tlnp` or `ps aux` does not verify a fix. It verifies the current in-memory state. The real source of truth is the config file that gets loaded on the next restart. Always:

1. Fix the config file.
2. Restart the service.
3. Re-verify the running state after restart.

Any fix that skips steps 2 and 3 is not complete.

---

## Checklist

### 1. UFW — No Stale Rules

A stale rule is a port open in UFW with nothing listening on it. Every open port must map to a live service.

```bash
# List all open ports with rule numbers
ufw status numbered

# For each port, verify something is actually listening
ss -tlnp | grep <port>
```

Expected: every port in `ufw status` has a matching entry in `ss -tlnp`.

Fix: delete any rule with no listener:
```bash
# Delete by rule number (delete higher numbers first to avoid renumbering)
ufw delete <rule_number>
ufw status verbose   # verify
```

---

### 2. Service Bindings — Config vs Runtime

For every service that binds to a network address, verify the config file matches what is running, and that a clean restart does not regress.

```bash
# Find the EnvironmentFile for a systemd service
systemctl show <service> -p EnvironmentFiles

# Read the config
cat /etc/<service>.env

# Restart and re-verify
systemctl restart <service>
sleep 2
ss -tlnp | grep <port>
```

Expected: internal services should bind to `127.0.0.1` or a private network IP (e.g. Tailscale), not `0.0.0.0`.

---

### 3. File Permissions

All secret-containing files must have correct permissions before any task is closed.

```bash
# Systemd EnvironmentFiles
stat -c '%a %U:%G %n' /etc/<service>.env
# Expected: 600 root:root

# Application .env files
find ~/srv -name ".env" -not -path "*/.git/*" -exec stat -c '%a %n' {} \;
# Expected: all 600

# Private keys
stat -c '%a %n' ~/.ssh/id_*
# Expected: 600
```

Fix:
```bash
chmod 600 <file>
chown root:root <file>   # for system files only
```

---

### 4. SSH Hardening

```bash
sshd -T | grep -E "^(permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries|port)"
```

Expected:
```
permitrootlogin prohibit-password
passwordauthentication no
pubkeyauthentication yes
maxauthtries 3
```

If any line differs, edit `/etc/ssh/sshd_config` and run `systemctl reload sshd`.

Review authorized keys:
```bash
cat ~/.ssh/authorized_keys
wc -l ~/.ssh/authorized_keys
```

Remove any key you don't recognize.

---

### 5. fail2ban Active

```bash
fail2ban-client status sshd
```

Expected: `Currently banned: 0` (unless someone is actually being blocked).

If fail2ban is not running:
```bash
systemctl start fail2ban
systemctl enable fail2ban
```

---

### 6. Orphan Processes

Every long-running process must be owned by systemd. Check for orphans started manually:

```bash
ps aux | grep -v "\[" | awk 'NR>1' | grep -v "systemd\|sshd\|docker\|containerd\|cron\|grep"
```

If a process needs to run continuously, it needs a unit file. If it is an old experiment, kill it.

---

### 7. Docker Socket

```bash
stat -c '%a %n' /var/run/docker.sock
# Expected: 660

ls -la /var/run/docker.sock
# Expected: srw-rw---- root docker
```

No container should run with `privileged: true` unless explicitly documented:

```bash
docker inspect $(docker ps -q) --format '{{.Name}} privileged={{.HostConfig.Privileged}}' | grep "privileged=true"
# Expected: no output
```

---

### 8. Docker Daemon Not Exposed on TCP

```bash
ss -tlnp | grep 2375
ss -tlnp | grep 2376
# Expected: no output
```

If either returns output, the Docker daemon is exposed on TCP without TLS. This is critical — stop it immediately.

---

### 9. iptables DOCKER-USER Chain

Docker bypasses UFW for container port-forwarding. The real security for container-exposed ports comes from the `DOCKER-USER` chain.

```bash
iptables -L DOCKER-USER -n -v
```

Verify rules are restricting access to container-bound ports. If you expose services via Docker port bindings to `0.0.0.0`, you must add `DOCKER-USER` rules to restrict who can reach them.

**Note:** `DOCKER-USER` rules that reference container IPs become stale when containers are redeployed. Re-verify after any redeploy.

---

## After the Checklist

1. If all items pass: document the date in your audit log.
2. If any item fails: fix it before declaring done. Do not close a task with known open findings.
3. Update your audit log with the date and outcome.
