# VPS Migration Plan: Vultr → Hostinger KVM 2

**Source:** dash.t9i.ai (Vultr, 1.9GB RAM)  
**Target:** Hostinger KVM 2 (8GB RAM)  
**Date:** 2026-02-09  
**Status:** New VPS locked down, OpenClaw installed, Mission Control pending

---

## PHASE 1: Pre-Migration Checklist

### ✅ Source VPS (Current) - Verify Before Migration

**Critical Files to Backup:**

1. **OpenClaw Configuration**
   - [ ] `/home/node/.openclaw/openclaw.json` - Main config
   - [ ] `/home/node/.openclaw/agents/main/agent/auth-profiles.json` - API keys
   - [ ] `/home/node/.openclaw/config.yaml` - If exists

2. **Workspace Files**
   - [ ] `/home/node/.openclaw/workspace/` - All project files
   - [ ] `SOUL.md` - Personality config
   - [ ] `IDENTITY.md` - Identity config
   - [ ] `USER.md` - User profile
   - [ ] `MEMORY.md` - Long-term memory
   - [ ] `AGENTS.md` - Squad config
   - [ ] `HEARTBEAT.md` - Periodic tasks
   - [ ] `SCHEDULED_TASKS.md` - Task schedule
   - [ ] `memory/*.md` - Daily journals
   - [ ] `BUGS.md` - Bug tracking
   - [ ] `INFRASTRUCTURE-ROADMAP.md` - Infrastructure plans
   - [ ] `TASK-API-SPEC.md` - API specifications
   - [ ] `REALTIME-CHAT-FIX-SPEC.md` - Chat implementation
   - [ ] Any other .md files

3. **Tools & Scripts**
   - [ ] `/home/node/.openclaw/mission-control-chat.sh`
   - [ ] `/home/node/.openclaw/mission-control-approve.sh`
   - [ ] `/home/node/.openclaw/workspace/fetch-url.js`
   - [ ] `/home/node/.openclaw/workspace/fetch-x-tweet.sh`
   - [ ] Any other custom scripts

4. **Mission Control (If Recoverable)**
   - [ ] `/opt/mission-control/.env` - Environment variables
   - [ ] `/opt/mission-control/data/mission-control.db` - SQLite database
   - [ ] Docker compose files

5. **Docker Volumes**
   - [ ] `openclaw_openclaw_config` - OpenClaw config volume
   - [ ] Any other Docker volumes

---

## PHASE 2: Migration Steps

### Step 1: Create Clean Backups on Source

```bash
# SSH to source VPS
ssh dash.t9i.ai

# Create backup directory
mkdir -p /tmp/migration-backup-$(date +%Y%m%d)
BACKUP_DIR="/tmp/migration-backup-$(date +%Y%m%d)"

# 1. Backup OpenClaw config
cp /home/node/.openclaw/openclaw.json $BACKUP_DIR/
cp /home/node/.openclaw/agents/main/agent/auth-profiles.json $BACKUP_DIR/

# 2. Backup workspace (exclude node_modules, .git, etc.)
tar czf $BACKUP_DIR/workspace.tar.gz -C /home/node/.openclaw workspace \
  --exclude='node_modules' --exclude='.git' --exclude='*.log'

# 3. Backup Mission Control (if accessible)
if [ -f /opt/mission-control/data/mission-control.db ]; then
  cp /opt/mission-control/data/mission-control.db $BACKUP_DIR/
  cp /opt/mission-control/.env $BACKUP_DIR/mission-control.env
fi

# 4. Backup Docker volumes
docker run --rm -v openclaw_openclaw_config:/config -v $BACKUP_DIR:/backup alpine \
  tar czf /backup/openclaw-config.tar.gz -C /config .

# 5. Create manifest
cat > $BACKUP_DIR/MANIFEST.txt << 'EOF'
Migration Backup Manifest
=========================
Date: $(date)
Source: dash.t9i.ai
Target: Hostinger KVM 2

Files Included:
- openclaw.json (main config)
- auth-profiles.json (API keys)
- workspace.tar.gz (all project files)
- mission-control.db (task database)
- mission-control.env (MC config)
- openclaw-config.tar.gz (Docker volume)

Prerequisites on Target:
- Docker installed
- Docker Compose installed
- Cloudflared installed (for tunnel)
- Node.js installed (for OpenClaw)
- Git installed
EOF

# 6. Compress everything
tar czf ~/migration-$(date +%Y%m%d).tar.gz -C /tmp migration-backup-$(date +%Y%m%d)

# 7. Show backup location
ls -lh ~/migration-$(date +%Y%m%d).tar.gz
```

### Step 2: Transfer to Target VPS

```bash
# On target VPS (Hostinger)
# Create receiving directory
mkdir -p /tmp/migration

# Option A: Direct scp from source
scp dash.t9i.ai:~/migration-20260209.tar.gz /tmp/migration/

# Option B: If direct scp fails, use intermediate
# Download to local machine first, then upload to Hostinger

# Extract backup
cd /tmp/migration
tar xzf migration-20260209.tar.gz
```

### Step 3: Install OpenClaw on Target

```bash
# 1. Install prerequisites
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs docker.io docker-compose-plugin

# 2. Install OpenClaw
npm install -g openclaw

# 3. Initialize OpenClaw
openclaw onboard

# 4. Stop the default OpenClaw to restore config
openclaw gateway stop
```

### Step 4: Restore Configuration

```bash
# 1. Restore OpenClaw config
cp /tmp/migration/migration-backup-20260209/openclaw.json /home/node/.openclaw/
cp /tmp/migration/migration-backup-20260209/auth-profiles.json /home/node/.openclaw/agents/main/agent/

# 2. Restore workspace
tar xzf /tmp/migration/migration-backup-20260209/workspace.tar.gz -C /home/node/.openclaw/

# 3. Fix permissions
sudo chown -R node:node /home/node/.openclaw
chmod 600 /home/node/.openclaw/agents/main/agent/auth-profiles.json

# 4. Start OpenClaw
openclaw gateway start
```

### Step 5: Setup Mission Control

```bash
# 1. Create Mission Control directory
mkdir -p /opt/mission-control
cd /opt/mission-control

# 2. Clone or copy Mission Control code
# (Claude Code will provide this)

# 3. Restore database (if available)
mkdir -p data
cp /tmp/migration/migration-backup-20260209/mission-control.db data/

# 4. Setup environment
cp /tmp/migration/migration-backup-20260209/mission-control.env .env

# 5. Update environment for new VPS
# Edit .env to update:
# - OPENCLAW_GATEWAY_URL=http://127.0.0.1:18789 (or new IP)
# - Any IP-specific configs

# 6. Start Mission Control
docker-compose up -d
```

### Step 6: Setup Cloudflare Tunnel

```bash
# 1. Install cloudflared
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# 2. Authenticate (if new tunnel needed)
cloudflared tunnel login

# 3. Create or restore tunnel config
# Copy tunnel credentials from old VPS if possible
# Or create new tunnel and update DNS

# 4. Start tunnel
cloudflared tunnel run
```

---

## PHASE 3: Post-Migration Verification

### Critical Tests

- [ ] OpenClaw Gateway responding on port 18789
- [ ] Web UI accessible via Cloudflare tunnel
- [ ] T9I can connect and respond
- [ ] Real-time chat working
- [ ] Mission Control accessible
- [ ] Task board loading
- [ ] All workspace files present
- [ ] Scheduled tasks intact
- [ ] Memory settings correct (8GB available!)

### Performance Tests

- [ ] Memory usage under 2GB (should have 6GB headroom now!)
- [ ] CPU usage reasonable
- [ ] Response times <100ms
- [ ] Can install Playwright (memory test)
- [ ] Can download QMD models (memory test)

---

## PHASE 4: Cleanup

### Old VPS

- [ ] Verify new VPS working for 24 hours
- [ ] Final backup from old VPS
- [ ] Cancel Vultr subscription
- [ ] Archive old VPS data

### New VPS

- [ ] Setup automated backups
- [ ] Configure monitoring
- [ ] Document new setup
- [ ] Update any IP-specific configurations
- [ ] Test failover procedures

---

## Critical Reminders

⚠️ **DO NOT** delete old VPS until new one is 100% working for 24+ hours  
⚠️ **DO** test real-time chat immediately after migration  
⚠️ **DO** verify all API keys work on new VPS  
⚠️ **DO** update DNS records if IP changes  
⚠️ **DO** keep backups of both systems during transition  

---

## New Capabilities (8GB RAM!)

Once migrated, you can finally:
- ✅ Install Playwright (needs ~1GB)
- ✅ Download QMD reranker model (1.28GB)
- ✅ Run concurrent agents without memory issues
- ✅ Host local LLM (optional)
- ✅ Process images/media
- ✅ Heavy web scraping workflows

---

**This is your escape from 1.9GB prison! 🚀**
