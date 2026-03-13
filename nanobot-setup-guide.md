# 🐈 Nanobot Local AI Agent — Complete Setup Guide

> A fully local AI agent running on macOS M-series with Telegram integration, automated news digests, and job search automation — using Ollama and free public APIs.

---

## Table of Contents
1. [System Requirements](#system-requirements)
2. [Installation](#installation)
3. [Configuring Ollama as the Model Provider](#configuring-ollama-as-the-model-provider)
4. [Telegram Bot Setup](#telegram-bot-setup)
5. [Starting the Gateway](#starting-the-gateway)
6. [SOUL.md — Behavior Instructions](#soulmd--behavior-instructions)
7. [Automated News Digest](#automated-news-digest)
8. [Job Search Automation](#job-search-automation)
9. [Scheduling with launchd](#scheduling-with-launchd)
10. [Problems Faced & Solutions](#problems-faced--solutions)
11. [Final Setup Summary](#final-setup-summary)

---

## System Requirements

| Component | Details |
|-----------|---------|
| Hardware | MacBook Air M-series, 16GB RAM |
| OS | macOS (Ventura/Sonoma) |
| Python | 3.12 (via miniforge/conda) |
| Ollama | Installed with local models |
| Docker | Not required for nanobot |

### Ollama Models Available
| Model | Size | Tool Support |
|-------|------|-------------|
| qwen2.5:7b | ~4.5GB | ✅ Best local tool-use |
| qwen2.5:14b | ~8GB | ✅ Better but slow on 16GB |
| llama3.1-fast | 4.9GB | ✅ Limited |
| gpt-oss:120b-cloud | Cloud | ✅ Fast, no local RAM |

---

## Installation

### Step 1 — Install nanobot
```bash
pip install nanobot-ai
```

**Why:** nanobot-ai is a lightweight Python-based local AI agent framework that supports Ollama, Telegram, cron jobs, and tool execution with minimal setup.

### Step 2 — Run onboarding
```bash
nanobot onboard
```

**Why:** Creates the config file at `~/.nanobot/config.json` and workspace files including `SOUL.md`, `MEMORY.md`, `AGENTS.md`.

> ⚠️ Note: `nanobot init` does NOT exist. The correct command is `nanobot onboard`.

### Step 3 — Verify installation
```bash
nanobot status
```

---

## Configuring Ollama as the Model Provider

By default nanobot uses `anthropic/claude-opus-4-5`. We switched to a local Ollama model to run fully offline.

Open the config:
```bash
open ~/.nanobot/config.json
```

### Changes made to `~/.nanobot/config.json`

**1. Change the model and provider:**
```json
"model": "qwen2.5:7b",
"provider": "custom",
```

**Why `custom` instead of `ollama`?** Nanobot does not have a built-in `ollama` provider name. Using `custom` with the Ollama OpenAI-compatible endpoint works correctly.

**2. Configure the custom provider:**
```json
"providers": {
  "custom": {
    "apiKey": "ollama",
    "apiBase": "http://localhost:11434/v1"
  }
}
```

**Why `apiKey: "ollama"`?** Ollama's API requires a non-empty API key string but doesn't validate it. Any string works.

### Test the setup
```bash
nanobot agent -m 'Hello'
```

Expected output:
```
🐈 nanobot
Hello! How can I assist you today?
```

### Model Selection Notes

| Model | Speed | RAM Usage | Tool Reliability |
|-------|-------|-----------|-----------------|
| qwen2.5:7b | Fast | ~6GB | Medium |
| qwen2.5:14b | Slow | ~17GB | Good |
| gpt-oss:120b-cloud | Very Fast | 0 (cloud) | Good |

**Final choice:** `gpt-oss:120b-cloud` for general use via Telegram (fast, no RAM usage).  
**Use `qwen2.5:7b` locally** for sensitive queries that should not leave your machine.

To switch models, update `config.json` and restart the gateway:
```json
"model": "gpt-oss:120b-cloud",
```

---

## Telegram Bot Setup

### Step 1 — Create a bot
1. Open Telegram → search `@BotFather`
2. Send `/newbot`
3. Follow prompts — note the bot token

### Step 2 — Get your Telegram user ID
Send a message to `@userinfobot` — it returns your numeric user ID.

### Step 3 — Configure Telegram in `~/.nanobot/config.json`
```json
"telegram": {
  "enabled": true,
  "token": "YOUR_BOT_TOKEN",
  "allowFrom": ["YOUR_TELEGRAM_USER_ID"],
  "proxy": null,
  "replyToMessage": false
}
```

**Why `allowFrom`?** Restricts the bot to respond only to your Telegram user ID, preventing others from using your agent.

> ⚠️ Security: Never share your bot token publicly. If exposed, revoke it immediately via BotFather → `/mybots` → API Token → Revoke.

---

## Starting the Gateway

```bash
nanobot gateway
```

**Why `gateway` and not `start`?** Nanobot does not have a `start` command. `gateway` is the correct command to start the agent with all channels (Telegram, etc.) active.

### Available commands
```bash
nanobot --help        # Show all commands
nanobot gateway -v    # Verbose output
nanobot gateway -p 18790  # Custom port
nanobot status        # Check configuration
nanobot agent -m 'text'  # Direct CLI interaction
```

### Run as background service (launchd)
```bash
# Start
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.nanobot.gateway.plist

# Stop
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.nanobot.gateway.plist
```

---

## SOUL.md — Behavior Instructions

`~/.nanobot/SOUL.md` contains persistent behavior guidelines for the agent.

Create it:
```bash
cat > ~/.nanobot/SOUL.md << 'EOF'
# Behavior Guidelines

## Weather
When asked about weather, always use exec to run:
- Current: /usr/bin/curl -s "https://wttr.in/CITY?format=3"
- Forecast: /usr/bin/curl -s "https://wttr.in/CITY?format=v2"
Replace CITY with the requested location. Never use web_search or OpenWeatherMap.

## General
Be concise. Keep responses short unless asked for detail.
EOF
```

**Why SOUL.md?** Without explicit instructions, local models often attempt to use non-existent "skills" or web search tools. SOUL.md tells the agent exactly which tools and commands to use for common tasks.

---

## Automated News Digest

Three separate Python scripts deliver categorized news to Telegram automatically.

### News Sources

**Geopolitical (7 feeds):**
- BBC World: `https://feeds.bbci.co.uk/news/world/rss.xml`
- BBC Politics: `https://feeds.bbci.co.uk/news/politics/rss.xml`
- DW World: `https://rss.dw.com/rdf/rss-en-world`
- Al Jazeera: `https://www.aljazeera.com/xml/rss/all.xml`
- Sky News: `https://feeds.skynews.com/feeds/rss/world.xml`
- DW Top: `https://rss.dw.com/rdf/rss-en-top`
- TIME World: `https://feeds.feedburner.com/time/world`

**Entertainment (4 feeds):**
- BBC Entertainment: `https://feeds.bbci.co.uk/news/entertainment_and_arts/rss.xml`
- Variety: `https://variety.com/feed/`
- Deadline: `https://deadline.com/feed/`
- Bollywood Hungama: `https://www.bollywoodhungama.com/feed/`

**AI Updates (8 feeds):**
- MIT Tech Review: `https://www.technologyreview.com/feed/`
- VentureBeat AI: `https://venturebeat.com/ai/feed/`
- Google AI Blog: `https://blog.google/technology/ai/rss/`
- DeepMind: `https://deepmind.com/blog/feed/basic/`
- Hugging Face: `https://huggingface.co/blog/feed.xml`
- arXiv cs.AI: `https://arxiv.org/rss/cs.AI`
- Google AI (legacy): `https://feeds.feedburner.com/blogspot/gJZg`
- AI News: `https://www.artificialintelligence-news.com/feed/`

### News Categorization Logic

Stories are categorized by keywords in the headline:

| Category | Keywords |
|----------|---------|
| 🚨 Breaking | attack, war, crisis, killed, explosion, coup, arrest, sanctions |
| 🔥 Viral | viral, trending, millions, record, celebrity, wins, banned |
| 📌 Notable | election, law, policy, president, minister, government, court |
| 😮 Unusual | bizarre, rare, shocking, unexpected, mystery, smuggl, weird |

### Script Files
| File | Category | Schedule |
|------|----------|----------|
| `./scripts/news_geo.py` | 🌍 Geopolitical | Every 3 hours |
| `./scripts/news_ent.py` | 🎬 Entertainment | Daily 08:00 CET |
| `./scripts/news_ai.py` | 🤖 AI Updates | Daily 09:00 CET |

### Run manually
```bash
python3 scripts/news_geo.py
python3 scripts/news_ent.py
python3 scripts/news_ai.py
```

---

## Job Search Automation

Uses the **Bundesagentur für Arbeit (Federal Employment Agency) API** — free, no API key required, covers the entire German job market.

### Why Arbeitsagentur API?
- ✅ Free and public
- ✅ No API key or registration needed
- ✅ Official German job listings
- ✅ Returns structured JSON
- ❌ LinkedIn — requires JavaScript/login
- ❌ StepStone — blocked
- ❌ Xing — returns HTML instead of RSS

### API Endpoint
```
https://rest.arbeitsagentur.de/jobboerse/jobsuche-service/pc/v4/jobs
```

**Parameters:**
- `was` — job title / keywords
- `wo` — location
- `umkreis` — radius in km
- `angebotsart=1` — full-time jobs

### Search Queries Used
```python
SEARCHES = [
    ("System Test Engineer", "Bayern"),
    ("HIL Test Engineer", "Bayern"),
    ("SIL Test Engineer", "Bayern"),
    ("Embedded Test Engineer", "Bayern"),
    ("Software Test Engineer", "Ingolstadt"),
    ("ECU Test Engineer", "Bayern"),
    ("Testingenieur Embedded", "Bayern"),
    ("HIL SIL", "Bayern"),
    ("CANoe Test", "Bayern"),
    ("ASPICE Test", "Bayern"),
    ("Manual Test Engineer", "Bayern"),
    ("Automation Test Engineer", "Bayern"),
]
```

**Why multiple searches?** The API searches by exact keyword match. Using multiple relevant terms ensures broader coverage of job listings relevant to automotive embedded testing.

### Script File
- `./scripts/jobs_digest.py` — fetches and sends job listings to Telegram
- Scheduled daily at **07:00 CET** via launchd

---

## Scheduling with launchd

**Why launchd instead of cron?**  
macOS restricts cron from accessing the filesystem without Full Disk Access permissions, which requires manual security settings. launchd is the native macOS scheduler and works without extra permissions.

### Active launchd Jobs

| Label | Script | Schedule |
|-------|--------|----------|
| `com.nanobot.news.geo` | `scripts/news_geo.py` | Every 3 hours |
| `com.nanobot.news.ent` | `scripts/news_ent.py` | Daily 07:00 UTC (08:00 CET) |
| `com.nanobot.news.ai` | `scripts/news_ai.py` | Daily 08:00 UTC (09:00 CET) |
| `com.nanobot.jobs` | `scripts/jobs_digest.py` | Daily 06:00 UTC (07:00 CET) |

### Useful Commands
```bash
# Check all running jobs
launchctl list | grep nanobot

# Stop a job
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.nanobot.news.geo.plist

# Start a job
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.nanobot.news.geo.plist

# Check logs
cat /tmp/com.nanobot.news.geo.log
cat /tmp/com.nanobot.news.ent.log
cat /tmp/com.nanobot.news.ai.log
cat /tmp/com.nanobot.jobs.log
```

> ⚠️ Timezone note: launchd uses UTC. Germany is CET (UTC+1) in winter and CEST (UTC+2) from March 29. Update plist Hour values after the clock change.

---

## Problems Faced & Solutions

### 1. `nanobot init` command not found
**Problem:** Tried `nanobot init` to initialize the workspace.  
**Solution:** The correct command is `nanobot onboard`. There is no `init` command.

---

### 2. No API key configured error
**Problem:** `nanobot agent -m 'Hello'` returned `Error: No API key configured`.  
**Root cause:** The `ollama` provider name is not built into nanobot.  
**Solution:** Use `"provider": "custom"` with `apiBase: http://localhost:11434/v1` and `apiKey: "ollama"`.

---

### 3. Model not responding via Telegram (only CLI)
**Problem:** `nanobot agent` worked fine in terminal but Telegram returned `I've completed processing but have no response to give.`  
**Root cause:** `qwen2.5:7b` was completing too fast (5 seconds) without executing tools, silently failing.  
**Solution:** Send more explicit commands like `run /usr/bin/curl -s "..."` to force direct tool execution. Also update SOUL.md with explicit instructions.

---

### 4. qwen2.5:14b slowing down the Mac
**Problem:** After pulling the 14B model, the Mac became unusably slow with 17GB RAM consumed.  
**Solution:** Stopped the model with `ollama stop qwen2.5:14b` and switched to `gpt-oss:120b-cloud` which uses zero local RAM.

---

### 5. cron jobs not running on macOS
**Problem:** Added cron job via `crontab -e` but log file never appeared, job never ran.  
**Root cause:** macOS requires Full Disk Access for cron, which must be manually granted in System Settings.  
**Solution:** Switched to **launchd** (`~/Library/LaunchAgents/*.plist`) which works without extra permissions.

---

### 6. News digest sending duplicate messages
**Problem:** News was arriving twice every 30 minutes.  
**Root cause:** Both `crontab` and `launchd` were running the same script simultaneously.  
**Solution:** Removed crontab with `crontab -r` and kept only launchd.

---

### 7. Telegram HTTP 400 error when sending news
**Problem:** News script failed with `HTTP Error 400: Bad Request` when sending to Telegram.  
**Root cause:** Two issues combined:
  - Message exceeded Telegram's 4096 character limit
  - HTML tags (`<b>`, `<i>`) in message with `parse_mode: HTML` but titles contained unescaped special characters (`&`, `'`, `"`)  

**Solution:**
  - Split messages into 4000-character chunks
  - Removed all HTML tags from messages and sent as plain text without `parse_mode`

---

### 8. Split scripts containing unwanted send() calls
**Problem:** `news_ent.py` was also sending geopolitical news.  
**Root cause:** The script split logic used `rfind('send(build_message')` which found the wrong position, including extra send calls from the original combined script.  
**Solution:** Used `find('\ngeo_items = fetch')` to find the exact split point before any send calls.

---

### 9. zsh `!` character breaking commands
**Problem:** Commands with `!` in strings caused `zsh: event not found` errors.  
**Solution:** Always use **single quotes** `'` instead of double quotes `"` in zsh for strings containing `!`.

---

### 10. Xing and StepStone RSS blocked
**Problem:** Both returned 200 status but delivered HTML instead of RSS/XML.  
**Root cause:** Both platforms block programmatic access and require JavaScript rendering or login.  
**Solution:** Switched entirely to **Bundesagentur für Arbeit API** which is public, free, and returns clean JSON with 25+ relevant jobs per search.

---

### 11. `nanobot cron` command not found
**Problem:** `nanobot cron add` returned `No such command 'cron'`.  
**Root cause:** The installed version of nanobot-ai does not include a cron subcommand.  
**Solution:** Used macOS **launchd** for all scheduling instead.

---

### 12. launchd plist load failed (I/O error)
**Problem:** `launchctl load` returned `Load failed: 5: Input/output error`.  
**Root cause:** `launchctl load` is deprecated on newer macOS versions.  
**Solution:** Use `launchctl bootstrap gui/$(id -u) <plist_path>` instead.

---

## Final Setup Summary

```
~/.nanobot/
├── config.json          # Main config (model, Telegram, providers)
├── SOUL.md              # Agent behavior instructions
└── workspace/           # Agent workspace files

path/to/NanoBotAI/
├── scripts/
│   ├── news_geo.py      # Geopolitical news → Telegram every 3hrs
│   ├── news_ent.py      # Entertainment news → Telegram 08:00 CET
│   ├── news_ai.py       # AI updates → Telegram 09:00 CET
│   └── jobs_digest.py   # Job listings → Telegram 07:00 CET
└── launchd/
    ├── com.nanobot.news.geo.plist
    ├── com.nanobot.news.ent.plist
    ├── com.nanobot.news.ai.plist
    └── com.nanobot.jobs.plist
```

### Quick Reference

| Task | Command |
|------|---------|
| Start nanobot | `nanobot gateway` |
| Test agent | `nanobot agent -m 'Hello'` |
| Check status | `nanobot status` |
| Check jobs | `launchctl list \| grep nanobot` |
| Run news manually | `python3 scripts/news_geo.py` |
| Run jobs manually | `python3 scripts/jobs_digest.py` |
| Check logs | `cat /tmp/com.nanobot.news.geo.log` |

---

*Setup completed: March 2026 | macOS M-series | nanobot-ai + Ollama + Telegram*
