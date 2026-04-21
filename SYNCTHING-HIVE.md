# Company Brain (Hive) — Agent Deployment Guide

Read this entire document before starting. Follow every step in order. Do not skip anything.

You are joining the Pund-IT Company Brain — a shared memory system that syncs knowledge between all agents via Syncthing. Once connected, your memory/ folder will stay in sync with every other agent in the hive.

---

## What You Need to Know

- **Hub** is the central sync relay at `170.205.18.11` (device ID: `HLBW742-2EYCQ7X-NC5MNKM-R4NBAMH-2XMBND4-G7MX7KX-QKBUMM7-DRDINAQ`, name: `hino-wiki-sync`)
- **Hub API Key:** `hLjecHTSiKDcw9qpCiT6VNL6vZsF7oC6`
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

Now get your **Device ID** — you'll need it for Steps 5 and 8:

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

Your agent won't sync until the Hub knows about you. Send your Device ID (from Step 3) to whoever manages the Hub.

### Option A: Hub Web UI (easiest)

1. Open http://170.205.18.11:8384
2. Click **Add Remote Device**
3. Paste your Device ID
4. Under **Shared Folders**, check the `memory` folder
5. Save

### Option B: Hub API (automated)

Run this on the Hub host (`170.205.18.11`). Replace `NEW_DEVICE_ID` with your agent's Device ID and `NEW_HOSTNAME` with your hostname.

```bash
# Add the device
curl -s -X POST http://localhost:8384/rest/config/devices \
  -H 'X-API-Key: hLjecHTSiKDcw9qpCiT6VNL6vZsF7oC6' \
  -H 'Content-Type: application/json' \
  -d '{
    "deviceID": "NEW_DEVICE_ID",
    "name": "NEW_HOSTNAME",
    "addresses": ["dynamic"],
    "compression": "metadata",
    "introducer": false,
    "autoAcceptFolders": false
  }'

# Add the device to the memory folder
curl -s http://localhost:8384/rest/config/folders/memory \
  -H 'X-API-Key: hLjecHTSiKDcw9qpCiT6VNL6vZsF7oC6' > /tmp/folder.json

python3 -c "
import json
with open('/tmp/folder.json') as f:
    data = json.load(f)
data['devices'].append({'deviceID': 'NEW_DEVICE_ID', 'introducedBy': ''})
with open('/tmp/folder.json', 'w') as f:
    json.dump(data, f)
"

curl -s -X PUT http://localhost:8384/rest/config/folders/memory \
  -H 'X-API-Key: hLjecHTSiKDcw9qpCiT6VNL6vZsF7oC6' \
  -H 'Content-Type: application/json' \
  -d @/tmp/folder.json
```

---

## Step 9: Verify Sync Is Working

Wait 30-60 seconds after the Hub accepts your device, then check:

```bash
# Get your API key from config
API_KEY=$(grep -oP '<apikey>\K[^<]+' ~/.config/syncthing/config.xml 2>/dev/null || \
  docker exec openclaw-wiki-sync cat /var/syncthing/config/config.xml 2>/dev/null | \
  grep -oP '<apikey>\K[^<]+')

# Check connections — you should see HLBW742 (Hub) as connected=true
curl -s http://localhost:8384/rest/system/connections \
  -H "X-API-Key: $API_KEY" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for dev_id, info in data.get('connections', {}).items():
    print(f'{dev_id[:20]}... connected={info.get(\"connected\", False)}')
"

# Check folder status — state should be \"idle\", not \"scanning\" or \"error\"
curl -s "http://localhost:8384/rest/db/status?folder=memory" \
  -H "X-API-Key: $API_KEY" | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(f'State: {data.get(\"state\", \"?\")}')
print(f'Local files: {data.get(\"localFiles\", \"?\")}')
print(f'Global files: {data.get(\"globalFiles\", \"?\")}')
if data.get('need'):
    print(f'Still syncing: {data[\"need\"].get(\"files\", 0)} files remaining')
"
```

**Expected output:**
```
HLBW742-2EYCQ7X-NC5M... connected=True
State: idle
Local files: 227
Global files: 227
```

If `connected=False`, check:
- **Firewall:** ports 22000/tcp, 22000/udp, 21027/udp must be open
- **Hub:** Is your device ID registered? (Step 8)
- **Both sides:** Do both configs have each other's device ID?

---

## Step 10: Add Memory Sync Cron

So your agent periodically picks up new memories from other agents:

```bash
openclaw cron add \
  --name "memory-sync-check" \
  --cron "*/30 * * * *" \
  --message "Check memory/ for files modified in last 30 minutes. Run memory_search for recent topics. If significant new content found from other agents (via Syncthing sync), read and incorporate into your current session context. Focus on: new synthesis pages, daily notes, and log entries from other agents."
```

---

## Troubleshooting Reference

### "insufficient space on disk for database"
**Cause:** Less than 1% free disk space.
**Fix:** Expand disk BEFORE deploying. See Step 1.

### "folder path missing" (Docker only)
**Cause:** Config has host path `/root/.openclaw/workspace/memory` but container sees `/var/syncthing/memory`.
**Fix:** Change `<folder path="/var/syncthing/memory">` in config. This is the #1 Docker mistake.

### "CSRF Token Invalid" on API calls
**Cause:** Syncthing's web UI auth blocks REST API.
**Fix:** Edit config.xml directly. Never use the API for initial setup.

### "Failed to acquire lock: is another Syncthing instance already running?"
**Cause:** Orphan Syncthing process holding the lock file.
**Fix:**
```bash
pkill -9 syncthing
systemctl reset-failed syncthing@root
systemctl start syncthing@root
```

### "Start request repeated too quickly"
**Cause:** systemd marked the service as failed after too many restart attempts.
**Fix:**
```bash
systemctl reset-failed syncthing@root
systemctl start syncthing@root
```

---

## Additional Troubleshooting

### Devices show "Disconnected"
1. Both devices added to each other's config?
2. Same folder ID `memory` on both?
3. Firewall open on port 22000?
4. Syncthing running on both?
5. Can both hosts reach each other on port 22000? `nc -zv <hub-ip> 22000`

### Malformed XML / weird tags like `<<`
**Cause:** Used sed to inject device IDs into config.xml.
**Fix:** Always write the complete config with `cat > config.xml << 'EOF'`. Never use sed on XML.

### Sync stuck at ~47%
**Cause:** Folder path mismatch in Docker setup (see "folder path missing" above).
**Fix:** Fix the path, restart container.

### Sync conflict files appearing
**Cause:** Two agents modified the same file simultaneously.
**Fix:** The `.stignore` from Step 6 prevents these from propagating. You can safely delete them:
```bash
find /root/.openclaw/workspace/memory/ -name "*.sync-conflict-*" -delete
```

### `.dreams/` folder showing up from other agents
**Cause:** Agent recall/dream files that are specific to one agent.
**Fix:** Already handled by `.stignore` (Step 6).

---

## The 6 Golden Rules

1. **Never edit config.xml while Syncthing is running** — it overwrites on shutdown
2. **Never use sed on XML** — use heredoc to write the whole file
3. **Check disk space first** — 1% minimum or Syncthing refuses to start
4. **Docker folder path = container path** — `/var/syncthing/memory`, NOT the host path
5. **Enable the systemd service** — `systemctl enable syncthing@root`
6. **Add `.stignore`** — prevents agent-specific junk from polluting the hive
