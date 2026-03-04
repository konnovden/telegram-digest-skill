---
name: telegram-digest
version: 3.1.0
description: |
  Generates a daily analytical digest from all your Telegram channels.
  Three modes: V2 (2-Pass) stable production-ready, Claude Code analysis (free), V3 Map-Reduce (experimental).
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - TodoWrite
---

# Telegram Digest Generator v3.1

This skill generates an automatic daily digest from all your Telegram channels.

## Three Operating Modes

### Mode 1: V2 (2-Pass) - Production-Ready (recommended)
- **Stable** - proven architecture with 8 failure points eliminated
- **Fixed format** - HTML template engine (no design drift)
- **Automatic retry** - exponential backoff on rate limits
- **Lock management** - prevents concurrent execution
- **Pagination** - 500 messages per channel (doesn't lose old posts)
- **Logging** - structured logs in `~/.local/share/telegram-digest/logs/`
- **Early validation** - fail-fast for missing dependencies
- **Cost:** ~$0.45 per digest
- **Time:** ~3-5 minutes

**Architecture:**
- **Pass 1 (Haiku):** Filter 600+ messages -> 150 relevant (~$0.01)
- **Pass 2 (Sonnet):** Deep analysis + synthesis -> final digest (~$0.44)

### Mode 2: Claude Code Analysis (FREE)
- **Free** - no Anthropic API calls
- Analysis in conversation with incremental summaries
- Batch processing for context management
- Slower (~5-10 minutes for 70+ groups)
- Manual context management

### Mode 3: V3 Map-Reduce (experimental)
- **Paid** - uses Anthropic API (~$0.50-0.75 per full report)
- Faster (~1-2 minutes)
- Parallel group processing
- Rate limit 5 requests/minute (may require retries)
- Format may drift

## How to Use

### Basic Run (V2 Production-Ready)
```bash
/telegram-digest
```

Uses V2 (2-Pass) by default - stable and reliable.

### With Mode Selection
```bash
/telegram-digest --mode=v2          # V2 (2-Pass) - production-ready (default)
/telegram-digest --mode=claude      # Claude Code Analysis (free, slower)
/telegram-digest --mode=api         # V3 Map-Reduce (experimental)
/telegram-digest --hours 48         # Last 48 hours
/telegram-digest --output custom.html  # Custom filename
```

## Prerequisites

### Required
1. **Claude Code** subscription
2. **Telegram session** via [mcp-telegram](https://github.com/nicedouble/mcp-telegram):
   ```bash
   uvx mcp-telegram login
   ```
3. **Telegram API credentials** - get from https://my.telegram.org:
   - Set `API_ID` and `API_HASH` environment variables
4. **jq** installed (`brew install jq` on macOS)

### Optional (for V2/V3 modes)
5. **ANTHROPIC_API_KEY** environment variable
6. **Node.js 18+** for running V2/V3 scripts

### Setup
Create a `.env` file in your project directory:
```bash
API_ID=your_telegram_api_id
API_HASH=your_telegram_api_hash
ANTHROPIC_API_KEY=your_anthropic_key  # optional, for V2/V3 modes
```

## Execution Instructions

**Before starting:**
1. Use TodoWrite for tracking (3-5 steps for V2, 6-7 for Claude Code)
2. Parse arguments `--hours` (default 24) and `--mode` (default "v2")
3. Determine today's date for filenames

**Default mode: V2 (2-Pass)**
- If user didn't specify `--mode`, use V2
- V2 is the most stable and reliable for production
- Only requires running the script, everything else is automatic

### Step 1: Collect Messages from Telegram

```bash
cd $PROJECT_DIR

# Basic run (24 hours)
uvx --with telethon python scripts/telegram-collect-messages.py

# With parameters
uvx --with telethon python scripts/telegram-collect-messages.py --hours 48
```

The script saves: `data/telegram-messages-YYYY-MM-DD.json`

**NOTE:** The collection script requires `API_ID` and `API_HASH` environment variables. It reads the Telegram session from `~/.local/state/mcp-telegram/session`. Make sure mcp-telegram is not running simultaneously (session lock conflict). If it is, copy the session file to a temp location and modify the script path.

### Step 2: Verify JSON

```bash
ls -lh data/telegram-messages-*.json | tail -1
jq '.metadata' data/telegram-messages-$(date +%Y-%m-%d).json
```

### Step 3A: V2 Mode (2-Pass) - Production-Ready

**RECOMMENDED:** Stable architecture with all known issues resolved.

```bash
cd $PROJECT_DIR

# Basic run (24 hours)
node scripts/telegram-digest-v2.js

# Custom period
node scripts/telegram-digest-v2.js --hours 48

# Custom filename
node scripts/telegram-digest-v2.js --output custom-digest.html
```

**What V2 does:**

1. **Pass 1 (Haiku filter):** Filters 600+ messages -> 150 relevant (~$0.01)
2. **Pass 2 (Sonnet synthesis):** Deep analysis + synthesis -> final digest (~$0.44)

**Key features:**

- **Lock file:** Prevents concurrent execution (`/tmp/telegram-digest.lock`)
- **Early validation:** Checks ANTHROPIC_API_KEY, Telegram session, config, template
- **Retry logic:** Exponential backoff (1s, 2s, 4s) on rate limits
- **Template engine:** Uses fixed HTML template (`/templates/digest-template.html`)
- **Pagination:** 500 messages per channel (doesn't lose old posts)
- **Logging:**
  - Session logs: `~/.local/share/telegram-digest/logs/digest.log`
  - Summary: `~/.local/share/telegram-digest/logs/summary.jsonl`
  - Errors: `~/.local/share/telegram-digest/logs/errors.jsonl`
- **Notifications:** macOS notifications on critical errors

### Step 3B: Claude Code Analysis Mode (no API)

**CRITICAL:** Analyze ALL messages, don't skip anything by channel name/topic!

1. **Group by author/channel:**

```bash
cd $PROJECT_DIR

# Group messages by channelUsername
jq '.messages | group_by(.channelUsername) | map({
  author: (.[0].channelUsername | if . == null then "unknown" else . end),
  channel: .[0].channel,
  count: length,
  messages: map({text, date, link, messageId})
}) | sort_by(-.count)' data/telegram-messages-$(date +%Y-%m-%d).json > /tmp/message-groups.json

# Check number of groups
jq '. | length' /tmp/message-groups.json
```

2. **Batch Analysis (in parts):**

**Strategy:**
- Batch 1: 12 groups with the most messages (detailed analysis)
- Batch 2: Next 12 groups (detailed analysis)
- Batch 3-N: Remaining groups (brief summaries for 1-2 messages)

**Batch 1 (0-11):**
```bash
# Extract first 12 groups
jq '.[0:12]' /tmp/message-groups.json > /tmp/batch1-groups.json
```

Read via Read tool `/tmp/batch1-groups.json`, analyze EACH group:

For each group create a structured summary:
```json
{
  "groupId": 0,
  "author": "channel_username",
  "channel": "Channel Name",
  "messageCount": 99,
  "summary": "Factual description (2-3 paragraphs)",
  "keyTopics": ["topic1", "topic2"],
  "importance": "high | medium | low",
  "insights": ["insight 1", "insight 2"],
  "sourceLink": "https://t.me/channel_username"
}
```

Save summaries:
```bash
# Via Write tool create /tmp/digest-summaries-batch1.json
```

**Batch 2 (12-23):**
Repeat process for groups 12-23.

**Batch 3-N (24+):**
For groups with 1-2 messages use a shorter format:
```json
{
  "groupId": 24,
  "author": "channel_username",
  "count": 3,
  "summary": "Brief factual description in 1-2 sentences",
  "importance": "low",
  "sourceLink": "https://t.me/channel_username"
}
```

3. **Final Digest Synthesis:**

Read ALL saved summaries (`batch1.json`, `batch2.json`, `batch3.json`) and synthesize the final digest:

- **System synthesis** (2-3 paragraphs): overall picture of the day, key trends
- **Main events** (10-15 items): high importance with details
- **Categories:**
  - Tech & AI
  - Business & Startups
  - Finance & Markets
  - Geopolitics & World
  - Local / Regional
  - Skip (low priority)

**Style:** Factual news (not prescriptive recommendations), close to original content.

### Step 3C: V3 Mode Map-Reduce (experimental)

```bash
cd $PROJECT_DIR

# Run with API
node scripts/telegram-digest-v3.js
```

**Rate Limits:** 5 requests/minute. The script automatically retries on 429 errors.

**Format may drift:** V3 doesn't use a fixed template. For production use V2.

### Step 4: Generate HTML (Premium Design)

**IMPORTANT:** Always use premium design with dark theme, navigation, and animations!

Based on the synthesized digest, create an HTML file:

```javascript
// HTML structure
{
  synthesis: "System analysis of the day...",
  highlights: [
    {
      urgency: "urgent" | "high",
      title: "Title",
      description: "Detailed description",
      sourceLink: "https://t.me/..."
    }
  ],
  tech: [...],        // Tech & AI items
  business: [...],    // Business items
  finance: [...],     // Finance items
  geopolitics: [...], // Geopolitics items
  local: [...],       // Local/Regional items
  skip: ["tag1", "tag2"]  // Skipped low-priority tags
}
```

Use Write tool to create:
- `data/telegram-digest-YYYY-MM-DD-full.html`

**Required premium design features:**

1. **Dark/Light Theme Toggle**
   - Fixed button top-left
   - Save choice in localStorage
   - CSS variables for both themes

2. **Progress Bar**
   - Fixed at top (3px)
   - Gradient blue -> purple
   - Animation on scroll

3. **Fixed Navigation**
   - Right side with section icons
   - Active state on scroll
   - Smooth scroll to sections

4. **Premium styles:**
   - Gradient backgrounds for hero
   - Card hover effects with transform
   - Priority dots (urgent with pulsation)
   - FadeInUp animations for sections
   - JetBrains Mono for statistics
   - Inter font for everything else

5. **CSS Variables:**
   ```css
   --bg-primary, --bg-secondary, --bg-tertiary
   --text-primary, --text-secondary, --text-muted
   --accent-blue, --accent-purple, --accent-green, etc.
   --radius-sm/md/lg/xl
   --shadow-card, --shadow-hover
   ```

6. **Responsive Design**
   - Hide navigation on mobile
   - Adaptive font sizes
   - Reduced padding

**JavaScript functions:**
- Theme toggle with localStorage
- Progress bar on scroll
- Active navigation state
- Smooth scroll to sections

### Step 5: Show Result

```
Digest created!

Statistics:
   - Channels: N
   - Messages: N
   - Groups analyzed: N
   - Main events: N
   - Tech & AI: N
   - Business: N
   - Finance: N
   - Geopolitics: N
   - Local: N

File: telegram-digest-YYYY-MM-DD-full.html
Cost: $0.00 (Claude Code analysis)

Design: Premium (dark/light theme, navigation, animations)

Mode: Claude Code Analysis (no API)
```

## Personalization Context

> **IMPORTANT:** Customize this section to match YOUR interests and professional context.
> This determines how the digest prioritizes and categorizes content.

**Your Company (example — replace with yours):**
- What your company does
- Key competitors
- Target audience
- Current focus areas and markets

**Your Interests (example — replace with yours):**
- AI and content automation
- Industry-specific technologies
- Marketing automation
- Competitor intelligence
- Local topics (taxes, regulations in your country)
- Global fintech trends
- Geopolitics affecting business

**High-Value Authors (optional):**
You can configure a list of Telegram authors whose posts should always be flagged as important:
```json
{
  "highValueAuthors": ["author1", "author2", "author3"]
}
```

## Important Rules

1. **Don't skip channels** by name/topic — the user is intentionally subscribed to each channel
2. **All text in the user's language** (synthesis, descriptions, ratings)
3. **Factual style** — write what happened, not prescriptive advice
4. **Links to original** for each item
5. **Highlights structure** — not "priorities/opportunities", but "main events" + categories
6. **Categorization** by topics (AI, Business, Finance, Geopolitics, Local)
7. **Highlight key points** with `<strong>` and `<span class="highlight">`

## Error Handling

### "Telegram session not found"
```
Telegram session not found.

Log in via MCP:
uvx mcp-telegram login
```

### "Rate limit 429" (API mode)
```
Rate limit reached (5 req/min).
Script automatically retries...
```

### "Session database is locked"
```
Another process (mcp-telegram) is using the session.
Copy the session file to /tmp/ and use the copy:
cp ~/.local/state/mcp-telegram/session.session /tmp/telegram-session-copy.session
```

## Mode Comparison

| Criteria | V2 (2-Pass) | Claude Code | V3 Map-Reduce |
|----------|-------------|-------------|---------------|
| **Cost** | ~$0.45 | $0 | ~$0.50-0.75 |
| **Speed** | 3-5 min | 5-10 min | 1-2 min |
| **Stability** | Production | Stable | Experimental |
| **HTML Format** | Fixed | Fixed | May drift |
| **Rate limits** | Retry logic | None | 5 req/min |
| **Lock file** | Yes | No | No |
| **Logging** | Structured | Console | Console |
| **Pagination** | 500 msgs | All in memory | All in memory |
| **Quality** | Excellent | Excellent | Excellent |
| **Use case** | Daily production | One-off analysis | Experiments |

**Recommendations:**

- **V2 (2-Pass)** - for daily production use (reliability + quality)
- **Claude Code** - for one-off analysis without API costs
- **V3 Map-Reduce** - for experiments when maximum speed is needed

## Final Checklist

### V2 (2-Pass) Mode
- [ ] Telegram session active
- [ ] ANTHROPIC_API_KEY set
- [ ] telegram-digest-v2.js script ran
- [ ] Digest HTML created
- [ ] Statistics shown (channels, messages, highlights, cost)
- [ ] Opened in browser to verify format

### Claude Code Analysis Mode
- [ ] Collection script ran
- [ ] JSON read and grouped
- [ ] All groups analyzed (no skips!)
- [ ] Summaries saved incrementally
- [ ] Final digest synthesized
- [ ] HTML created
- [ ] Statistics shown

### V3 Map-Reduce Mode
- [ ] telegram-digest-v3.js script ran
- [ ] Rate limits handled (retry on 429)
- [ ] HTML created
- [ ] Format checked for drift

---

## Changelog

**v3.1.0** - V2 Stabilization
- Added V2 (2-Pass) as recommended production-ready mode
- Lock file management (prevents concurrent execution)
- Template engine (prevents format drift)
- Retry logic with exponential backoff
- Pagination (500 messages per channel)
- Structured logging (session logs + summary + errors)
- Early validation (fail-fast for missing dependencies)
- Error handling with macOS notifications
- All 8 known failure points addressed

**v3.0.0** - Initial release
- Claude Code Analysis mode (free)
- V3 Map-Reduce mode (experimental)
- Premium HTML design
