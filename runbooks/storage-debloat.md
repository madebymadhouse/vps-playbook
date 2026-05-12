# Storage Debloat

Run this when disk usage climbs above 60-70% or the server is getting sluggish.

---

## Step 1 — Check current state

```bash
# Disk overview
df -h /

# Docker-specific breakdown
docker system df
```

`docker system df` shows: images, containers, volumes, build cache — and which are unused.

---

## Step 2 — Docker cleanup

**Safe cleanup (removes unused resources, does not touch running containers):**

```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune -f
```

This is always safe while the Coolify stack is live. Running containers keep their images.

**Targeted cleanup:**

```bash
# Dangling images only
docker image prune -f

# Remove images not used in the last 24 hours
docker image prune -a -f --filter "until=24h"

# Stopped containers
docker container prune -f

# Unused volumes (careful: removes volumes with no attached container)
docker volume prune -f

# Build cache only — often the biggest win (10–30 GB on active build servers)
docker builder prune -f
```

Run `docker system df` after each prune to see what freed space.

---

## Step 3 — If still high: deeper passes

### Remove all unused images

```bash
# WARNING: Coolify will re-pull images on next deploy
docker image prune -a -f
```

Only do this if disk is critically full. All running services keep their images; this only removes images not currently in use by any container.

### NVM versions

```bash
nvm ls
ls ~/.nvm/versions/
# Remove versions you don't use
nvm uninstall <old-version>
```

Keep only the active version (and one LTS if you're switching).

### Rust toolchains

```bash
rustup toolchain list
rustup toolchain uninstall <old-toolchain>
```

### Package manager caches

```bash
# npm cache
npm cache clean --force

# pip cache
pip cache purge

# pnpm cache
pnpm store prune
```

### Find large files

```bash
# Files over 200 MB
find / -type f -size +200M -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# Top directories by size
du -sh /var/* /data/* 2>/dev/null | sort -rh | head -20
```

Common culprits:
- `/var/lib/docker/overlay2/` (Docker layers — use `docker system prune` not direct deletion)
- `/data/coolify/` (Coolify data, logs, build artifacts)
- Database data directories

---

## Step 4 — Log rotation

If `/var/lib/docker/containers/` is large, containers are writing unbounded logs.

Configure rotation globally:

```bash
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

systemctl restart docker
```

This limits each container's logs to 30 MB max. Applies to new containers; restart existing containers for it to take effect.

---

## Step 5 — Verify

```bash
df -h /
docker system df
```

Record the before/after numbers. A healthy post-debloat state is under 50% disk usage.
