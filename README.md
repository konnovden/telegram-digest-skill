# Telegram Digest — Claude Code Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates a daily analytical digest from **all** your Telegram channels. One command, premium HTML output.

## What It Does

```
You: /telegram-digest
Claude Code: Collects 500+ messages → Analyzes → Generates premium HTML digest
```

**Output:** A beautiful HTML report with dark/light theme, navigation, priority indicators, and categorized news — opened automatically in your browser.

## Three Modes

| Mode | Cost | Speed | Best For |
|------|------|-------|----------|
| **V2 (2-Pass)** | ~$0.45 | 3-5 min | Daily production use |
| **Claude Code** | **Free** | 5-10 min | One-off analysis |
| **V3 Map-Reduce** | ~$0.50-0.75 | 1-2 min | Experiments |

The **Claude Code mode** is completely free — analysis happens inside the Claude Code conversation itself, no additional API calls needed.

## Prerequisites

1. **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — Anthropic's CLI tool
2. **Telegram session** via [mcp-telegram](https://github.com/nicedouble/mcp-telegram):
   ```bash
   uvx mcp-telegram login
   ```
3. **Telegram API credentials** from https://my.telegram.org — set as environment variables:
   ```bash
   export API_ID=your_api_id
   export API_HASH=your_api_hash
   ```
4. **jq** — `brew install jq` (macOS) or `apt install jq` (Linux)
5. *(Optional, for V2/V3)* **ANTHROPIC_API_KEY** environment variable

## Installation

### 1. Copy the skill file

```bash
mkdir -p ~/.claude/skills/telegram-digest
cp SKILL.md ~/.claude/skills/telegram-digest/SKILL.md
```

### 2. Create the message collection script

Create `scripts/telegram-collect-messages.py` in your project directory:

```python
#!/usr/bin/env python3
"""Collect messages from all Telegram channels for the last N hours."""

import asyncio, json, os, sys
from datetime import datetime, timedelta, timezone
from telethon import TelegramClient
from telethon.tl.types import Channel, Chat

async def main():
    hours_back = 24
    if "--hours" in sys.argv:
        try:
            idx = sys.argv.index("--hours")
            hours_back = int(sys.argv[idx + 1])
        except (IndexError, ValueError):
            print("ERROR: --hours requires a number", file=sys.stderr)
            sys.exit(1)

    api_id = os.environ.get("API_ID")
    api_hash = os.environ.get("API_HASH")
    if not api_id or not api_hash:
        print("ERROR: Set API_ID and API_HASH environment variables", file=sys.stderr)
        print("Get them at https://my.telegram.org", file=sys.stderr)
        sys.exit(1)

    session_path = os.path.expanduser("~/.local/state/mcp-telegram/session")

    print("Telegram Message Collector\n")
    print("Connecting to Telegram...")

    client = TelegramClient(session_path, int(api_id), api_hash)
    await client.connect()

    if not await client.is_user_authorized():
        print("Not authorized. Run: uvx mcp-telegram login", file=sys.stderr)
        sys.exit(1)

    print("Connected to Telegram")
    print("Getting channel list...")

    dialogs = await client.get_dialogs()
    target_dialogs = [d for d in dialogs if isinstance(d.entity, (Channel, Chat))]

    print(f"Found {len(target_dialogs)} channels/groups")
    print(f"Collecting messages for the last {hours_back} hours...\n")

    cutoff_time = datetime.now(timezone.utc) - timedelta(hours=hours_back)
    all_messages = []
    channels_info = []

    for dialog in target_dialogs:
        entity = dialog.entity
        channel_name = dialog.title or dialog.name or "Unknown"
        channel_username = getattr(entity, "username", None)
        channels_info.append({"name": channel_name, "username": channel_username})

        try:
            messages = await client.get_messages(entity, limit=100)
            count = 0
            for msg in messages:
                if not msg.message or msg.date < cutoff_time:
                    continue
                link = f"https://t.me/{channel_username}/{msg.id}" if channel_username else f"https://t.me/c/{entity.id}/{msg.id}"
                all_messages.append({
                    "channel": channel_name,
                    "channelUsername": channel_username,
                    "messageId": msg.id,
                    "text": msg.message,
                    "date": msg.date.isoformat(),
                    "timestamp": int(msg.date.timestamp()),
                    "link": link
                })
                count += 1
            if count > 0:
                print(f"  {channel_name:40s}: {count} messages")
        except Exception as e:
            print(f"  Error reading {channel_name}: {e}", file=sys.stderr)

    await client.disconnect()
    print(f"\nCollected {len(all_messages)} messages")

    output_data = {
        "metadata": {
            "collectedAt": datetime.now().isoformat(),
            "hoursBack": hours_back,
            "totalChannels": len(target_dialogs),
            "totalMessages": len(all_messages)
        },
        "channels": channels_info,
        "messages": all_messages
    }

    os.makedirs("data", exist_ok=True)
    output_path = f"data/telegram-messages-{datetime.now().strftime('%Y-%m-%d')}.json"
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(output_data, f, ensure_ascii=False, indent=2)

    print(f"\nMessages saved: {output_path}")
    print(f"Channels: {len(target_dialogs)}")
    print(f"Messages: {len(all_messages)}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 3. Personalize the skill

Edit `~/.claude/skills/telegram-digest/SKILL.md` and fill in the **"Personalization Context"** section with your own company, interests, and high-value authors. This determines how the digest prioritizes content for you.

### 4. Run it

```bash
claude
# then type:
/telegram-digest
```

## How It Works (Claude Code Mode)

```
Telegram API ──→ JSON (500+ messages)
                   │
                   ▼
            jq grouping by channel
                   │
                   ▼
         Claude Code analyzes batches
          (no extra API calls!)
                   │
                   ▼
           Synthesis + categorization
                   │
                   ▼
         Premium HTML with dark/light
         theme, navigation, animations
                   │
                   ▼
              Opens in browser
```

## Customization

### Categories
Default categories in the digest:
- Tech & AI
- Business & Startups
- Finance & Markets
- Geopolitics & World
- Local / Regional
- Skip (low priority)

Edit the SKILL.md to change categories to match your interests.

### Digest Language
The default SKILL.md produces digests in the language of the source content. To force a specific language, add a note in the "Important Rules" section of SKILL.md.

### High-Value Authors
Configure authors whose posts should always be prioritized in the "Personalization Context" section.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Session not found" | Run `uvx mcp-telegram login` |
| "Database is locked" | Stop mcp-telegram or copy session to `/tmp/` |
| "API_ID not set" | Export `API_ID` and `API_HASH` env vars |
| Rate limit (429) | Script auto-retries with exponential backoff |
| Too many messages | Reduce `--hours` parameter |

## License

MIT

## Credits

Built with [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Telethon](https://github.com/LonamiWebs/Telethon), and [mcp-telegram](https://github.com/nicedouble/mcp-telegram).
