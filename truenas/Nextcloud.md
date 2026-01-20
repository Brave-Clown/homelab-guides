# Self-Hosting Nextcloud on TrueNAS SCALE

**Date Created:** January 2026  
**TrueNAS SCALE Version:** (Electric Eel / 24.10 branch or later – confirm in your Dashboard)  
**Goal:** Replace OneDrive/Google Drive with self-hosted cloud storage, including web UI, desktop sync (Windows + Bazzite Linux), and local file editing.

This guide documents a **basic working installation** using the official Nextcloud app from the TrueNAS catalog. We used host-path datasets for data persistence and resolved common deployment and "untrusted domain" issues.

## Prerequisites
- TrueNAS SCALE installed and updated (preferably 24.10+ for best app stability).
- A storage pool with space for datasets (e.g., your main tank/pool).
- Basic knowledge of TrueNAS UI (Datasets, Apps, Storage).
- Desktop clients ready: Nextcloud client for Windows and Flatpak on Bazzite.

**Note:** This is a **local-only** setup (LAN access via IP:port). Remote access via NPM (on Proxmox) and dynamic DNS is planned for later.

## Step 1: Create Required Datasets
The official Nextcloud app requires specific dataset structure with the **Apps** preset for proper permissions.

1. Go to **Storage > Datasets**.
2. Create a parent dataset:
   - Name: `nextcloud`
   - Preset: **Apps**
3. Create child datasets (all under the parent, preset **Apps**):
   - `appdata` (for Nextcloud config & installed apps)
   - `userdata` (for user files – this will hold your consolidated data)
   - `pgdata` (for PostgreSQL database – renamed from `postgres` during troubleshooting)
   - `pgbackup` (for PostgreSQL backups)
4. **Important:** Do **not** enable ACLs manually unless needed – we used automatic permissions during install.

**Note:** All datasets use the **Apps** preset to optimize case-sensitivity and permissions for containers (www-data UID/GID 568, postgres UID 999).

## Step 2: Install the Nextcloud App
1. Go to **Apps > Available Applications**.
2. Search for **Nextcloud** (official iXsystems chart).
3. Click **Install**.
4. **Application Name:** `nextcloud` (or custom).
5. **Credentials:** Set strong admin username & password (you'll use these to log in).
6. **Storage Configuration** (critical – map to your host paths):
   - Nextcloud AppData Storage → Host Path: `/mnt/yourpool/nextcloud/appdata`
   - Nextcloud User Data Storage → Host Path: `/mnt/yourpool/nextcloud/userdata`
   - Nextcloud Postgres Data Storage → Host Path: `/mnt/yourpool/nextcloud/pgdata`
   - Nextcloud Postgres Backup Storage → Host Path: `/mnt/yourpool/nextcloud/pgbackup`
7. **PostgreSQL Version:** Select the highest available (e.g., 18) to avoid upgrade script issues on fresh install.
8. **Network Configuration:** Note the assigned **Web Port** (in your case: **30027**).
9. Skip GPU passthrough.
10. **Additional Environment Variables:** (we tried but removed some due to conflicts – none needed after final fix).
11. **Permissions:** Check **Auto Configure Permissions** (or similar) – this resolved postgres_upgrade failures.
12. Click **Save** → wait for **Active** status (may take 5–10 minutes).

**Troubleshooting Deployment Failures**
- **postgres_upgrade exited with code 1**: Caused by mismatched datasets, permissions, or Redis volume trigger. Fixes we used:
  - Recreate datasets with Apps preset.
  - Delete/reinstall app + prune Docker volumes (`docker volume prune -f` in shell).
  - Change Postgres version to latest.
  - Enable auto permissions during install.
- Monitor logs: **Apps > Installed > Nextcloud > Logs**.

## Step 3: Initial Access & "Untrusted Domain" Fix
1. Access web UI: **http://192.168.XXX.XXX:30027** (your TrueNAS IP + port).
2. Log in with admin credentials.

**Initial Error:** "Access through untrusted domain" – common on fresh installs.

**Fix (via Environment Variables – what finally worked):**
1. Edit app (**Apps > Installed > Edit**).
2. Add **Additional Environment Variables**:
    - OVERWRITEHOST = 192.168.X.XXX:30027
    - OVERWRITECLIURL = http://192.168.X.XXX:30027
    - OVERWRITEPROTOCOL = http
    - (Do **not** add NEXTCLOUD_TRUSTED_DOMAINS – it conflicted with chart defaults.)
3. Save → redeploy.
4. Clear browser cache → reload → login succeeds.

**Alternative (if env vars fail):** Edit `config.php` directly via SMB share on `appdata` dataset:
- Add to `'trusted_domains'` array: `'192.168.XXX.XXX'`, `'192.168.XXX.XXX:30027'`.
- Add overwrite lines as above.

## Step 4: Desktop Client Setup (Windows + Bazzite)
1. Download client: https://nextcloud.com/install/#install-clients
- Windows: .exe installer.
- Bazzite (Linux): `flatpak install flathub org.nextcloud.Nextcloud` or flatpak from Bazzar.
2. Launch client → Server address: **http://192.168.XXX.XXX:30027**
3. Log in with Nextcloud user credentials.
4. Choose local sync folder (e.g., ~/Nextcloud on Bazzite → appears as /var/home/USERNAME/Nextcloud).
5. Sync completes → files appear locally.
6. **Local Editing:** Double-click files in sync folder → open in chosen office app → edit/save → auto-syncs back.

**Note:** On Bazzite, Dolphin sees the sync folder as normal local storage (no special Dolphin plugin needed for basic use). Right right and add to places for quick easy access.

## Step 5: Post-Setup Verification
- Upload test file via web UI → confirm in userdata dataset and desktop clients.
- Edit file locally on Bazzite/Windows → check syncs to web UI.
- Create regular user in Nextcloud UI (Users section) for daily use (avoid admin for clients).

## Future Steps (Not Yet Completed)
- Remote access from parents' house: NPM on Proxmox + nextcloud.example.domain.org + HTTPS.
- Online editing: ONLYOFFICE Document Server + connector app.
- Snapshots & backups: Periodic Snapshot Task on `nextcloud` parent dataset.
- HTTPS / external: Update config.php with domain, trusted_proxies (Proxmox IP).

## Troubleshooting Summary
- **Untrusted domain:** Use OVERWRITE* env vars or config.php edits.
- **Deployment stuck:** Prune volumes, check Postgres paths/permissions, use auto perms.
- **Sync issues:** Confirm firewall allows port 30027 locally; use exact URL in clients.

Enjoy your self-hosted cloud! Files are now synced locally across devices with full control.
