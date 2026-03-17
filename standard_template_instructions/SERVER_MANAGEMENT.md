# Server Management Guide

**IMPORTANT**: This document is for managing the production Linux server deployment at options-edge.johnapplications.com. This is SEPARATE from the Windows development environment described in WINDOWS_DEVELOPMENT.md.

## Critical Lesson: Always Use Systemd Services

### ❌ MISTAKE: Bypassing systemd service management

**What I did wrong:**
```bash
# When "systemctl restart options-edge-seller-backend" failed with "Unit not found"
# I attempted to manually kill the process and restart outside of systemd:
sudo kill <PID> && /opt/options-edge-seller/.venv/bin/python backend/run.py
```

**Why this is wrong:**
- Bypasses proper service management
- Loses systemd benefits (auto-restart, logging, monitoring)
- Can create inconsistent state with systemd still trying to manage the process
- May create orphaned processes or port conflicts
- No graceful shutdown handling

### ✅ CORRECT: Find and use the actual service name

**Correct approach when a service name is wrong:**
```bash
# 1. Search for the correct service name
systemctl list-units --type=service --all | grep -i "option"

# 2. Use the correct service name
sudo systemctl restart options-backend
```

## Server Architecture

### Production Deployment Stack

**Server:** Linux (Ubuntu) at `john-applications` (VPS)
**Domain:** options-edge.johnapplications.com
**API Domain:** api.options-edge.johnapplications.com
**SSL:** Let's Encrypt certificates (auto-renewal via certbot)
**Reverse Proxy:** Nginx with HTTP/2 and SSL termination
**Application User:** www-data
**Project Root:** `/opt/options-edge-seller`
**Virtual Environment:** `/opt/options-edge-seller/.venv`

### Service Architecture

The application runs as 4 systemd services:

| Service | Description | Entry Point | Port | User |
|---------|-------------|-------------|------|------|
| `options-backend.service` | FastAPI backend API | `backend/run.py` | 8010 | www-data |
| `options-frontend.service` | Next.js production server | `frontend/.next/standalone/server.js` | 3010 | www-data |
| `iv-worker.service` | Background IV processing worker | `backend/app/workers/iv_worker.py` | - | www-data |
| `iv-snapshot.service` | Daily IV snapshot (timer-triggered) | `backend/scripts/iv_snapshot_daily.py` | - | www-data |

**Timer:**
- `iv-snapshot.timer` - Triggers snapshot Mon-Fri at 21:30 UTC (after market close)

### Network Flow

```
Internet (HTTPS:443)
  ↓
Nginx Reverse Proxy
  ├─ options-edge.johnapplications.com → localhost:3010 (Frontend)
  └─ api.options-edge.johnapplications.com → localhost:8010 (Backend API)
```

## Service Management Commands

### Service Files Location
All service files are in `/etc/systemd/system/`:
- `/etc/systemd/system/options-backend.service`
- `/etc/systemd/system/options-frontend.service`
- `/etc/systemd/system/iv-worker.service`
- `/etc/systemd/system/iv-snapshot.service`
- `/etc/systemd/system/iv-snapshot.timer`

### Essential Commands

**List all options-related services:**
```bash
systemctl list-units --type=service | grep -E "(options|iv-)"
```

**Check service status:**
```bash
sudo systemctl status options-backend
sudo systemctl status options-frontend
sudo systemctl status iv-worker
sudo systemctl status iv-snapshot
```

**Restart services (after code changes):**
```bash
# Backend only (most common for Python changes)
sudo systemctl restart options-backend

# Frontend (after rebuilding Next.js)
sudo systemctl restart options-frontend

# IV Worker (after worker logic changes)
sudo systemctl restart iv-worker

# All application services
sudo systemctl restart options-backend options-frontend iv-worker
```

**Stop/Start services:**
```bash
sudo systemctl stop options-backend
sudo systemctl start options-backend
```

**Enable/disable auto-start on boot:**
```bash
sudo systemctl enable options-backend
sudo systemctl disable options-backend
```

**Reload systemd after editing service files:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart <service-name>
```

### Timer Management

**Check timer status:**
```bash
systemctl list-timers --all | grep iv-snapshot
```

**Manually trigger snapshot (for testing):**
```bash
sudo systemctl start iv-snapshot.service
```

**Enable/disable timer:**
```bash
sudo systemctl enable iv-snapshot.timer
sudo systemctl disable iv-snapshot.timer
```

## Logging & Troubleshooting

### Log Architecture

All services output to **systemd journal** (journald) with structured logging.

**SyslogIdentifiers:**
- `options-backend` - Backend API logs
- `options-frontend` - Frontend logs
- `iv-worker` - Background worker logs
- `iv-snapshot` - Daily snapshot logs

**Structured Log Format:**
```
2025-11-24 13:45:45,164 INFO [module.name] [EVENT-TAG] key=value key2=value2
```

**Common Event Tags:**
- `[SERVICE-START]` - Service startup
- `[IV-SNAPSHOT-RUN]` - Snapshot execution
- `[IV-SNAPSHOT-COMPLETE]` - Snapshot finished
- `[IV-BACKFILL]` - Historical IV backfill operations
- `[IV-REPAIR]` - Gap repair operations
- `[IV-JOB]` - Job queue operations
- `status=error` - Error conditions

### Essential Log Queries

**View recent logs for a service:**
```bash
# Last 50 lines
sudo journalctl -u options-backend -n 50

# Follow (tail -f equivalent)
sudo journalctl -u options-backend -f

# Last 30 minutes
sudo journalctl -u options-backend --since "30 minutes ago"

# Specific time range
sudo journalctl -u options-backend --since "2025-11-24 10:00" --until "2025-11-24 11:00"
```

**Common troubleshooting queries:**

```bash
# Did yesterday's snapshots run successfully?
sudo journalctl -u iv-snapshot.service --since yesterday | grep 'IV-SNAPSHOT-RUN'
sudo journalctl -u iv-snapshot.service --since yesterday | grep 'IV-SNAPSHOT-COMPLETE'

# Which jobs failed last night?
sudo journalctl -u iv-worker.service --since yesterday | grep 'status=error'

# How many tickers still have repair gaps?
sudo journalctl -u iv-worker.service --since yesterday | grep 'IV-REPAIR'

# Did backfill for specific ticker finish?
sudo journalctl -u iv-worker.service --since yesterday | grep 'IV-BACKFILL' | grep 'AAPL'

# Check worker startup/restart events
sudo journalctl -u iv-worker.service --since today | grep 'Worker'

# Backend API errors
sudo journalctl -u options-backend --since "1 hour ago" | grep -i error

# Frontend crashes or errors
sudo journalctl -u options-frontend --since today | grep -E "(error|Error|ERROR|exception)"
```

**Check disk usage and retention:**
```bash
# How much space journals use
sudo journalctl --disk-usage

# Vacuum (clean old logs) - keep last 7 days
sudo journalctl --vacuum-time=7d

# Vacuum - keep only 500MB
sudo journalctl --vacuum-size=500M
```

### Service-Specific Troubleshooting

**Backend won't start:**
```bash
# Check detailed status
sudo systemctl status options-backend -l

# Check recent errors
sudo journalctl -u options-backend --since "5 minutes ago"

# Verify Python environment
sudo -u www-data /opt/options-edge-seller/.venv/bin/python --version

# Check if port 8010 is already bound
sudo lsof -i :8010
```

**Frontend won't start:**
```bash
# Check if standalone build exists
ls -la /opt/options-edge-seller/frontend/.next/standalone/

# Verify Node.js version
node --version

# Check if port 3010 is bound
sudo lsof -i :3010
```

**Frontend loads but has no CSS/styling:**

If the frontend loads but appears unstyled:

```bash
# Check if static assets exist
ls -la /opt/options-edge-seller/frontend/.next/standalone/.next/static

# If missing, rebuild with proper asset copying
cd /opt/options-edge-seller
./scripts/rebuild-frontend.sh
```

This is a common issue with Next.js standalone builds - the static assets must be manually copied.

**IV Worker stuck or not processing:**
```bash
# Check current worker status
sudo journalctl -u iv-worker.service -n 100 | tail -20

# Look for database locks
sqlite3 /opt/options-edge-seller/backend/data/options.db "PRAGMA integrity_check;"

# Check job queue status (via backend API)
curl http://localhost:8010/api/iv/jobs
```

**Snapshot didn't run:**
```bash
# Check timer status
systemctl list-timers | grep iv-snapshot

# Check timer logs
sudo journalctl -u iv-snapshot.timer --since yesterday

# Check last snapshot attempt
sudo journalctl -u iv-snapshot.service --since "2 days ago"
```

## Nginx Configuration

**Config file:** `/etc/nginx/sites-available/options-edge-seller`
**Symlink:** `/etc/nginx/sites-enabled/options-edge-seller`

**Test config after changes:**
```bash
sudo nginx -t
```

**Reload nginx (without dropping connections):**
```bash
sudo systemctl reload nginx
```

**Restart nginx (drops connections):**
```bash
sudo systemctl restart nginx
```

**View nginx access logs:**
```bash
sudo tail -f /var/log/nginx/access.log
```

**View nginx error logs:**
```bash
sudo tail -f /var/log/nginx/error.log
```

## SSL Certificate Management

**Certificates location:** `/etc/letsencrypt/live/options-edge.johnapplications.com/`

**Check certificate expiration:**
```bash
sudo certbot certificates
```

**Renew certificates manually:**
```bash
sudo certbot renew
```

**Certificate auto-renewal:**
- Certbot installs a systemd timer: `certbot.timer`
- Auto-renewal happens twice daily
- Check status: `systemctl list-timers | grep certbot`

**After certificate renewal:**
```bash
sudo systemctl reload nginx
```

## Common Operational Tasks

### Deploying Code Changes

**CRITICAL PRINCIPLE:** Before starting any deployment, **ALWAYS ask the user which part of the codebase changed**:
- "What changed in this update - frontend, backend, or both?"
- "Does this include any database schema changes?"
- "Are there new dependencies to install?"

**If the user's request is ambiguous** (e.g., "deploy the latest code"), ask for clarification rather than assuming.

#### Why Frontend Rebuild is Different from Backend

**Backend (Python):**
- Python is **interpreted** - files are loaded directly at runtime
- **Restart = Deploy**: Just `systemctl restart` picks up the new code
- No build step needed

**Frontend (Next.js):**
- Next.js is **compiled/bundled** - TypeScript/React source code gets compiled into optimized JavaScript during build
- Production server serves **pre-compiled bundles** from `.next/standalone/`
- **Restart alone does NOT deploy new code** - it continues serving old bundles
- **Must rebuild** to compile new source code into new bundles

**Common mistake:** Restarting services without rebuilding frontend. This causes the site to serve old frontend code even though git is up-to-date.

#### Deployment Decision Tree

Use this decision tree to choose the right deployment workflow:

```
┌─────────────────────────────────────────┐
│ What changed in the git pull?          │
└─────────────────────────────────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
┌─────────┐   ┌──────────┐   ┌─────────────┐
│Frontend │   │ Backend  │   │  Database   │
│  Only   │   │   Only   │   │  Migration  │
└─────────┘   └──────────┘   └─────────────┘
    │               │               │
    ▼               ▼               ▼
Use Workflow     Use Workflow   Use Full Migration
    #1              #2           Workflow (#3)
```

**When to use each:**
- **Workflow #1 (Frontend Only):** Changes only to `frontend/` directory, no backend changes
- **Workflow #2 (Backend Only):** Changes only to `backend/` directory (Python files, scripts), no database schema changes
- **Workflow #3 (Full Migration):** Any database schema changes, new Alembic migrations, or when uncertain

#### Workflow #1: Frontend-Only Deployment

**Use when:** Only `frontend/` files changed (components, pages, styles, etc.)

**Time:** ~30-40 seconds

```bash
# Navigate to project root
cd /opt/options-edge-seller

# Rebuild and restart frontend (script handles everything)
./scripts/rebuild-frontend.sh

# Verify frontend is running
sudo systemctl status options-frontend --no-pager | grep "Active"
```

**What the script does:**
1. Runs `pnpm build` to compile TypeScript/React
2. Copies static assets to standalone build
3. Sets proper ownership (www-data)
4. Restarts the frontend service

**No need to:**
- Stop/restart backend services
- Run database migrations
- Backup database

---

#### Workflow #2: Backend-Only Deployment

**Use when:** Only `backend/` files changed (Python code, services, routes) AND no database schema changes

**Time:** ~5-10 seconds

```bash
# Navigate to project root
cd /opt/options-edge-seller

# Restart backend services to pick up new Python code
sudo systemctl restart options-backend iv-worker

# Verify services are running
sudo systemctl status options-backend iv-worker --no-pager | grep "Active"

# Check for errors in logs
sudo journalctl -u options-backend -n 20 --no-pager
sudo journalctl -u iv-worker -n 20 --no-pager
```

**No need to:**
- Rebuild frontend
- Run database migrations (unless schema changed)
- Backup database (unless schema changed)

---

#### Workflow #3: Full Deployment with Database Migrations

**Use when:**
- Database schema changes (new Alembic migrations)
- Changes to both frontend and backend
- New dependencies in `requirements.txt` or `package.json`
- **When uncertain** about what changed

**Time:** ~60-90 seconds

**IMPORTANT:** Commands must be run sequentially as shown. Do NOT chain with `&&` across directory changes.

```bash
# Navigate to project root
cd /opt/options-edge-seller

# ========================================
# STEP 1: STOP DATABASE-TOUCHING SERVICES
# ========================================
sudo systemctl stop options-backend iv-worker iv-snapshot.service iv-snapshot.timer

# Verify services stopped
systemctl status options-backend iv-worker iv-snapshot.service iv-snapshot.timer --no-pager | grep "Active"
```

```bash
# ========================================
# STEP 2: BACKUP DATABASE
# ========================================
# Create backup (run from project root)
cd /opt/options-edge-seller/backend/data
cp options.db "options.db.pre_migration_$(date +%Y%m%d_%H%M)"
cd ../..
```

```bash
# Prune old backups - keep only 3 most recent (run from project root)
cd /opt/options-edge-seller/backend/data
ls -t options.db.pre_migration_* | tail -n +4 | xargs rm -f
ls -lt options.db.pre_migration_* | head -3
cd ../..
```

```bash
# ========================================
# STEP 3: UPDATE DEPENDENCIES
# ========================================
cd /opt/options-edge-seller

# Backend dependencies
source .venv/bin/activate
pip install -r requirements.txt
deactivate

# Frontend dependencies
cd frontend
pnpm install
cd ..
```

```bash
# ========================================
# STEP 4: RUN DATABASE MIGRATIONS
# ========================================
cd /opt/options-edge-seller/backend
../.venv/bin/alembic upgrade head
cd ..
```

```bash
# ========================================
# STEP 5: REBUILD FRONTEND
# ========================================
cd /opt/options-edge-seller
./scripts/rebuild-frontend.sh
```

```bash
# ========================================
# STEP 6: RESTART ALL SERVICES
# ========================================
# Frontend already restarted by rebuild script
sudo systemctl start options-backend iv-worker iv-snapshot.timer
```

```bash
# ========================================
# STEP 7: VERIFY ALL SERVICES
# ========================================
sudo systemctl status options-backend options-frontend iv-worker iv-snapshot.timer --no-pager | grep -E "(Loaded|Active)"

# Check for errors
sudo journalctl -u options-backend -n 20 --no-pager
sudo journalctl -u options-frontend -n 20 --no-pager
sudo journalctl -u iv-worker -n 20 --no-pager
```

**Safety features:**
- Database backed up before migrations
- Services stopped during schema changes
- All dependencies updated
- Full verification at end

#### Frontend Rebuild Process (Technical Details)

The `./scripts/rebuild-frontend.sh` helper script handles the complete process:

1. Runs `pnpm build` to compile TypeScript/React into optimized bundles
2. Copies static assets to standalone build (CSS, JS, images)
3. Sets proper ownership (`www-data`)
4. Restarts the frontend service

**Manual rebuild (only if script fails):**

```bash
cd /opt/options-edge-seller/frontend

# Build the frontend
pnpm build

# Copy static assets (CRITICAL for Next.js standalone builds)
cp -r .next/static .next/standalone/.next/static

# Fix ownership
sudo chown -R www-data:www-data .next/standalone/.next/static

# Restart service
sudo systemctl restart options-frontend
```

**Important:** The static assets copy step is **required** for Next.js standalone builds. Without it, CSS and JavaScript files won't load.

---

#### Checking for Pending Database Migrations

**Before using Workflow #2 (Backend Only)**, verify there are no pending migrations:

```bash
cd /opt/options-edge-seller/backend
../.venv/bin/alembic current    # Shows current database version
../.venv/bin/alembic heads      # Shows latest migration version
```

**If the versions match:** No pending migrations, safe to use Workflow #2

**If the versions differ:** Pending migrations exist, **must use Workflow #3** (Full Deployment)

### Database Operations

**Database location:** `/opt/options-edge-seller/backend/data/options.db`

**Backup database:**
```bash
sqlite3 /opt/options-edge-seller/backend/data/options.db ".backup /opt/options-edge-seller/backend/data/options_backup_$(date +%Y%m%d).db"
```

**Check database size:**
```bash
du -h /opt/options-edge-seller/backend/data/options.db
```

**Vacuum database (reclaim space):**
```bash
sqlite3 /opt/options-edge-seller/backend/data/options.db "VACUUM;"
```

**Optimize database performance:**
```bash
# Run these periodically for better query performance
sqlite3 /opt/options-edge-seller/backend/data/options.db "VACUUM; ANALYZE;"
```

#### Automated Backup Strategy

**Create Daily Backup Script:**

Create `/etc/cron.daily/options-backup`:

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/options-edge-seller"
DATA_DIR="/opt/options-edge-seller/backend/data"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR
cp $DATA_DIR/options.db $BACKUP_DIR/options.db.$DATE
gzip $BACKUP_DIR/options.db.$DATE

# Keep only last 30 days
find $BACKUP_DIR -name "options.db.*" -mtime +30 -delete
```

Make executable:
```bash
sudo chmod +x /etc/cron.daily/options-backup
```

### Monitoring System Resources

**Check memory usage:**
```bash
free -h
```

**Check disk space:**
```bash
df -h
```

**Check process resource usage:**
```bash
# All options services
ps aux | grep -E "(python.*backend|node.*server)" | grep -v grep

# Detailed systemd resource stats
systemctl status options-backend --no-pager -l
```

**Check network connections:**
```bash
# Who's connected to the backend API
sudo lsof -i :8010

# Who's connected to the frontend
sudo lsof -i :3010
```

## Environment Variables

**Backend .env location:** `/opt/options-edge-seller/.env`

**Required variables:**
```
POLYGON_API_KEY=your_key_here
```

**After changing .env:**
```bash
# Restart services that need the variables
sudo systemctl restart options-backend iv-worker
```

## Firewall (if UFW is enabled)

**Check firewall status:**
```bash
sudo ufw status
```

**Required open ports:**
- 80 (HTTP - redirects to HTTPS)
- 443 (HTTPS)
- 22 (SSH)

**Configure firewall:**
```bash
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## Performance Tuning

### Nginx Worker Connections

For better concurrent connection handling, edit `/etc/nginx/nginx.conf`:

```nginx
worker_processes auto;
events {
    worker_connections 2048;
}
```

Then reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Database Optimization

Run these commands periodically for optimal database performance:

```bash
# Reclaim space and update query planner statistics
sqlite3 /opt/options-edge-seller/backend/data/options.db "VACUUM; ANALYZE;"
```

## Server Reboot

If you need to reboot the server:

```bash
sudo reboot
```

All services (options-backend, options-frontend, iv-worker) are configured with `enable` to auto-start on boot.

**After reboot, verify services:**
```bash
# Check all services started successfully
sudo systemctl status options-backend options-frontend iv-worker

# Check timer is active
systemctl list-timers | grep iv-snapshot
```

## Quick Reference Checklist

**Deploying code changes?**
1. ✓ Ask user what changed (frontend/backend/both/migrations)
2. ✓ Choose correct workflow (#1 Frontend, #2 Backend, #3 Full)
3. ✓ For backend changes: check for pending migrations first
4. ✓ Verify services are running after deployment
5. ✓ Check logs for errors

**Service won't start?**
1. ✓ Check service status: `sudo systemctl status <service>`
2. ✓ Check recent logs: `sudo journalctl -u <service> -n 50`
3. ✓ Check port availability: `sudo lsof -i :<port>`
4. ✓ Verify file permissions in project directory
5. ✓ Check .env file exists and has correct variables

**Code deployed but not running?**
1. ✓ Did you rebuild frontend (if frontend changed)?
2. ✓ Did you restart the correct services?
3. ✓ Did you reload systemd if service file changed?
4. ✓ Are logs showing the new code running?
5. ✓ Did you clear browser cache for frontend changes?

**Performance issues?**
1. ✓ Check memory: `free -h`
2. ✓ Check disk: `df -h`
3. ✓ Check service resource usage: `systemctl status <service>`
4. ✓ Check database size and vacuum if needed
5. ✓ Check journal disk usage: `sudo journalctl --disk-usage`

**SSL certificate issues?**
1. ✓ Check expiration: `sudo certbot certificates`
2. ✓ Check nginx config: `sudo nginx -t`
3. ✓ Check nginx error logs: `sudo tail /var/log/nginx/error.log`
4. ✓ Verify certbot timer: `systemctl list-timers | grep certbot`

## Resources

- Project root: `/opt/options-edge-seller`
- Service files: `/etc/systemd/system/`
- Nginx config: `/etc/nginx/sites-available/options-edge-seller`
- SSL certs: `/etc/letsencrypt/live/options-edge.johnapplications.com/`
- Database: `/opt/options-edge-seller/backend/data/options.db`
- Logs: Access via `journalctl` (no separate log files)
- Application user: `www-data`
- Virtual environment: `/opt/options-edge-seller/.venv`
