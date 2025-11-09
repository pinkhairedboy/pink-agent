# Pink Agent

Cross-platform Telegram agent for remote Claude Code control. Send text or voice messages from anywhere, execute commands on your machine.

## Architecture

Two independent background processes communicate via JSONL queues:

- **pink-claude-agent** - Claude Code execution agent
- **pink-telegram-agent** - Telegram agent (receiver + sender with infinite retry)

IPC through persistent queues:
- `commands.jsonl` - Telegram → Claude
- `responses.jsonl` - Claude → Telegram

## Requirements

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) package manager
- Claude Code CLI (`claude` in PATH)
- [pink-transcriber](https://github.com/pinkhairedboy/pink-transcriber) (optional, for voice messages)

## Installation

```bash
# Install uv if not installed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Install globally in editable mode (for pink-agent send CLI)
uv tool install -e .

# Run setup (creates ~/.claude/CLAUDE.md and installs /summarize command)
./setup.sh

# Configure Telegram credentials
cp .env.example .env
nano .env
# Edit:
#   TELEGRAM_BOT_TOKEN (from @BotFather)
#   TELEGRAM_USER_ID (from @userinfobot)
```

## Usage

**Run the bot:**
```bash
# macOS (prevents sleep on battery)
caffeinate -id uv run pink-agent

# Linux
uv run pink-agent
```

**Development mode (verbose logging):**
```bash
# macOS
VERBOSE=1 caffeinate -id uv run pink-agent

# Linux
VERBOSE=1 uv run pink-agent
```

**Telegram commands:**
- `/new` - Reset Claude context
- `/compact` - Manually trigger auto-compact
- `/restart` - Auto-update and restart (runs `git pull` + `uv sync` before restart)

**CLI Tools:**

`pink-agent send` - Send messages and files to user via Telegram bot (for Claude Code to communicate directly)

```bash
# Send text message
pink-agent send "message"

# Send file
pink-agent send -f screenshot.png

# Send text with file(s)
pink-agent send "caption" -f photo.jpg -f document.pdf

# Send multiple files
pink-agent send -f file1.txt -f file2.txt
```

**Monitoring:**
```bash
ls -la *.jsonl  # Check queue status
# Empty queues = healthy
# Growing responses.jsonl = Telegram stuck
```

## How it works

**Message flow:**

1. Voice/text → Telegram
2. pink-telegram-agent: transcribe (if voice) → `commands.jsonl`
3. pink-claude-agent: read queue → execute Claude → `responses.jsonl`
4. pink-telegram-agent: read queue → send to Telegram (infinite retry)

**Fault isolation:**
- Claude crash → Telegram continues accepting messages
- Telegram API down → infinite retry, Claude continues processing
- All queues persisted on disk → nothing lost on crash

## Structure

```
pink-agent/
├── commands.jsonl           # IPC: Telegram → Claude
├── responses.jsonl          # IPC: Claude → Telegram
└── src/pink_agent/
    ├── config.py            # Global configuration
    ├── daemon/              # Process management
    │   ├── supervisor.py    # Main supervisor
    │   └── singleton.py     # Single instance enforcement
    ├── cli/                 # CLI tools
    │   └── send.py          # pink-agent send (direct message/file sending)
    ├── claude/              # Claude Code agent
    │   ├── agent.py         # Entry point
    │   ├── executor.py      # Claude Code subprocess
    │   ├── sessions.py      # Session + auto-compact
    │   ├── processor.py     # Command processing loop
    │   └── output.py        # Tool output formatting
    ├── telegram/            # Telegram agent
    │   ├── agent.py         # Entry point
    │   ├── receiver.py      # Messages → commands.jsonl
    │   ├── sender.py        # responses.jsonl → Telegram
    │   ├── output.py        # Markdown formatting
    │   ├── transcriber.py   # pink-transcriber client
    │   ├── files.py         # File attachment handling
    │   └── commands.py      # /start /new /compact /restart
    └── queue/               # Shared IPC
        ├── storage.py       # CRUD + fcntl locks
        └── watcher.py       # Cross-platform file monitoring
```

## Tech Stack

- Python 3.12+
- uv - fast Python package manager
- python-telegram-bot - async Telegram API
- watchfiles - high-performance file monitoring (Rust-based)
- asyncio - concurrent tasks
- JSONL - persistent queues (fcntl-locked)
- multiprocessing - isolated command execution
- subprocess - Claude Code CLI
