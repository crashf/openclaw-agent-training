# Company Brain (Hive) — Agent Deployment Guide

Read this entire document before starting. Follow every step in order. Do not skip anything.

You are joining the Pund-IT Company Brain — a shared memory system that syncs knowledge between all agents via Syncthing. Once connected, your memory/ folder will stay in sync with every other agent in the hive.

---

## What You Need to Know

- **Hub** is the central sync relay at `170.205.18.11`
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
cat > ~/.config/syncthing/config.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<configuration version="40">
  <gui enabled="true" tls="false" bind-address="127.0.0.1" port="8384">
    <address>127.0.0.1:8384</address>
  </gui>
  <options>
    <deviceAddresses>
      <address>dynamic</address>
    </deviceAddresses>
    <relays>
      <address>relays://relays.syncthing.net:22067</address>
      <address>relay://47.244.91.78:22067</address>
    </relays>
    <maxConns>0</maxConns>
    <unackingNotificationLimit>0</unackingNotificationLimit>
  </options>
  <device id="YOUR_DEVICE_ID" name="YOUR_HOSTNAME" compression="metadata" relayServerEnabled="false" customCertDHE="" introducer="false" bypassGateway="true">
    <address>170.205.18.11</address>
  </device>
  <folder id="memory" label="memory" path="/root/.openclaw/workspace/memory/" type="sendreceive" rescanIntervalS="30" fsWatcherEnabled="true" fsWatcherDelayS="3" ignorePerms="false" autoNormalize="true">
    <device id="YOUR_DEVICE_ID" introducedBy="">
      <encryption password=""/>
    </device>
    <device id="REPLACED_BY_HUB" introducedBy="">
      <encryption password=""/>
    </device>
    <minDiskFree unit="1">1</minDiskFree>
    <maxConflicts>-1</maxConflicts>
    <fsync>true</fsync>
    <ignoreDelete>false</ignoreDelete>
    <ignoreUpdateSaveFile>false</ignoreUpdateSaveFile>
    <ignoreUpdateSaveDirFile>false</ignoreUpdateSaveDirFile>
    <ignoreUpdateSaveReaddir>false</ignoreUpdateSaveReaddir>
    <ignoreUpdateSaveVersioner=false</ignoreUpdateSaveVersioner>
    <copiers>0</copiers>
    <pullerMaxPendingKiB">0</pullerMaxPendingKiB>
    <hashers>0</hashers>
    <autoDefrag="true" />
    <sendOwnCert="true" />
    <syncOwnershipReadOnly="false" />
    <syncOwnership="true" />
    <syncGroupReadOnly="false" />
    <syncGroup="true" />
  </folder>
</configuration>
EOF
```

### If Running Inside Docker (some agents):

```bash
cat > ~/.config/syncthing/config.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<configuration version="40">
  <gui enabled="true" tls="false" bind-address="127.0.0.1" port="8384">
    <address>127.0.0.1:8384</address>
  </gui>
  <options>
    <deviceAddresses>
      <address>dynamic</address>
    </deviceAddresses>
    <relays>
      <address>relays://relays.syncthing.net:22067</address>
      <address>relay://47.244.91.78:22067</address>
    </relays>
    <maxConns>0</maxConns>
    <unackingNotificationLimit>0</unackingNotificationLimit>
  </options>
  <device id="YOUR_DEVICE_ID" name="YOUR_HOSTNAME" compression="metadata" relayServerEnabled="false" customCertDHE="" introducer="false" bypassGateway="true">
    <address>170.205.18.11</address>
  </device>
  <folder id="memory" label="memory" path="/var/syncthing/memory" type="sendreceive" rescanIntervalS="30" fsWatcherEnabled="true" fsWatcherDelayS="3" ignorePerms="false" autoNormalize="true">
    <device id="YOUR_DEVICE_ID" introducedBy="">
      <encryption password=""/>
    </device>
    <device id="REPLACED_BY_HUB" introducedBy="">
      <encryption password=""/>
    </device>
    <minDiskFree unit="1">1</minDiskFree>
    <maxConflicts>-1</maxConflicts>
    <fsync>true</fsync>
    <copiers>0</copiers>
    <pullerMaxPendingKiB">0</pullerMaxPendingKiB>
    <hashers>0</hashers>
    <autoDefrag="true" />
    <sendOwnCert="true" />
    <syncOwnershipReadOnly="false" />
    <syncOwnership="true" />
    <syncGroupReadOnly="false" />
    <syncGroup="true" />
  </folder>
</configuration>
EOF
```

> **Note:** The Hub device ID (`REPLACED_BY_HUB`) will be added by the Hub administrator. You will receive this after contacting them with your Device ID.

---

## Step 6: Start Syncthing

```bash
# Native:
systemctl enable --now syncthing@root

# Docker:
systemctl enable --now syncthing
```

---

## Step 7: Verify It's Working

Check the Syncthing Web UI:

```bash
curl -s http://127.0.0.1:8384 | head -5
```

Or check the journal:

```bash
journalctl -u syncthing@root -n 20 --no-pager
```

---

## Step 8: Contact the Hub Admin

Send the Hub admin your:
1. **Device ID** (from Step 3)
2. **Hostname** (e.g. `punaioff-ROSITA`)

They will add your device to the Hub and send you back the Hub's device ID to complete Step 5.

---

## Troubleshooting

### Syncthing won't start — " insufficient disk space"
See Step 1. The disk is likely at 99%+. Expand it.

### Config.xml keeps resetting
You edited it while Syncthing was running. Always stop first (Step 4).

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
