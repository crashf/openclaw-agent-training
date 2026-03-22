# Recommended OpenClaw Config Additions

Add these to your `agents.defaults` section in `/root/.openclaw/openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "memoryFlush": {
          "enabled": true
        }
      },
      "memorySearch": {
        "experimental": {
          "sessionMemory": true
        }
      }
    }
  }
}
```

These settings help with:
- **memoryFlush**: Automatically saves session context to disk when near token limit, preventing memory loss
- **sessionMemory**: Enables searching across past session memory for better context recall

## Heartbeat Configuration

Also ensure `HEARTBEAT.md` exists in your workspace with tasks to run during heartbeat polls. See `HEARTBEAT.md` in this repo for a template.
