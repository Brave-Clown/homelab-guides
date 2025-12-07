# Upgrading Proxmox VE from 8.2.2 to 9.1.2

**Date:** December 6th and 7th (Six Seven), 2025  
**Duration:** ~1 hour (not including backup time and drama)  
**Outcome:** ✅ Success - all VMs and containers running smoothly

## Pre-Upgrade Drama: The Backup Nightmare

**Problem:** After manually shrinking the disk of Windows VM 100 inside Windows (Disk Management → shrink volume), Proxmox's thin-LVM pool (local-lvm) got out of sync:
- GUI and config still showed 427 GiB
- Actual logical volume size dropped to ~125 GiB
- Thin-pool metadata still thought ~196 GiB were mapped
- VM at one point needed a lot more drive space, but that changed many many months ago, and I failed to cleanup properly at the time, leading to some issues with backing up my LXC 106.

**Result:**
- Every backup job (even unrelated LXCs like OpenWebUI ID 106) spammed: `WARNING: Thin volume pve/vm-100-disk-0 maps 211186941952 while the size is only 134217728000`
- Backups either hung forever or crawled at glacial speed because vzdump was trying to read phantom space

**The Fix (the exact 3 commands that solved it):**
```bash
# 1. See the real (broken) size
lvs -o+lv_size,data_percent,metadata_percent /dev/pve/vm-100-disk-0

# 2. Shrink the logical volume to the size you actually want
#    (VM must be stopped)
lvresize -L 125G /dev/pve/vm-100-disk-0
# → answer "y" to the warning prompt

# 3. Make Proxmox GUI catch up with reality
qm rescan --vmid 100
```

**After step 3:**
- GUI instantly flipped from 427 GiB → 125 GiB
- The thin-metadata warning disappeared forever
- All backups (VMs and LXCs) finished in minutes instead of hanging or taking hours (maybe they would have finished, but I stopped the backup jobs after ~5minutes)

## Pre-Upgrade Preparation

1. **Backed up all VMs and LXC containers** (critical - don't skip this! Even with backup drama, having backups is essential)

2. **Updated to latest Proxmox 8.4.x** to get the upgrade checker tool:
```bash
   apt update
   apt dist-upgrade
   reboot
```

3. **Fixed repository configuration** - switched from enterprise to no-subscription repos:
```bash
   # Commented out enterprise repos in:
   /etc/apt/sources.list.d/pve-enterprise.list
   /etc/apt/sources.list.d/ceph.list
   
   # Added no-subscription repos:
   # /etc/apt/sources.list.d/pve-no-subscription.list
   deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

## Running the Pre-Upgrade Checker
```bash
pve8to9 --full
```

The checker identified several issues that needed fixing:

**Critical (blocking upgrade):**
- `systemd-boot` meta-package conflict
- GRUB bootloader configuration issue

**Warnings (recommended fixes):**
- Missing AMD microcode package
- VM 100 had old QEMU machine version
- 1 running guest needed to be stopped
- Thin volume warnings still present (informational only after the fix above)

## Fixing the Issues

### 1. Removed systemd-boot conflict
```bash
apt remove systemd-boot
```

### 2. Fixed GRUB bootloader
```bash
echo 'grub-efi-amd64 grub2/force_efi_extra_removable boolean true' | debconf-set-selections -v -u
apt install --reinstall grub-efi-amd64
```

### 3. Installed AMD microcode
Added `non-free-firmware` to apt sources:
```bash
# Edited /etc/apt/sources.list to add non-free-firmware:
deb http://ftp.us.debian.org/debian bookworm main contrib non-free-firmware
deb http://ftp.us.debian.org/debian bookworm-updates main contrib non-free-firmware
deb http://security.debian.org bookworm-security main contrib non-free-firmware
```

Then installed the package:
```bash
apt update
apt install amd64-microcode
```

### 4. Stopped all running VMs/containers
```bash
qm stop 100
```

### 5. Cleaned up duplicate repository entries
Removed duplicate entries in `/etc/apt/sources.list` to eliminate warnings.

## The Actual Upgrade

### 1. Updated repository sources from Debian Bookworm to Trixie

Edited `/etc/apt/sources.list` and changed all "bookworm" to "trixie":
```bash
deb http://ftp.us.debian.org/debian trixie main contrib non-free-firmware
deb http://ftp.us.debian.org/debian trixie-updates main contrib non-free-firmware
deb http://security.debian.org trixie-security main contrib non-free-firmware
```

### 2. Updated Proxmox repositories

Updated `/etc/apt/sources.list.d/pve-no-subscription.list`:
```bash
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
```

Updated `/etc/apt/sources.list.d/ceph.list` (removed old ceph-quincy):
```bash
deb http://download.proxmox.com/debian/ceph-squid trixie no-subscription
```

### 3. Performed the upgrade
```bash
apt update
apt dist-upgrade
```

### 4. Configuration file prompts during upgrade

During the upgrade process, several configuration file conflicts appeared:

- `/etc/issue` → Accepted maintainer's version (Y)
- Restart services automatically → Yes
- `/etc/systemd/journald.conf` → Accepted maintainer's version (Y)
- `/etc/lvm/lvm.conf` → **Accepted maintainer's version (Y)** - important for PVE 9

### 5. Rebooted
```bash
reboot
```

## Post-Upgrade Verification

### Check version
```bash
pveversion
```
Output: `pve-manager/9.1.2/9d436f37a0ac4172 (running kernel: 6.17.2-2-pve)`

### Verify services
```bash
systemctl status pveproxy pvedaemon pvescheduler pvestatd
```
All services: active (running) ✅

### Check storage
```bash
pvesm status
```
All storage active ✅

### Start VMs
```bash
qm start 100
```
VM with old machine version started without issues ✅

## Gotchas and Notes

- **Thin volume warnings:** The persistent `WARNING: Thin volume pve/vm-100-disk-0 maps...` messages continued after upgrade but are informational only (related to the disk shrinking adventure from before the upgrade)
- **Never shrink disks in Windows Disk Management alone:** Always coordinate disk changes with Proxmox's LVM layer using `lvresize` and `qm rescan` to keep everything in sync
- **Old machine version:** VM 100 with pre-v6.0 QEMU machine type still works fine in PVE 9.1 - no immediate action needed
- **Ceph repository confusion:** Had stray `ceph-quincy` entry in `pve-no-subscription.list` that needed removal
- **Auto-restart services:** Saying "Yes" to automatically restarting services during upgrade made the process much smoother

## Final Result

- **Before:** Proxmox VE 8.2.2 (Debian Bookworm)
- **After:** Proxmox VE 9.1.2 (Debian Trixie) with kernel 6.17.2-2-pve
- **Downtime:** ~15 minutes (reboot + service startup)
- **Issues:** None - all VMs and LXC containers started successfully

## Lessons Learned

1. **NEVER shrink VM disks inside the guest OS without coordinating with Proxmox** - use `lvresize` and `qm rescan` to keep thin-LVM metadata in sync
2. The `pve8to9 --full` checker is your friend - run it and fix everything it complains about
3. systemd-boot and GRUB conflicts are common blockers
4. Accept the new `/etc/lvm/lvm.conf` during upgrade (important for PVE 9 LVM changes)
5. Old QEMU machine versions aren't immediate dealbreakers - VMs will likely still work
6. Always have backups before major upgrades (did I mention backups?)
7. When in doubt, the Proxmox community and AI assistants can help troubleshoot weird issues

---

**Note:** This guide is based on a real upgrade experience on December 7, 2025. Your mileage may vary depending on your specific configuration, but these steps should provide a solid foundation for anyone upgrading from Proxmox VE 8.x to 9.x.
