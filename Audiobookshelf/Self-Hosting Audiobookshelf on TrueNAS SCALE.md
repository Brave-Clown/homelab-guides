# Self-Hosting Audiobookshelf on TrueNAS SCALE

## Goal

Set up a self-hosted audiobook server using Audiobookshelf (ABS) on TrueNAS SCALE to:

- Store and organize my personal audiobook collection
- Access proper metadata (authors, series, covers)
- Stream audiobooks to my phone on the go
- Manage everything from my home network

## Prerequisites

- TrueNAS SCALE installed and accessible
- Basic familiarity with TrueNAS web interface
- A pool/dataset for storing media

## Step-by-Step Setup

### 1. Install Audiobookshelf

1. Navigate to **Apps** > **Available Applications**
2. Search for "Audiobookshelf" (official community app)
3. Install with default settings (config and metadata will use auto-created ixVolumes)

### 2. Create the Media Dataset

1. Go to **Datasets** > **Create Dataset**
2. Name: `audiobooks` (under your main pool, e.g., `pool1/audiobooks`)
3. Use default settings

### 3. Mount the Dataset to the App

This is critical - the container needs to see your audiobook files.

1. **Apps** > **Installed** > **Audiobookshelf** > **Edit**
2. Scroll to **Additional App Storage** section
3. Click **Add**:
   - **Type**: Host Path
   - **Host Path**: `/mnt/pool1/audiobooks` (use your exact pool name)
   - **Mount Path**: `/audiobooks`
4. **Save** (app will redeploy)

### 4. Set Up SMB Share for File Management

This allows easy file transfers from Windows/Mac.

1. **Shares** > **Add Windows (SMB) Share**
2. Select your `audiobooks` dataset as the path
3. Enable the share
4. Create an SMB user:
   - **Credentials** > **Local Users** > **Add**
   - Set username and password
   - **Enable "Samba Authentication"** (critical!)
   - Save

### 5. Configure Dataset Permissions

This was the trickiest part. The default permissions prevent both SMB write access and proper app access.

1. **Datasets** > `audiobooks` > **Permissions** > **Edit ACL**
2. Click **Select a preset ACL** > choose **POSIX_OPEN**
3. This adds three default entries:
   - **User Obj** – default – root
   - **Group Obj** – default – root
   - **Other** – default
4. **Verify all three entries have Read, Write, and Execute permissions checked**
5. Check **Apply recursively**
6. **Save**

**Why this works**: The POSIX_OPEN preset with full Read/Write/Execute on all three entries allows both authenticated SMB users and the Audiobookshelf container (running as the `apps` user) to access files without needing to add specific named users or groups.

### 6. Connect via SMB from Windows

1. Open File Explorer
2. In the address bar: `\\YOUR-NAS-IP\audiobooks` (use actual IP address)
3. Enter credentials when prompted:
   - Username: the user you created (or `TRUENAS-HOSTNAME\username`)
   - Password: the password you set
4. Check "Remember credentials"
5. If you have connection issues, clear old credentials: Open Command Prompt and run `net use * /delete`

### 7. Organize Your Audiobooks

Use this folder structure for best automatic metadata detection:

```
audiobooks/
└── Author Name/
    └── Series Name/
        └── Book Title/
            ├── audio_file_001.mp3
            ├── audio_file_002.mp3
            └── cover.jpg
```

**Example**:

```
audiobooks/
└── Sarah J. Maas/
    └── Throne of Glass/
        └── 01 - Throne of Glass/
            ├── ToG 000.mp3
            ├── ToG 001.mp3
            ├── ToG 002.mp3
            └── cover.jpg
```

**Tips**:

- Include sequence numbers in book folder names (01, 02) for series
- Add `cover.jpg` or `cover.png` in each book folder
- Supported formats: MP3, M4B, M4A, AAC, FLAC, OGG, OPUS

### 8. Add Library in Audiobookshelf

1. Open the ABS web interface
2. **Settings** > **Libraries** > **Add Library**
3. Click **Browse** and select `/audiobooks`
4. Set Type to **Audiobooks**
5. **Save** and run a full scan
6. Books should appear with metadata pulled from folder structure

### 9. Mobile Access

- **Android**: Official "Audiobookshelf" app (Google Play, free)
- **iOS**: ShelfPlayer (~$5) or SoundLeaf (paid) - official app not yet on App Store

For remote access outside your home network, configure port forwarding or use a VPN/Tailscale.

## Issues Encountered & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| `/audiobooks` folder not visible in ABS library browser | Host path mount didn't apply properly after editing | Stop app → Edit (no changes) → Save → Start. Updating the app also forced a clean redeploy |
| Could connect to SMB share but couldn't open/write to folders | Dataset permissions didn't allow write access | Applied POSIX_OPEN preset and verified all three entries (User Obj, Group Obj, Other) had Read + Write + Execute checked |
| "Named POSIX ACL entries require a mask entry" error when trying to add specific users | POSIX ACL strict requirements | Avoided the issue entirely by using POSIX_OPEN preset with full permissions on the three default entries instead of adding named users |
| Chapters show as filenames instead of proper chapter names | Source audiobook was split into multiple MP3 files | ABS treats each file as a chapter marker - works fine for playback. Can later use ABS Tools > "Encode as M4B" to merge into single file with proper chapters |
| App kept restarting in loop after storage changes | Edited existing config storage instead of adding additional storage | Never modify the existing config/metadata storage - always add audiobook storage as **Additional App Storage** |

## Final Thoughts

The biggest hurdle was understanding TrueNAS SCALE's permissions system. The POSIX_OPEN preset with Read/Write/Execute on all three default entries (User Obj, Group Obj, and Other) is simple and works reliably for home use.

**For future reference**: Creating the dataset with **SMB preset** and **NFSv4 ACL type** from the start would avoid most POSIX-related headaches, but the current setup works perfectly.

Once configured, adding new audiobooks is as simple as drag-and-drop via the SMB share. Progress syncs beautifully across devices.

Enjoy your self-hosted audiobook library! 🎧

---

## Additional Resources

- [Audiobookshelf Official Docs](https://www.audiobookshelf.org/guides)
- [Book Directory Structure Guide](https://www.audiobookshelf.org/guides)
- TrueNAS SCALE Forums for troubleshooting
