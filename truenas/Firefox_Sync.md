# Self-Hosting Firefox Sync on TrueNAS SCALE

A comprehensive guide documenting the journey of setting up a self-hosted Firefox Sync server (syncstorage-rs) on TrueNAS SCALE.

## ⚠️ Important Disclaimer

**This setup is NOT production-ready.** After extensive testing and troubleshooting:

- ❌ High maintenance overhead (10-60 min rebuilds for updates)
- ❌ Fragile database configuration requiring manual fixes
- ❌ Not visible in TrueNAS Apps UI
- ❌ Official Mozilla Docker images are broken/outdated
- ⚠️ Requires command-line management and monitoring

**Recommendation:** Unless you enjoy tinkering and command-line administration, use Mozilla's official free sync service instead.

---

## Table of Contents

- [Overview](#overview)
- [Why Self-Host Firefox Sync?](#why-self-host-firefox-sync)
- [Prerequisites](#prerequisites)
- [The Problem: Official Images Don't Work](#the-problem-official-images-dont-work)
- [The Solution: Build from Source](#the-solution-build-from-source)
- [Installation Guide](#installation-guide)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Known Issues](#known-issues)
- [Maintenance](#maintenance)
- [Alternative Solutions](#alternative-solutions)

---

## Overview

Firefox Sync allows you to synchronize bookmarks, history, passwords, and open tabs across devices. While Mozilla provides this service for free, some users prefer to self-host for privacy or control reasons.

This guide documents the process of setting up `syncstorage-rs` (Mozilla's Rust-based sync server) on TrueNAS SCALE using Docker Compose.

**What Works:**
- ✅ Sync server running on TrueNAS
- ✅ Desktop Firefox syncing successfully
- ✅ Data persistence across restarts
- ✅ Basic functionality confirmed

**What Doesn't Work:**
- ❌ TrueNAS Apps UI integration
- ❌ Official Mozilla Docker images
- ❌ Simple setup process
- ⚠️ Multi-device sync may require HTTPS

---

## Why Self-Host Firefox Sync?

**Pros:**
- Complete control over your data
- Privacy from Mozilla's servers
- Learning experience with containerization
- Local network sync (faster in theory)

**Cons:**
- Complex setup and maintenance
- No official support for self-hosting
- Requires technical knowledge
- Time-consuming troubleshooting
- Updates require rebuilding containers

---

## Prerequisites

### Required:
- TrueNAS SCALE (tested on current version)
- Basic command-line knowledge
- SSH access to TrueNAS
- A dataset for persistent storage
- 10-60 minutes for initial build
- Patience for troubleshooting

### Optional but Recommended:
- Domain name for HTTPS
- Nginx Proxy Manager for reverse proxy
- Basic Docker/Docker Compose knowledge

---

## The Problem: Official Images Don't Work

### What We Discovered

The official `mozilla/syncstorage-rs:latest` Docker image has critical issues:
```yaml
# ❌ DON'T USE THIS - IT'S BROKEN
image: mozilla/syncstorage-rs:latest
```

**Issues:**
1. **Severely Outdated:** Image from March 2020 (version 0.2.5)
2. **Wrong Configuration:** Hardcoded for Google Spanner, not MySQL
3. **Broken Tokenserver:** OAuth incompatible with modern Firefox
4. **No MySQL Support:** Environment variables don't override Spanner config

**Error Example:**
```
Services.Common.TokenServerClient WARN Invalid JSON returned by server: Err: Invalid Authorization
```

### Why Official Documentation Fails

Mozilla's documentation assumes you're using their production setup or Google Cloud. The Docker Hub image is unmaintained and incompatible with MySQL-based self-hosting.

---

## The Solution: Build from Source

We use [dan-r's syncstorage-rs-docker](https://github.com/dan-r/syncstorage-rs-docker) repository, which:
- Builds syncstorage-rs from source (version 0.18.3)
- Properly configures MySQL/MariaDB
- Includes working tokenserver setup
- Uses Rust compilation for the latest stable code

---

## Installation Guide

### Step 1: Prepare TrueNAS Storage

Create a dataset for persistent data:
```bash
# Via TrueNAS Shell or SSH
mkdir -p /mnt/your-pool/firefox-sync
cd /mnt/your-pool/firefox-sync
```

### Step 2: Clone the Working Repository
```bash
git clone https://github.com/dan-r/syncstorage-rs-docker.git
cd syncstorage-rs-docker
```

### Step 3: Generate Secrets

You need two 64-character random secrets:
```bash
# Generate SYNC_MASTER_SECRET (64 chars)
openssl rand -hex 32

# Generate METRICS_HASH_SECRET (64 chars)
openssl rand -hex 32
```

Save both outputs - you'll need them in the next step.

### Step 4: Configure Environment Variables
```bash
cp example.env .env
sudo nano .env
```

Edit the `.env` file:
```bash
##########################################
## Firefox SyncStorage RS Configuration ##
##########################################

# Your server URL (use IP for now, domain later with HTTPS)
SYNC_URL=http://192.168.4.177:8000

# MySQL passwords (change these to strong random passwords)
MYSQL_ROOT_PASSWORD=YourStrongRootPassword123!
MYSQL_PASSWORD=YourStrongUserPassword456!

# Master sync key (paste the first 64-char secret you generated)
SYNC_MASTER_SECRET=your_64_character_hex_string_here

# Hashing secret (paste the second 64-char secret you generated)
METRICS_HASH_SECRET=your_other_64_character_hex_string_here
```

**⚠️ Important:** Change the IP address to your TrueNAS IP.

### Step 5: Fix Database Configuration

The default configuration has a mismatch. Edit `docker-compose.yaml`:
```bash
sudo nano docker-compose.yaml
```

Find these lines (around line 25-26):
```yaml
# ❌ WRONG - Database names don't match what MariaDB creates
SYNC_SYNCSTORAGE_DATABASE_URL: mysql://sync:${MYSQL_PASSWORD}@mariadb:3306/syncstorage_rs
SYNC_TOKENSERVER_DATABASE_URL: mysql://sync:${MYSQL_PASSWORD}@mariadb:3306/tokenserver_rs
```

Change to:
```yaml
# ✅ CORRECT - Remove the _rs suffix
SYNC_SYNCSTORAGE_DATABASE_URL: mysql://sync:${MYSQL_PASSWORD}@mariadb:3306/syncstorage
SYNC_TOKENSERVER_DATABASE_URL: mysql://sync:${MYSQL_PASSWORD}@mariadb:3306/tokenserver
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

### Step 6: Build and Start
```bash
# This will take 10-60 minutes on first run
sudo docker compose up -d --build
```

**What's happening:**
1. Downloads Rust compiler and dependencies
2. Clones Mozilla's syncstorage-rs repository
3. Compiles everything from source (this is the slow part)
4. Starts MariaDB and sync server containers

Monitor progress:
```bash
# Watch build logs
sudo docker compose logs -f
```

Press Ctrl+C to stop watching (containers keep running).

### Step 7: Verify Server is Running
```bash
# Check container status
sudo docker compose ps

# Should show:
# firefox_mariadb      Up
# firefox_syncserver   Up (or starting)
```

Test the server:
```bash
curl http://192.168.4.177:8000/__heartbeat__
```

Expected response:
```json
{"status":"Ok","version":"0.18.3","quota":{"enabled":false,"size":0},"database":"Ok"}
```

✅ If you see this, the server is running!

---

## Configuration

### Initialize Tokenserver Database

**Critical Step:** The database tables exist but are empty. You must manually populate them.
```bash
# Connect to MariaDB
sudo docker exec -it firefox_mariadb mariadb -u sync -pYourStrongUserPassword456!
```

At the MariaDB prompt:
```sql
-- Switch to tokenserver database
USE tokenserver;

-- Check if tables are empty (they will be)
SELECT * FROM services;
SELECT * FROM nodes;

-- Populate the tables
DELETE FROM services;
INSERT INTO services (id, service, pattern) VALUES (1, 'sync-1.5', '{node}/1.5/{uid}');

INSERT INTO nodes (id, service, node, capacity, available, current_load, downed, backoff) 
VALUES (1, 1, 'http://192.168.4.177:8000', 10, 10, 0, 0, 0);

-- Verify they're populated
SELECT * FROM services;
SELECT * FROM nodes;

-- Exit
exit
```

**⚠️ Change the IP** in the INSERT statement to match your TrueNAS IP.

### Configure Firefox

1. Open Firefox
2. Type `about:config` in the address bar
3. Click "Accept the Risk and Continue"
4. Search for: `identity.sync.tokenserver.uri`
5. Click the pencil icon to edit
6. Change the value to: `http://192.168.4.177:8000/1.0/sync/1.5`
   - ⚠️ Use your TrueNAS IP
   - ⚠️ Use port **8000** (not 5000)
   - ⚠️ Path is `/1.0/sync/1.5` (not `/token/1.0/sync/1.5`)
7. Restart Firefox
8. Sign in to Firefox Sync with your Mozilla account

### Verify Sync is Working

1. In Firefox, go to `about:sync-log`
2. Look for recent sync activity
3. Successful sync shows:
```
   Sync.Service INFO Starting sync at [timestamp]
   Sync.Resource DEBUG GET success 200 http://192.168.4.177:8000/1.5/[uid]/info/collections
   Sync.Synchronizer INFO Sync completed at [timestamp] after 0.08 secs.
```

4. Test by creating a bookmark - it should sync immediately

---

## Troubleshooting

### Server Won't Start

**Check logs:**
```bash
sudo docker compose logs syncserver | tail -50
```

**Common issues:**

1. **Database connection failed:**
```
   Access denied for user 'sync'@'%' to database 'tokenserver'
```
   
   **Fix:** Manually create the database:
```bash
   sudo docker exec -it firefox_mariadb mariadb -u root -pYourRootPassword
```
```sql
   CREATE DATABASE IF NOT EXISTS tokenserver;
   GRANT ALL PRIVILEGES ON tokenserver.* TO 'sync'@'%';
   FLUSH PRIVILEGES;
   exit
```

2. **Migration errors:**
```
   PROCEDURE UPDATE_165600 already exists
```
   
   **Fix:** Clean database and restart:
```bash
   sudo docker compose down
   sudo rm -rf ./data/
   sudo docker compose up -d
```

### Firefox Can't Connect

**Error: Connection Refused**

Check the URL in `about:config`:
- ❌ Wrong: `http://192.168.4.177:5000/token/1.0/sync/1.5`
- ✅ Correct: `http://192.168.4.177:8000/1.0/sync/1.5`

**Error: 404 Not Found**

Remove `/token` from the path:
- ❌ Wrong: `http://192.168.4.177:8000/token/1.0/sync/1.5`
- ✅ Correct: `http://192.168.4.177:8000/1.0/sync/1.5`

**Error: 500 Internal Server - "Database error: NotFound"**

Tokenserver tables are empty. Follow the [Initialize Tokenserver Database](#initialize-tokenserver-database) section.

**Error: "Invalid Authorization"**

You're likely using the old broken Docker image. Make sure you're building from source using dan-r's repository.

### Check Server Health
```bash
# View all logs
sudo docker compose logs

# View specific container
sudo docker compose logs syncserver

# Follow logs in real-time
sudo docker compose logs -f syncserver

# Check container status
sudo docker compose ps

# Restart containers
sudo docker compose restart

# Stop everything
sudo docker compose down

# Start everything
sudo docker compose up -d
```

### Database Issues

**Connect to MariaDB:**
```bash
sudo docker exec -it firefox_mariadb mariadb -u sync -pYourPassword
```

**Useful SQL commands:**
```sql
-- Show all databases
SHOW DATABASES;

-- Use a specific database
USE tokenserver;

-- Show tables in current database
SHOW TABLES;

-- View service configuration
SELECT * FROM services;

-- View node configuration
SELECT * FROM nodes;

-- Exit
exit
```

---

## Known Issues

### 1. Not Visible in TrueNAS UI

**Issue:** Containers don't appear in TrueNAS Apps or Container Manager.

**Cause:** We bypassed the Apps UI to use Docker Compose directly because TrueNAS Apps can't handle build contexts.

**Impact:**
- No graphical management
- No start/stop buttons in UI
- Must use command line for all operations

**Workaround:** Manage via SSH/Shell using `docker compose` commands.

### 2. Laptop/Other Devices Won't Sync

**Issue:** Desktop syncs fine, but laptop shows no sync activity.

**Possible causes:**
1. **HTTP vs HTTPS:** Some Firefox versions require HTTPS for sync
2. **Network:** Different network segments
3. **Firewall:** Port 8000 blocked

**Solutions:**
- Set up HTTPS with Nginx Proxy Manager (recommended)
- Ensure all devices can reach TrueNAS IP on port 8000
- Check TrueNAS firewall rules

### 3. Long Build Times

**Issue:** Initial build takes 10-60 minutes.

**Cause:** Compiling Rust code from source.

**Impact:** Updates also require full rebuilds.

**No workaround:** This is inherent to building from source.

### 4. Healthcheck Shows "Unhealthy"

**Issue:** Container status shows unhealthy even when working.

**Cause:** `curl` command not available in container for healthcheck.

**Impact:** Cosmetic only - server functions normally.

**Workaround:** Ignore this status or remove healthcheck from docker-compose.yaml.

---

## Maintenance

### Updating the Server
```bash
cd /mnt/your-pool/firefox-sync/syncstorage-rs-docker

# Pull latest code
git pull

# Rebuild (takes 10-60 minutes)
sudo docker compose down
sudo docker compose up -d --build
```

### Backing Up

**Important data locations:**
```bash
# Database files
/mnt/your-pool/firefox-sync/syncstorage-rs-docker/data/config/

# Configuration
/mnt/your-pool/firefox-sync/syncstorage-rs-docker/.env
/mnt/your-pool/firefox-sync/syncstorage-rs-docker/docker-compose.yaml
```

**Backup command:**
```bash
# Stop containers
sudo docker compose down

# Backup entire directory
tar -czf firefox-sync-backup-$(date +%Y%m%d).tar.gz /mnt/your-pool/firefox-sync/syncstorage-rs-docker/

# Start containers
cd /mnt/your-pool/firefox-sync/syncstorage-rs-docker
sudo docker compose up -d
```

### Monitoring

**Check logs regularly:**
```bash
# View recent activity
sudo docker compose logs --tail=100

# Watch for errors
sudo docker compose logs | grep -i error

# Monitor in real-time
sudo docker compose logs -f
```

**Health check:**
```bash
curl http://192.168.4.177:8000/__heartbeat__
```

Should return: `{"status":"Ok",...}`

---

## Setting Up HTTPS (Optional but Recommended)

Using Nginx Proxy Manager for SSL:

### Prerequisites
- Domain name pointing to your TrueNAS
- Nginx Proxy Manager installed on TrueNAS
- Port 80/443 forwarded to NPM

### Configuration

1. **Create Proxy Host in NPM:**
   - Domain: `sync.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.4.177`
   - Forward Port: `8000`
   - ✅ Block Common Exploits
   - ✅ Websockets Support

2. **SSL Tab:**
   - Request new Let's Encrypt certificate
   - ✅ Force SSL
   - ✅ HTTP/2 Support

3. **Update `.env` file:**
```bash
   SYNC_URL=https://sync.yourdomain.com
```

4. **Restart server:**
```bash
   sudo docker compose restart syncserver
```

5. **Update Firefox `about:config`:**
```
   identity.sync.tokenserver.uri = https://sync.yourdomain.com/1.0/sync/1.5
```

---

## Alternative Solutions

If this setup seems too complex, consider:

### 1. Mozilla's Official Sync (Recommended)
- **Pros:** Free, maintained, reliable, works everywhere
- **Cons:** Data on Mozilla's servers
- **Best for:** Most users who just want sync to work

### 2. Nextcloud Bookmarks
- **Pros:** TrueNAS app available, easy to manage, includes other features
- **Cons:** Only syncs bookmarks, not full Firefox data
- **Best for:** Those already using Nextcloud

### 3. xBrowserSync
- **Pros:** Simpler than Firefox Sync, cross-browser support
- **Cons:** Bookmarks only, requires browser extension
- **Best for:** Multi-browser users

### 4. Radicale
- **Pros:** Simple, lightweight, well-documented
- **Cons:** Calendar/contacts only, no bookmark sync
- **Best for:** CalDAV/CardDAV needs

---

## Technical Architecture
```
Firefox Client
    ↓ (OAuth)
Mozilla FXA Server (accounts.firefox.com)
    ↓ (OAuth Token)
Your Tokenserver (TrueNAS:8000/1.0/sync/1.5)
    ↓ (maps user to storage location)
MariaDB: tokenserver database
    ↓
Syncstorage API (TrueNAS:8000/1.5/{uid}/)
    ↓ (stores encrypted sync data)
MariaDB: syncstorage database
```

**Key Points:**
- Mozilla still handles authentication (you can't self-host this easily)
- Your server only stores encrypted sync data
- Tokenserver maps authenticated users to storage nodes

---

## File Structure
```
/mnt/your-pool/firefox-sync/syncstorage-rs-docker/
├── .env                          # Your secrets and configuration
├── docker-compose.yaml           # Container definitions
├── app/
│   ├── Dockerfile               # Builds syncstorage-rs from source
│   └── entrypoint.sh            # Initialization script
├── data/
│   ├── config/                  # MariaDB data (persistent)
│   └── initdb.d/               # Database initialization
└── README.md                    # Original repo documentation
```

---

## Commands Quick Reference
```bash
# Start containers
sudo docker compose up -d

# Stop containers
sudo docker compose down

# View logs
sudo docker compose logs

# Follow logs
sudo docker compose logs -f

# Restart specific container
sudo docker compose restart syncserver

# Rebuild and restart
sudo docker compose up -d --build

# Check container status
sudo docker compose ps

# Access MariaDB
sudo docker exec -it firefox_mariadb mariadb -u sync -pYourPassword

# Test server
curl http://192.168.4.177:8000/__heartbeat__

# View server version
curl http://192.168.4.177:8000/
```

---

## Credits and Resources

- **dan-r's syncstorage-rs-docker:** https://github.com/dan-r/syncstorage-rs-docker
- **Mozilla syncstorage-rs:** https://github.com/mozilla-services/syncstorage-rs
- **Firefox Sync Documentation:** https://mozilla-services.readthedocs.io/

---

## License

This guide is provided as-is for educational purposes. Firefox Sync and syncstorage-rs are Mozilla projects. Refer to their respective licenses.

---

## Final Thoughts

**Is this worth it?**

After going through this entire process:
- ✅ **Yes, if:** You enjoy learning, tinkering, and don't mind command-line management
- ❌ **No, if:** You want something that "just works" with minimal maintenance

**Reality check:**
- Setup time: 2-3 hours (including troubleshooting)
- Build time: 10-60 minutes per update
- Maintenance: Periodic monitoring required
- Complexity: High
- Reliability: Moderate (depends on your skill level)

For most users, **Mozilla's free sync service is the better choice**. This guide exists for those who want to learn or have specific privacy requirements.

---

**Questions or improvements?** Feel free to open an issue or submit a PR!
