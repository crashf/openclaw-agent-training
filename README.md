# OpenClaw Agent Training

A training package for onboarding new OpenClaw AI agents. Drop the link to this repo into a chat with any bot to get them up to speed.

## Quick Start

Paste this into a chat with your bot:

```
Read this repo and follow the instructions: https://github.com/crashf/openclaw-agent-training

1. Fetch the GUIDE.md file from that repo using web_fetch or curl
2. Save it to ~/.openclaw/workspace/docs/openclaw-agent-guide.md
3. Add this line to your AGENTS.md under any "Tools" or "Reference" section:
   - **Agent Guide:** Read `docs/openclaw-agent-guide.md` for tool usage, cron, memory management, and common patterns
4. Read the guide and confirm you understand the key points
5. Tell me:
   - What tools/skills you have available
   - What's in your SOUL.md and USER.md
   - What's in your HEARTBEAT.md
   - Your last 2 days of memory files

This is a training update from the fleet admin.
```

## What's Included

- **[GUIDE.md](GUIDE.md)** — The core survival guide covering tools, memory, cron, heartbeats, common patterns, and tips
- **[PROMPT.md](PROMPT.md)** — Ready-to-paste onboarding prompt (just copy and send to your bot)

## About

Built by [Pund-IT](https://pund-it.ca) for managing a fleet of OpenClaw agents. If you're running multiple claws and want them all operating at the same baseline, this is your starting point.
