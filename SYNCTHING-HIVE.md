# Company Brain (Hive) — Agent Deployment Guide

Read this entire document before starting. Follow every step in order. Do not skip anything.

You are joining the Pund-IT Company Brain — a shared memory system that syncs knowledge between all agents via Syncthing. Once connected, your memory/ folder will stay in sync with every other agent in the hive.

---

## What You Need to Know

- **Hub** is the central sync relay at `170.205.18.11` (device ID: `HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ`, name: `hino-wiki-sync`)
- **Folder ID** is `memory` — all agents use this exact ID
- **Your sync path** is `/root/.openclaw/workspace/memory/` (native) or `/var/syncthing/memory` (Docker)
- **Shared knowledge** lives in `memory/synthesis/`, `memory/index.md`, `memory/log.md`, and daily notes
- **Your identity files** (`SOUL.md`, `IDENTITY.md`, `USER.md`, etc.) are **NOT** shared — they live outside `memory/`

---

## Step 1: Check Disk Space

Syncthing refuses to start if disk is less than 1% free. Check first:

```bash
df -h /
```

If Use% is 99%+, you **MUST** expand the disk before continuing. On VMware/LVM systems:

```bash
# Rescan the disk (after expanding in VMware/vSphere)
echo 1 > /sys/class/block/sda/device/rescan

# Grow the partition (adjust partition number if needed)
apt-get install -y cloud-utils-growpart
growpart /dev/sda 3

# Resize LVM and filesystem live (no reboot needed)
pvresize /dev/sda3
lvextend -rL +30G /dev/mapper/ubuntu--vg-ubuntu--lv

# Verify
df -h /
```

---

## Step 2: Install Syncthing

```bash
# Add Syncthing apt repository
curl -s https://syncthing.net/release-key.txt | gpg --dearmor | \
  tee /usr/share/keyrings/syncthing-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] \
  https://apt.syncthing.net/ syncthing stable" | \
  tee /etc/apt/sources.list.d/syncthing.list

apt-get update -qq && apt-get install -y syncthing
```

---

## Step 3: Generate Config & Get Your Device ID

```bash
mkdir -p ~/.config/syncthing
mkdir -p ~/.local/share/syncthing
mkdir -p /root/.openclaw/workspace/memory

syncthing -generate ~/.config/syncthing
```

Now get your **Device ID** — you'll need it for Step 5:

```bash
grep -oP 'device id="\K[^"]+' ~/.config/syncthing/config.xml | head -1
```

Write this ID down. You will paste it into the config in Step 5 and send it to whoever manages the Hub.

---

## Step 4: Stop Syncthing If Running

> ⚠️ **CRITICAL RULE:** Never edit `config.xml` while Syncthing is running. Syncthing overwrites the file on shutdown, destroying any changes you made.

```bash
pkill syncthing 2>/dev/null || true
systemctl stop syncthing@root 2>/dev/null || true
```

---

## Step 5: Write Your Config

Replace `YOUR_DEVICE_ID` with the ID from Step 3. Replace `YOUR_HOSTNAME` with your actual hostname.

### If Running Syncthing Natively (most agents):

```bash
cat > ~/.config/syncthing/config.xml << 'EOFXML'
<configuration version="37">
  <folder id="memory" label="Company Brain" path="/root/.openclaw/workspace/memory" type="sendreceive" rescanIntervalS="3600" fsWatcherEnabled="true" fsWatcherDelayS="10" fsWatcherTimeoutS="0" ignorePerms="false" autoNormalize="true">
    <filesystemType>basic</filesystemType>
    <device id="YOUR_DEVICE_ID" introducedBy="">
      <encryptionPassword></encryptionPassword>
    </device>
    <device id="HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ" introducedBy="">
      <encryptionPassword></encryptionPassword>
    </device>
    <minDiskFree unit="%">1</minDiskFree>
    <versioning>
      <cleanupIntervalS>3600</cleanupIntervalS>
      <fsPath></fsPath>
      <fsType>basic</fsType>
    </versioning>
    <copiers>0</copiers>
    <pullerMaxPendingKiB>0</pullerMaxPendingKiB>
    <hashers>0</hashers>
    <order>random</order>
    <ignoreDelete>false</ignoreDelete>
    <scanProgressIntervalS>0</scanProgressIntervalS>
    <pullerPauseS>0</pullerPauseS>
    <maxConflicts>10</maxConflicts>
    <disableSparseFiles>false</disableSparseFiles>
    <paused>false</paused>
    <markerName>.stfolder</markerName>
  </folder>
  <device id="YOUR_DEVICE_ID" name="YOUR_HOSTNAME" compression="metadata" introducer="false" skipIntroductionRemovals="false" introducedBy="">
    <address>dynamic</address>
    <paused>false</paused>
    <autoAcceptFolders>false</autoAcceptFolders>
  </device>
  <device id="HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ" name="hino-wiki-sync" compression="metadata" introducer="false" skipIntroductionRemovals="false" introducedBy="">
    <address>dynamic</address>
    <paused>false</paused>
    <autoAcceptFolders>false</autoAcceptFolders>
  </device>
  <gui enabled="true" tls="false" debugging="false" sendBasicAuthPrompt="false">
    <address>127.0.0.1:8384</address>
    <theme>default</theme>
  </gui>
  <options>
    <listenAddress>default</listenAddress>
    <globalAnnounceEnabled>true</globalAnnounceEnabled>
    <localAnnounceEnabled>true</localAnnounceEnabled>
    <relaysEnabled>true</relaysEnabled>
  </options>
</configuration>
EOFXML
```

### If Running Syncthing in Docker:

> ⚠️ **CRITICAL:** The folder path MUST be `/var/syncthing/memory` — NOT `/root/.openclaw/workspace/memory`. Inside the container, the host directory is mounted at `/var/syncthing/memory`. If you use the host path, sync will fail with "folder path missing" and get stuck at ~47%. This is the #1 Docker gotcha.

```bash
docker stop openclaw-wiki-sync 2>/dev/null || true

cat > /opt/syncthing-agent/config/config.xml << 'EOFXML'
<configuration version="37">
  <folder id="memory" label="Company Brain" path="/var/syncthing/memory" type="sendreceive" rescanIntervalS="3600" fsWatcherEnabled="true" fsWatcherDelayS="10" fsWatcherTimeoutS="0" ignorePerms="false" autoNormalize="true">
    <filesystemType>basic</filesystemType>
    <device id="YOUR_DEVICE_ID" introducedBy="">
      <encryptionPassword></encryptionPassword>
    </device>
    <device id="HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ" introducedBy="">
      <encryptionPassword></encryptionPassword>
    </device>
    <minDiskFree unit="%">1</minDiskFree>
    <versioning>
      <cleanupIntervalS>3600</cleanupIntervalS>
      <fsPath></fsPath>
      <fsType>basic</fsType>
    </versioning>
    <copiers>0</copiers>
    <pullerMaxPendingKiB>0</pullerMaxPendingKiB>
    <hashers>0</hashers>
    <order>random</order>
    <ignoreDelete>false</ignoreDelete>
    <scanProgressIntervalS>0</scanProgressIntervalS>
    <pullerPauseS>0</pullerPauseS>
    <maxConflicts>10</maxConflicts>
    <disableSparseFiles>false</disableSparseFiles>
    <paused>false</paused>
    <markerName>.stfolder</markerName>
  </folder>
  <device id="YOUR_DEVICE_ID" name="YOUR_HOSTNAME" compression="metadata" introducer="false" skipIntroductionRemovals="false" introducedBy="">
    <address>dynamic</address>
    <paused>false</paused>
    <autoAcceptFolders>false</autoAcceptFolders>
  </device>
  <device id="HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ" name="hino-wiki-sync" compression="metadata" introducer="false" skipIntroductionRemovals="false" introducedBy="">
    <address>dynamic</address>
    <paused>false</paused>
    <autoAcceptFolders>false</autoAcceptFolders>
  </device>
  <gui enabled="true" tls="false" debugging="false" sendBasicAuthPrompt="false">
    <address>0.0.0.0:8384</address>
    <theme>default</theme>
  </gui>
  <options>
    <listenAddress>default</listenAddress>
    <globalAnnounceEnabled>true</globalAnnounceEnabled>
    <localAnnounceEnabled>true</localAnnounceEnabled>
    <relaysEnabled>true</relaysEnabled>
  </options>
</configuration>
EOFXML

docker start openclaw-wiki-sync
```

> **Note:** Docker GUI binds to `0.0.0.0:8384` (accessible externally) while native binds to `127.0.0.1:8384` (localhost only).

---

## Step 6: Create .stignore

This prevents agent-specific and junk files from syncing to other agents:

```bash
mkdir -p /root/.openclaw/workspace/memory

cat > /root/.openclaw/workspace/memory/.stignore << 'EOF'
// Agent-specific recall files — each agent has its own
.dreams

// Sync conflict files — don't propagate conflicts
*.sync-conflict-*
EOF
```

---

## Step 7: Start Syncthing & Enable Boot Persistence

### Native:

```bash
systemctl enable syncthing@root
systemctl start syncthing@root
systemctl status syncthing@root
```

### Docker:

Already running from Step 5 (`docker start`). Verify:

```bash
docker ps | grep openclaw-wiki-sync
```

---

## Step 8: Register on the Hub

Your agent won't sync until the Hub knows about you. Send your Device ID (from Step 3) to whoever manages the Hub. They need to do one of these:

### Option A: Hub Web UI (easiest)

1. Open http://170.205.18.11:8384
2. Click **Add Remote Device**
3. Paste your Device ID
4. Under **Shared Folders**, check the `memory` folder
5. Save

### Option B: Hub API (automated)

```bash
# TBD — Hub admin may expose an API endpoint for auto-registration
```

---

## Troubleshooting

### Syncthing won't start — "insufficient disk space"
See Step 1. The disk is likely at 99%+. Expand it.

### Config.xml keeps resetting
You edited it while Syncthing was running. Always stop first (Step 4).

### Folder stuck at ~47% on Docker
You're using the host path (`/root/.openclaw/workspace/memory`) instead of the container path (`/var/syncthing/memory`). See the Docker note in Step 5.

### Nothing syncing
1. Check that your Device ID was added to the Hub
2. Verify the folder ID is exactly `memory` (case-sensitive)
3. Check `journalctl -u syncthing@root -n 50` for errors
4. Confirm your path matches: `/root/.openclaw/workspace/memory/` (native) or `/var/syncthing/memory` (Docker)

### Permission denied errors
Make sure the `memory/` directory exists and Syncthing has read/write access:
```bash
mkdir -p /root/.openclaw/workspace/memory
chmod 755 /root/.openclaw/workspace/memory
```
