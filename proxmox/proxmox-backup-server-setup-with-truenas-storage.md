# Proxmox Backup Server 4.1 Setup with TrueNAS SCALE

## Overview

This guide documents the complete setup of Proxmox Backup Server (PBS) 4.1 as a VM on Proxmox VE 9.1, with storage provided by TrueNAS SCALE Community via NFS. This project involved troubleshooting WD PR4100 NAS compatibility issues before ultimately switching to TrueNAS for proper ZFS support.

**Note**: The TrueNAS SCALE instance used in this guide is a test environment. Full migration from the WD PR4100 to a dedicated TrueNAS box is a future project.

---

## Initial PBS VM Configuration

**VM Specifications:**
- Machine: q35
- BIOS: SeaBIOS
- Disk: 32GB
- Memory: 6144 MB (6GB)
- OS: PBS 4.1 (Debian Trixie-based)

---

## Phase 1: Initial Setup & Repository Configuration

### 1.1 Fix Repository Access

PBS initially tries to access the enterprise repository without a subscription.

```bash
# Update package lists (will show enterprise repo error)
apt update

# Disable enterprise repository
echo "# deb https://enterprise.proxmox.com/debian/pbs trixie pbs-enterprise" > /etc/apt/sources.list.d/pbs-enterprise.list

# Add community (no-subscription) repository
echo "deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription" > /etc/apt/sources.list.d/pbs-no-subscription.list

# Remove enterprise repository files completely
rm /etc/apt/sources.list.d/pbs-enterprise.list
rm /etc/apt/sources.list.d/pbs-enterprise.sources

# Update and upgrade
apt update
apt upgrade -y
```

---

## Phase 2: Attempting WD PR4100 NAS Integration (Failed)

### 2.1 Initial CIFS/SMB Attempt

**Goal**: Mount the WD PR4100's SMB share for PBS storage.

```bash
# Install CIFS utilities
apt install cifs-utils -y

# Create mount point
mkdir -p /mnt/nas-backup

# Create credentials file
nano /root/.nascredentials
```

Add credentials:
```
username=your_email@gmail.com
password=your_password
```

Secure the file:
```bash
chmod 600 /root/.nascredentials
```

Add to `/etc/fstab`:
```
//192.168.4.252/proxmox/PBS /mnt/nas-backup cifs credentials=/root/.nascredentials,vers=3.0,_netdev 0 0
```

Mount:
```bash
systemctl daemon-reload
mount -a
```

**Result**: Mount succeeded, but datastore creation failed with `EMLINK: Too many links` error after creating ~70,000 subdirectories. SMB/CIFS cannot handle PBS's chunk store structure.

### 2.2 Switching to NFS on PR4100

Discovered the share was available via NFS: `nfs://192.168.4.252/nfs/Proxmox`

```bash
# Install NFS client
apt install nfs-common -y

# Update fstab for NFS
nano /etc/fstab
```

Changed to:
```
192.168.4.252:/nfs/Proxmox/PBS /mnt/nas-backup nfs vers=3,rw,sync,hard,intr,_netdev 0 0
```

```bash
systemctl daemon-reload
mount -a
```

**Result**: Mount succeeded, but still got `EMLINK: Too many links` error at ~70,000 subdirectories. The WD PR4100's underlying filesystem has a subdirectory limit that PBS's chunk store structure exceeds.

### 2.3 Attempting to Fix PR4100 Root Squash

**Problem**: Even after mounting, PBS datastore creation failed with `EPERM: Operation not permitted` due to root squashing.

**Critical Discovery**: WD PR4100 blocks root squash configuration via the web GUI.

#### SSH Access to PR4100
- **Username**: `sshd` (not root or admin!)
- **Password**: Set when enabling SSH in web GUI

```bash
# SSH into PR4100
ssh sshd@192.168.4.252

# Edit NFS exports
nano /etc/exports
```

Modified line to disable root squashing:
```
/nfs/Proxmox/PBS 192.168.4.0/24(rw,sync,no_subtree_check,no_root_squash,anonuid=34,anongid=34)
```

```bash
# Reload exports
exportfs -ra
```

**Result**: Root squash was fixed, but the `EMLINK: Too many links` error persisted due to filesystem limitations. **Decision made to switch to TrueNAS with ZFS.**

---

## Phase 3: TrueNAS SCALE Setup (Current Working Solution)

### 3.1 TrueNAS Configuration

**TrueNAS Setup:**
1. Created dataset: `/mnt/pool1/Proxmox`
2. Set dataset permissions:
   - Owner (user): `backup`
   - Owner (group): `backup`

3. Created NFS share:
   - Path: `/mnt/pool1/Proxmox`
   - Authorized Networks: `192.168.4.0/24`
   - Maproot User: (empty)
   - Maproot Group: (empty)
   - **Mapall User**: `backup`
   - **Mapall Group**: `backup`
   - Enabled "Allow non-root mount"

4. Enabled NFS service:
   - Services → NFS → Enable + Start automatically
   - In service settings, checked "Allow non-root mount"

**Important**: When changing ownership from root to backup, there's a confirmation checkbox that must be checked for changes to save.

### 3.2 PBS VM Configuration for TrueNAS

```bash
# Test mount (verify it works)
mount -t nfs 192.168.4.177:/mnt/pool1/Proxmox /mnt/truenas-pbs

# Verify ownership (should show backup:backup)
ls -ld /mnt/truenas-pbs
# Output: drwxr-xr-x 2 backup backup 2 Dec  7 19:20 /mnt/truenas-pbs

# Make permanent in fstab
nano /etc/fstab
```

Add line:
```
192.168.4.177:/mnt/pool1/Proxmox /mnt/truenas-pbs nfs defaults,_netdev 0 0
```

### 3.3 Create PBS Datastore

In PBS Web GUI:
1. Navigate to **Datastore** → **Add Datastore**
2. **Name**: `PBS` (case-sensitive!)
3. **Backing Path**: `/mnt/truenas-pbs`
4. Click **Add**

**Result**: Datastore created successfully! ZFS has no subdirectory limitations.

---

## Phase 4: Integration with Proxmox VE

### 4.1 Add PBS as Storage in Proxmox VE

1. In Proxmox VE GUI: **Datacenter** → **Storage** → **Add** → **Proxmox Backup Server**

2. Configuration:
   - **ID**: `pbs-truenas`
   - **Server**: `192.168.4.xxx` (PBS VM IP)
   - **Username**: `root@pam`
   - **Password**: (PBS root password)
   - **Datastore**: `PBS` (must match exact case!)
   - **Fingerprint**: Get from PBS VM

3. Get SSL fingerprint from PBS:
```bash
proxmox-backup-manager cert info | grep Fingerprint
```

Copy the fingerprint and paste it into the Proxmox VE storage configuration.

### 4.2 Test Backup

1. Select a VM or LXC container
2. **Backup** → **Backup now**
3. Select PBS storage
4. Run backup

**Result**: Backup successful!

---

## Phase 5: PR4100 Cleanup & Restoration

After switching to TrueNAS, the WD PR4100 was restored to its original configuration.

### 5.1 Restore PR4100 NFS Exports

```bash
# SSH into PR4100
ssh sshd@192.168.4.252

# Edit exports back to original
nano /etc/exports
```

Restored original line:
```
"/nfs/Proxmox" *(rw,all_squash,sync,no_wdelay,insecure_locks,insecure,no_subtree_check,anonuid=501,anongid=1000)
```

```bash
# Reload exports
exportfs -ra

# Exit SSH
exit
```

### 5.2 Disable SSH on PR4100

In PR4100 Web GUI:
- **Settings** → **Network** → **SSH** → Toggle **OFF**

### 5.3 Clean Up PBS VM

```bash
# Edit fstab
nano /etc/fstab
```

Comment out or remove old PR4100 mount lines:
```
# //192.168.4.252/proxmox/PBS /mnt/nas-backup cifs credentials=/root/.nascredentials,vers=3.0,nounix,noserverino,_netdev 0 0
# 192.168.4.252:/nfs/Proxmox/PBS /mnt/nas-backup nfs vers=3,rw,sync,hard,intr,_netdev 0 0
```

```bash
# Unmount if still mounted
umount /mnt/nas-backup

# Remove old mount point directory
rmdir /mnt/nas-backup

# Remove credentials file
rm /root/.nascredentials

# Verify cleanup
ls /mnt
cat /root/.nascredentials  # Should show "No such file"
```

---

## Verification & Performance

### Deduplication Verification

PBS uses chunk-based deduplication. To verify it's working:

```bash
# Check datastore status
proxmox-backup-manager datastore status

# Check actual disk usage
du -sh /mnt/truenas-pbs/.chunks/
du -sh /mnt/truenas-pbs/
```

**Test Results** (backing up same LXC container twice):
- PBS GUI shows: 1.79 GiB × 2 = ~3.58 GiB logical size
- Actual disk usage: 929 MB
- **Deduplication ratio**: ~4:1

This is expected behavior - the GUI shows logical backup size (what you'd restore), but PBS only stores unique chunks once.

---

## Key Lessons Learned

### Why WD PR4100 Failed
1. **CIFS/SMB**: Cannot handle PBS's 70,000+ subdirectory chunk store (EMLINK error)
2. **NFS**: Also hit subdirectory limits at ~70,000 directories (EMLINK error)
3. **Root squashing**: Blocked in web GUI, requires SSH access
4. **SSH username**: `sshd`, not `root` or `admin`
5. **Conclusion**: WD PR4100's filesystem has inherent limitations incompatible with PBS's storage structure

### Why TrueNAS Succeeded
1. **ZFS filesystem**: No practical subdirectory limits
2. **Proper NFS implementation**: Full control over maproot/mapall
3. **Professional-grade**: Designed for enterprise storage workloads
4. **PBS compatibility**: Built to handle chunk-based storage

### Filesystem Subdirectory Limits (General Reference)
Different filesystems have different practical limits for subdirectories:

- **ext3**: ~32,000 subdirectories per directory
- **ext4**: ~64,000 subdirectories per directory
- **XFS**: No practical limit (billions)
- **ZFS**: No practical limit (billions)
- **btrfs**: No practical limit (billions)

**Note**: The WD PR4100 hit limitations around 70,000 subdirectories, though we didn't verify the exact filesystem in use. The key takeaway is that consumer NAS devices may have limits that prevent PBS chunk store creation.

**Recommendation**: For PBS, use ZFS (TrueNAS), XFS, or btrfs on systems where you have full control.

---

## Future Plans

1. **Current State**: Using TrueNAS SCALE Community as test environment on a small dell mini pc
2. **Production Plan**: Migrate from WD PR4100 to dedicated TrueNAS hardware
3. **Backup Strategy**: Continue using PBS with TrueNAS ZFS storage

---

## Final Configuration Summary

### PBS VM (`/etc/fstab`)
```
# <file system> <mount point> <type> <options> <dump> <pass>
/dev/pbs/root / ext4 errors=remount-ro 0 1
/dev/pbs/swap none swap sw 0 0
proc /proc proc defaults 0 0
192.168.4.177:/mnt/pool1/Proxmox /mnt/truenas-pbs nfs defaults,_netdev 0 0
```

### TrueNAS NFS Export
- **Path**: `/mnt/pool1/Proxmox`
- **Allowed Networks**: `192.168.4.0/24`
- **Maproot User/Group**: (empty)
- **Mapall User/Group**: `backup` (UID/GID 34)
- **Options**: Allow non-root mount enabled

### Proxmox VE Storage
- **Type**: Proxmox Backup Server
- **Server**: PBS VM IP
- **Datastore**: `PBS`
- **Authentication**: root@pam

---

## Useful Commands

### PBS VM
```bash
# Check datastore status
proxmox-backup-manager datastore status

# Get SSL fingerprint
proxmox-backup-manager cert info | grep Fingerprint

# Check disk usage
df -h /mnt/truenas-pbs
du -sh /mnt/truenas-pbs/.chunks/

# Verify mount
mount | grep truenas
```

### TrueNAS
```bash
# Verify backup user
id backup
# Should show: uid=34(backup) gid=34(backup)

# Check NFS exports
cat /etc/exports
showmount -e localhost
```

---

## Troubleshooting Tips

### PBS Datastore Creation Fails
1. Check ownership: `ls -ld /mnt/truenas-pbs` (should show `backup:backup`)
2. Verify NFS mount: `mount | grep truenas`
3. Test file creation: `touch /mnt/truenas-pbs/test.txt`
3. Check TrueNAS mapall settings (must be `backup` for both user and group, maproot should be empty)

### Proxmox VE Can't Add PBS Storage
1. Verify PBS is accessible: `ping PBS_IP`
2. Check SSL fingerprint matches
3. Datastore name is case-sensitive (use exact name from PBS)
4. Ensure PBS web interface is running: `systemctl status proxmox-backup`

### Mount Fails After Reboot
1. Check fstab syntax
2. Verify `_netdev` option is present (delays mount until network is ready)
3. Check TrueNAS NFS service is running
4. Test manual mount: `mount -a`

---

## References

- [Proxmox Backup Server Documentation](https://pbs.proxmox.com/docs/)
- [TrueNAS SCALE Documentation](https://www.truenas.com/docs/scale/)
- [NFS Best Practices for PBS](https://forum.proxmox.com/threads/proxmox-backup-server-nfs-storage.90819/)

---

**Last Updated**: December 7, 2024  
**PBS Version**: 4.1  
**TrueNAS Version**: SCALE Community 25.10.0 (Goldeye)  
**Status**: Working - Production-ready pending TrueNAS hardware migration
