# VPS Playbook

Go from "fresh VPS" to "production-ready in 60 minutes."

This playbook covers the complete setup sequence for a personal or small-team VPS: hardening, firewall, Tailscale overlay, Docker, Coolify, DNS, and ongoing operations. Follow it top to bottom on a fresh machine.

**Reference implementation:** Mad House runs a Hostinger KVM2 VPS with 96 GB disk, 8 active services, Coolify 4.0, and a Tailscale overlay for private service access. This playbook is the generalized version of that setup.

---

## Table of Contents

1. [Why self-host](#1-why-self-host)
2. [Choosing a VPS provider](#2-choosing-a-vps-provider)
3. [Initial hardening](#3-initial-hardening)
4. [UFW firewall](#4-ufw-firewall)
5. [Tailscale setup](#5-tailscale-setup)
6. [Docker install](#6-docker-install)
7. [Coolify install](#7-coolify-install)
8. [Disk management](#8-disk-management)
9. [Monitoring](#9-monitoring)
10. [Backup patterns](#10-backup-patterns)
11. [DNS and Cloudflare](#11-dns-and-cloudflare)
12. [Cost optimization](#12-cost-optimization)

---

## 1. Why self-host

**Cost:** A $12/month VPS running 8 services costs less than a single Heroku dyno with a Postgres add-on. Vercel, Railway, and Render are excellent for small projects, but costs compound fast as you add services, team members, or bandwidth.

**Control:** You own the machine. You choose the Docker version, the OS, the firewall rules. You can run anything: databases, queues, crons, internal tools.

**Privacy:** Your data stays on hardware you control. No SaaS vendor processes your database contents.

**Learning:** Understanding what PaaS providers do for you makes you a better engineer. Running your own stack teaches Docker networking, TLS, DNS, and Linux administration in ways that managed platforms hide.

**Tradeoffs:** You are responsible for uptime, security patches, and backups. Managed platforms handle those for you. Self-hosting is the right choice when the cost savings and control outweigh the operational burden. For most solo founders and small teams, they do.

---

## 2. Choosing a VPS provider

Rough comparison of popular providers. Prices are approximate and change frequently. Check current pricing on each provider's site.

| Provider | Entry tier | Mid tier | Notes |
|---|---|---|---|
| Hetzner | ~$4/mo (2 vCPU, 4 GB) | ~$15/mo (4 vCPU, 16 GB) | Best price-to-performance in Europe. Datacenter locations: Germany, Finland, USA |
| Hostinger | ~$5/mo (2 vCPU, 8 GB) | ~$14/mo (8 vCPU, 32 GB) | Good price, global locations, KVM virtualization |
| DigitalOcean | ~$12/mo (2 vCPU, 2 GB) | ~$24/mo (2 vCPU, 4 GB) | Premium UX, strong docs, higher cost per GB |
| Vultr | ~$6/mo (1 vCPU, 1 GB) | ~$12/mo (1 vCPU, 2 GB) | Good global coverage, NVMe options |
| Linode/Akamai | ~$12/mo (2 vCPU, 4 GB) | ~$24/mo (4 vCPU, 8 GB) | Reliable, good managed options |

**Recommended specs for a Coolify server:**

- Minimum: 2 vCPU, 4 GB RAM, 40 GB disk
- Comfortable: 4 vCPU, 8 GB RAM, 80 GB disk
- Storage-heavy workloads: add a block volume or choose a storage-optimized tier

**What to look for:**

- KVM virtualization (not OpenVZ or LXC; Docker requires real kernel access)
- NVMe or SSD disk (HDD is too slow for container workloads)
- Hourly billing or easy resize (you may want to upgrade later without migrating)
- IPv4 included (some cheap providers charge extra)

**OS choice:** Ubuntu 22.04 LTS or 24.04 LTS. These have the widest support, best Docker compatibility, and long support windows. Debian 12 works equally well.

---

## 3. Initial hardening

Run these steps immediately after your VPS provisions. Do not expose your server to traffic before completing hardening.

### Connect as root

```bash
ssh root@YOUR_VPS_IP
```

### Create a non-root user

```bash
adduser sammy
usermod -aG sudo sammy
```

Replace `sammy` with your username. All subsequent steps can run as this user with `sudo`.

### Copy your SSH public key to the new user

From your local machine:

```bash
ssh-copy-id sammy@YOUR_VPS_IP
```

Or manually: on the VPS as root:

```bash
mkdir -p /home/sammy/.ssh
cat >> /home/sammy/.ssh/authorized_keys << 'EOF'
YOUR_PUBLIC_KEY_CONTENT_HERE
EOF
chown -R sammy:sammy /home/sammy/.ssh
chmod 700 /home/sammy/.ssh
chmod 600 /home/sammy/.ssh/authorized_keys
```

### Disable password authentication and root login

```bash
# Test SSH key login first in a new terminal before doing this
sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

Confirm you can still SSH as `sammy` before closing the root session.

### Install fail2ban

fail2ban watches SSH logs and bans IPs after repeated failed attempts.

```bash
apt update
apt install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

Default config bans IPs after 5 failed SSH attempts for 10 minutes. Adequate for most setups.

### Update the system

```bash
apt update && apt upgrade -y
apt autoremove -y
```

### Set the hostname

```bash
hostnamectl set-hostname myserver
```

---

## 4. UFW firewall

UFW (Uncomplicated Firewall) manages iptables rules with a simple interface.

### Install and configure

```bash
apt install -y ufw

# Default policy: deny all incoming, allow all outgoing
ufw default deny incoming
ufw default allow outgoing

# Allow SSH before enabling, or you will lock yourself out
ufw allow ssh

# Allow web traffic
ufw allow 80/tcp
ufw allow 443/tcp

# Allow Coolify dashboard (only if you are not putting it behind a domain)
# You can restrict this to your IP instead
ufw allow 8000/tcp

# Enable the firewall
ufw enable
```

### Verify

```bash
ufw status verbose
```

Output should show:

```
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
8000/tcp                   ALLOW IN    Anywhere
```

### Restrict dashboard to your IP (optional but recommended)

```bash
ufw delete allow 8000/tcp
ufw allow from YOUR_HOME_IP to any port 8000
```

Or use Tailscale (section 5) and bind the Coolify dashboard only to the Tailscale interface.

### Services behind Tailscale only

For internal services (databases, admin panels, internal APIs), do not open their ports in UFW. Bind them to the Tailscale IP only. Traffic only flows over the private overlay network.

---

## 5. Tailscale setup

Tailscale creates a private overlay network (WireGuard-based) across all your devices. Your VPS, laptop, and CI runner can all reach each other by hostname without exposing ports to the internet.

### Why Tailscale

- Database on VPS: accessible from your laptop without opening port 5432 to the world
- SSH: connect by hostname (`ssh myserver`) without IP changes when you move networks
- Internal APIs: services that should never be public are only reachable within your Tailnet
- No VPN headaches: Tailscale handles NAT traversal automatically

### Install on VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Follow the auth link to connect the VPS to your Tailnet. After auth:

```bash
tailscale ip -4
# Returns something like: 100.x.y.z
```

### Install on your laptop

Download from [tailscale.com/download](https://tailscale.com/download). Authenticate with the same account.

### Connect by hostname

```bash
ssh sammy@myserver
```

Tailscale DNS (MagicDNS) resolves machine names within your Tailnet. Enable it in the Tailscale admin panel.

### Bind services to Tailscale only

For a database that should only be accessible from your laptop:

```yaml
services:
  postgres:
    image: postgres:16
    ports:
      - "100.x.y.z:5432:5432"   # Bind to Tailscale IP only
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Replacing `100.x.y.z` with your VPS Tailscale IP. This port is not reachable from the internet, only from devices in your Tailnet.

### Tailscale ACLs

In the Tailscale admin panel, configure ACLs to control which devices can reach which services. The default allows all devices to reach all others in your Tailnet. For teams, restrict access by tag.

---

## 6. Docker install

Use the official Docker install method. The Ubuntu snap package is not suitable for production (permission issues, slower updates, different file paths).

```bash
# Remove any old Docker versions
apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

# Install dependencies
apt update
apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable Docker on boot
systemctl enable docker
systemctl start docker

# Add your user to the docker group (avoid sudo for every docker command)
usermod -aG docker sammy
```

Log out and back in for the group change to take effect. Verify:

```bash
docker run hello-world
```

### Docker Compose

The `docker-compose-plugin` package gives you `docker compose` (v2, space, no hyphen). The old `docker-compose` (hyphen) is v1 and is deprecated. Use `docker compose` in all scripts.

---

## 7. Coolify install

With Docker installed, install Coolify:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

For full Coolify configuration (projects, deploy keys, auto-deploy, Traefik routing), see the companion guide:

**[coolify-playbook](https://github.com/madebymadhouse/coolify-playbook)**

It covers everything from service registration to webhook setup to env var management.

---

## 8. Disk management

### Check disk usage

```bash
df -h
```

Watch the `/` partition. At 85% full, start cleaning. At 95%, Docker may fail to pull images.

```bash
# Docker-specific usage breakdown
docker system df
```

Shows how much space is used by images, containers, volumes, and build cache.

### Docker cleanup

**Safe cleanup (removes unused resources):**

```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune

# Also remove unused volumes (careful: this removes volumes not attached to any container)
docker system prune --volumes
```

**Targeted cleanup:**

```bash
# Remove dangling images only
docker image prune

# Remove images not used in the last 24 hours
docker image prune -a --filter "until=24h"

# Remove stopped containers
docker container prune

# Remove unused volumes
docker volume prune
```

Run `docker system prune` monthly as a habit. Coolify build cache grows fast.

### Log rotation

Docker containers write logs to JSON files on the host. Without rotation, these grow unbounded.

Configure log rotation globally:

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

This limits each container's log to 3 files of 10 MB each (30 MB per container max).

### Large file hunting

```bash
# Find files over 100 MB
find / -type f -size +100M -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# Check directory sizes
du -sh /var/* | sort -rh | head -20
du -sh /data/coolify/* | sort -rh
```

Common large-file culprits:
- `/var/lib/docker/overlay2/` (Docker layers)
- `/data/coolify/` (Coolify data, logs, build artifacts)
- Database data directories

---

## 9. Monitoring

### Coolify Sentinel

Coolify includes a Sentinel component that monitors resource metrics (CPU, memory, disk) and can send alerts. Enable it in Coolify settings:

1. Coolify sidebar > Servers > your server > Sentinel
2. Enable Sentinel
3. Configure alert thresholds and notification channels (email, Slack, Discord)

Sentinel updates every 5 seconds and stores metrics in the Coolify database.

### Simple uptime check

For public-facing services, use a free uptime monitoring service:

- [UptimeRobot](https://uptimerobot.com) (free tier: 50 monitors, 5-minute intervals)
- [BetterStack](https://betterstack.com) (free tier: 10 monitors)
- [Freshping](https://freshping.io) (free tier: 50 monitors, 1-minute intervals)

Configure an HTTP monitor pointing at your domain. Set up email or Slack alerts for downtime.

### System metrics

For a lightweight self-hosted option, install [Netdata](https://www.netdata.cloud) or run a simple cron that checks disk and emails you:

```bash
# Add to crontab: daily disk usage alert
0 9 * * * df -h | awk '$5+0 > 80 {print "DISK ALERT: " $0}' | mail -s "Disk Alert" you@example.com
```

Or use the Coolify-integrated approach: Coolify Sentinel handles this without additional tools.

---

## 10. Backup patterns

### What to back up

| Data | Backup method |
|---|---|
| PostgreSQL databases | `pg_dump` + upload to S3/R2 |
| Application uploads/files | Volume snapshot or rsync to remote |
| Docker volumes | `docker run --rm -v VOLUME:/data -v /backup:/backup alpine tar czf /backup/VOLUME.tar.gz /data` |
| Coolify configuration | `/data/coolify/` directory |
| SSH keys | Secure local copy |

### PostgreSQL backup

```bash
# Dump a Postgres database (run from VPS or inside the container)
docker exec POSTGRES_CONTAINER pg_dump -U postgres DBNAME > /tmp/backup_$(date +%Y%m%d).sql

# Compress it
gzip /tmp/backup_$(date +%Y%m%d).sql

# Upload to Cloudflare R2 (with rclone or aws cli)
aws s3 cp /tmp/backup_$(date +%Y%m%d).sql.gz s3://your-backup-bucket/postgres/ \
  --endpoint-url https://YOUR_ACCOUNT.r2.cloudflarestorage.com
```

Automate with a cron job:

```bash
0 2 * * * /home/sammy/scripts/backup-postgres.sh >> /var/log/backup.log 2>&1
```

### Volume backup

```bash
# Back up a named Docker volume
docker run --rm \
  -v myapp-data:/source \
  -v /tmp/backups:/backup \
  alpine \
  tar czf /backup/myapp-data-$(date +%Y%m%d).tar.gz -C /source .
```

### Coolify configuration backup

```bash
# Back up the full Coolify data directory
tar czf /tmp/coolify-backup-$(date +%Y%m%d).tar.gz /data/coolify/
```

This includes the Coolify database (PostgreSQL), SSH keys, and configuration. Store it off-server.

### Restore test

Run a restore drill quarterly. A backup you have never restored is not a backup.

---

## 11. DNS and Cloudflare

### Basic DNS setup

For each domain you host:

1. Log in to your DNS provider (or Cloudflare if you transferred the domain)
2. Add an A record: `@` or your subdomain pointing to your VPS IP
3. TTL: 300 seconds (5 minutes) for fast propagation during setup; increase to 3600 after stable

Example records:

```
Type  Name              Value            TTL
A     @                 203.0.113.42     300
A     www               203.0.113.42     300
A     api               203.0.113.42     300
A     coolify           203.0.113.42     300
```

### Cloudflare proxy (orange cloud)

Cloudflare can proxy traffic through their network, providing:
- DDoS protection
- WAF (Web Application Firewall)
- CDN caching for static assets
- Hides your origin IP

**Enable proxying:** toggle the orange cloud on the DNS record.

**When to use proxied:**
- Public-facing websites and APIs
- Any service you want DDoS protection for

**When to leave DNS-only (grey cloud):**
- Services that use non-HTTP protocols (Postgres, Redis, custom TCP)
- Services where Cloudflare caching would cause problems (some WebSocket apps need `Full (Strict)` SSL mode and no caching)
- The Coolify dashboard itself (Cloudflare can interfere with WebSocket connections; use grey cloud for the Coolify domain)

### Cloudflare SSL mode

When proxied, set SSL mode to `Full (Strict)` in Cloudflare SSL/TLS settings:
- `Flexible`: Cloudflare to origin is HTTP. Do not use. Your traffic is unencrypted to your server.
- `Full`: Cloudflare to origin is HTTPS with any cert. Better, but accepts self-signed certs.
- `Full (Strict)`: Cloudflare to origin is HTTPS with a valid cert. Use this. Let's Encrypt certs from Coolify/Traefik satisfy this requirement.

### Orange cloud gotchas

If Traefik/Let's Encrypt is failing cert issuance and you have Cloudflare proxy enabled:
- Cloudflare can intercept the ACME HTTP-01 challenge
- Use Cloudflare DNS-01 challenge instead (requires a Cloudflare API token and Traefik configuration)
- Or temporarily disable the proxy during initial cert issuance

---

## 12. Cost optimization

### Right-sizing your VPS

Monitor actual resource usage before deciding to upgrade.

```bash
# CPU and memory snapshot
top -b -n 1 | head -20

# Memory usage
free -h

# Per-container stats
docker stats --no-stream
```

If your 4 vCPU VPS runs at 5% CPU average and 20% memory, you are over-provisioned. Downsize to save money.

### What a 96 GB disk VPS actually runs

The Mad House reference VPS runs comfortably with:
- Coolify + Traefik + Soketi + Redis + Postgres (Coolify's own stack): ~2 GB RAM
- 8 application containers: ~4-6 GB RAM total
- Database volumes: ~5 GB on disk
- Docker images and build cache: ~10-15 GB on disk
- Remaining: headroom for bursts and logs

Total active RAM: ~8-10 GB. A 16 GB RAM VPS handles this with room to spare.

### Cost comparison for 8 services

| Approach | Monthly cost (estimate) |
|---|---|
| Self-hosted VPS (8 GB RAM) | $15-25 |
| Render (8 services, hobby) | $56+ (7 x $7 + database) |
| Railway (8 services) | $40-80 (usage-based) |
| Heroku (8 dynos, basic) | $56+ (8 x $7) |
| Vercel + PlanetScale + Upstash | $30-60+ (separate per-service billing) |

Self-hosting wins on cost at scale. The break-even versus Render is around 3-4 always-on services.

### Where not to cut corners

- Do not skimp on RAM. Containers OOM-killed mid-request causes silent data corruption in some databases.
- Do not use the cheapest provider for production without checking their SLA. Hetzner and Hostinger both have good uptime records.
- Do not skip backups to save storage costs. A $5/month Cloudflare R2 bucket can hold years of database dumps.

---

## Sequence checklist

Use this as a run-book for a fresh VPS:

- [ ] VPS provisioned, Ubuntu 22.04/24.04
- [ ] SSH in as root, create non-root user
- [ ] Copy SSH public key to non-root user
- [ ] Disable password auth and root SSH login
- [ ] Install fail2ban
- [ ] `apt update && apt upgrade -y`
- [ ] Configure UFW (deny incoming, allow SSH/80/443)
- [ ] Install Tailscale, join Tailnet
- [ ] Install Docker (official method)
- [ ] Install Coolify
- [ ] Configure Coolify domain and SSL
- [ ] Configure DNS A records
- [ ] Set up log rotation in `/etc/docker/daemon.json`
- [ ] Set up backup cron jobs
- [ ] Configure uptime monitoring
- [ ] Enable Coolify Sentinel

---

## Related

- [coolify-playbook](https://github.com/madebymadhouse/coolify-playbook) -- detailed Coolify configuration guide
- [agent-ops-playbook](https://github.com/madebymadhouse/agent-ops-playbook) -- running agentic workflows on your infra

---

## Contributing

Pull requests welcome. If your provider has specific quirks or you have a better pattern for any of these steps, open a PR.
