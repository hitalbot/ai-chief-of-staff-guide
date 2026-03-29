---
layout: default
title: AI Chief of Staff Setup Guide
---

# How to Set Up an AI Chief of Staff from Scratch

## The Complete, Battle-Tested Guide

**What this is:** A step-by-step guide for setting up an always-on AI agent on a Mac, connected to Slack, Telegram, and/or iMessage. It uses Claude Code as the AI brain, running on a flat $200/month Claude Max subscription.

**What makes this different from the original guide:** This version includes every bug, gotcha, and workaround discovered during an actual setup session. Warnings are flagged before you hit them, not after. It also covers **multi-user setup** (e.g., two people sharing one Mac, each with their own agent persona) and **multi-account email** access.

**Time estimate:** 4-6 hours if you follow this guide. The original guide says "a full day" — this version should be faster because you won't get stuck on the same issues we did.

-----

## Table of Contents

1. [What You're Building (and Why)](#1-what-youre-building-and-why)
1. [What You'll Need](#2-what-youll-need)
1. [Phase 1: Set Up the Mac](#phase-1-set-up-the-mac)
1. [Phase 2: Install Core Software](#phase-2-install-core-software)
1. [Phase 3: Create the Agent's Online Accounts](#phase-3-create-the-agents-online-accounts)
1. [Phase 4: Set Up Claude Code](#phase-4-set-up-claude-code)
1. **Connect a messaging channel (pick one or more):**
- [Phase 5A: Slack](#phase-5a-build-the-slack-bot)
- [Phase 5B: Telegram](#phase-5b-set-up-telegram-optional)
- [Phase 5C: iMessage](#phase-5c-set-up-imessage-optional)
1. [Phase 6: Connect Slack to the Mac via Cloudflare Tunnel](#phase-6-connect-slack-to-the-mac-via-cloudflare-tunnel)
1. [Phase 7: Make It Always On (launchd)](#phase-7-make-it-always-on-launchd)
1. [Phase 8: Give It an Identity](#phase-8-give-it-an-identity)
1. [Phase 9: Connect Google Services](#phase-9-connect-google-services)
1. [Phase 10: Set Up the Email Watcher](#phase-10-set-up-the-email-watcher)
1. [Phase 11: Set Up the File Browser](#phase-11-set-up-the-file-browser)
1. [Phase 12: Set Up Scheduled Jobs](#phase-12-set-up-scheduled-jobs)
1. [Phase 13: Optional Extras](#phase-13-optional-extras)
1. [Phase 14: Multi-User Setup (Second Agent)](#phase-14-multi-user-setup-second-agent)
1. [Maintenance & Troubleshooting](#maintenance--troubleshooting)
1. [Architecture Overview](#architecture-overview)

> **Which messaging channel should I pick?** All three work, and you can set up more than one — they all share the same brain and memory. **Slack** is best for teams (channels, threads, file sharing) but requires the most setup (Cloudflare Tunnel). **Telegram** is great for personal use or small teams, and setup is simpler (no tunnel needed). **iMessage** is the most convenient if it's just you — text your Mac from your phone and get a response. No accounts, no apps, no tunnel.

-----

## 1. What You're Building (and Why)

You're setting up an AI agent that:

- **Lives on a dedicated Mac** (Mac mini or MacBook Air) that's always powered on and connected to the internet
- **Talks to your team via Slack** — they DM it or @mention it in channels
- **Can also respond via Telegram** — an official Claude Code plugin bridges messages to/from Telegram
- **Can respond via iMessage** — text your Mac from your phone and get AI responses back. Reads chat.db directly, sends via AppleScript. No server, no tunnel needed.
- **Has its own email address** — it can receive, read, and reply to emails
- **Has access to Google Workspace** — Gmail, Calendar, Drive, Sheets, Docs
- **Runs on a schedule** — it can do things automatically (daily reports, pipeline refreshes, etc.)
- **Has a web-accessible file browser** — your team can see its files via a browser
- **Is reachable via a public URL** — Cloudflare Tunnel provides a secure connection without opening firewall ports
- **Supports multiple users** — each person can have their own agent persona, name, memory, and channels, all running on the same Mac

**The brain** behind all of this is **Claude Code** (Anthropic's AI coding agent), running via a **Claude Max subscription** ($200/month flat rate).

**Why Claude Code and not the API?** Claude Code is Anthropic's own CLI tool. Running it via a Slack bot on a Max subscription gives you flat $200/month for Claude Opus instead of unpredictable API bills that could run into thousands.

**Why a dedicated Mac?** The bot needs to be always on, always listening. Your personal laptop sleeps, travels, and gets used for other things. A dedicated Mac (even a cheap MacBook Air) sits plugged in, lid closed, running 24/7. It's like giving your AI agent its own desk.

-----

## 2. What You'll Need

### Hardware

- **Mac mini or MacBook Air** (Apple Silicon — M1, M2, M3, or M4). Mac mini is ideal (designed to run headless). MacBook Air works too — keep it plugged in with the lid closed.
- **Monitor, keyboard, mouse** (for initial setup only — you can disconnect these later. MacBook Air has these built in.)
- **Ethernet cable or stable WiFi** (ethernet via USB-C adapter is more reliable for an always-on machine)
- **Power cable** — keep it plugged in 24/7

### Accounts You'll Create

|Account               |What it's for                                  |Cost                                |
|----------------------|-----------------------------------------------|------------------------------------|
|Anthropic (Claude Max)|The AI brain                                   |$200/month                          |
|Slack workspace       |Where the bot talks to your team               |Free or existing                    |
|GitHub                |For the agent to store and manage code         |Free                                |
|Google Workspace      |Email address for the agent                    |Part of existing plan, or free Gmail|
|Cloudflare            |Secure tunnel from the internet to your Mac    |Free                                |
|Google Cloud          |Gmail push notifications + OAuth credentials   |Free tier                           |
|Tailscale             |Remote access from outside your home (optional)|Free                                |
|Gemini API            |Image generation (optional)                    |Free tier                           |

### Domain

You need a domain name you control (e.g., `yourcompany.com` or something new like `mybot.com`). You'll point a subdomain like `bot.yourcompany.com` through Cloudflare.

> **Tip:** You can buy a domain directly through Cloudflare. This is actually easier than buying elsewhere because you skip the step of changing nameservers — Cloudflare is already your registrar and DNS provider in one.

-----

## Phase 1: Set Up the Mac

### 1.1: Factory Reset (if previously used)

1. Back up anything worth keeping
1. Go to **System Settings → General → Transfer or Reset → Erase All Content and Settings**
1. Takes ~15-20 minutes. Keep it plugged in and on WiFi.

### 1.2: Initial macOS Setup

1. Plug in the Mac (power, and for Mac mini: monitor, keyboard, mouse, ethernet)
1. Turn it on
1. Walk through the setup wizard:
- Choose your language and region
- Connect to WiFi (if not using ethernet)
- **Create a user account** — use the agent's name (e.g., username: `clara`, full name: `Clara`). Pick a short username — you'll type it every time you SSH in.
- **Set a password.** macOS requires one for admin tasks. Pick something simple but memorable — you'll need it during setup. You can't skip this.
- **Sign in with your Apple ID** (needed for App Store / Tailscale)
- **FileVault (disk encryption): Skip it / turn it OFF.** If the Mac restarts after a power loss, FileVault shows a lock screen. Since it might restart unattended, skip encryption so it boots straight to the desktop.
- Skip all "share analytics" options

### 1.3: System Settings Checklist

Open **System Settings** and configure:

**Energy (or Battery):**

- Turn ON "Prevent automatic sleeping when the display is off"
- Turn ON "Start up automatically after a power failure" (Mac mini only)
- Turn ON "Wake for network access"
- Turn ON "Optimized Battery Charging"
- MacBook Air only: Under Battery → Options, set both "Turn display off" settings to **Never**

**Lock Screen:**

- "Start Screen Saver when inactive" → **Never**
- "Turn display off when inactive" → **Never**
- "Require password after screen saver begins or display is turned off" → **Never** (critical — otherwise the bot gets locked out)

**Users & Groups:**

- Set **Automatic login** to the user account you created

**General → Sharing:**

- Turn ON **Remote Login (SSH)**
- Turn ON **Screen Sharing**
- Note the command shown (something like `ssh clara@192.168.1.xxx`)

### 1.4: Keep It Awake with the Lid Closed (MacBook Air)

By default, closing the lid puts it to sleep. Override this. Open Terminal and run:

```bash
# Prevent sleep entirely — even with the lid closed
sudo pmset -a disablesleep 1

# Verify it worked (look for "disablesleep 1" in the output)
pmset -g
```

**What's `sudo`?** It means "run as admin." It will ask for your password. Type it — nothing will appear on screen as you type (no dots, no stars, nothing). That's normal. Press Enter.

> **Battery note:** Running plugged in 24/7 with the lid closed can cause battery swelling over months/years. "Optimized Battery Charging" helps. Worst case, battery replacement is ~$100-150. If the bottom bulges, get it replaced.

### 1.5: Grant Full Disk Access to Terminal

macOS blocks background processes from accessing files unless you allow it. Without this, you'll get constant popups that nobody can click (because the lid is closed).

1. Go to **System Settings → Privacy & Security → Full Disk Access**
1. Click the **+** button
1. Navigate to **Applications → Utilities → Terminal.app** and add it
1. Toggle it **ON**

### 1.6: Set It Up Physically

1. Plug into power (always — use a surge protector)
1. Connect to WiFi or ethernet
1. Mac mini: disconnect monitor, keyboard, mouse. Tuck it away.
1. MacBook Air: close the lid. Place on a hard surface for ventilation.
1. Note the local IP or hostname (from System Settings → Sharing)

**From your laptop** (same WiFi):

- **Screen Sharing:** Finder → Go → Connect to Server → `vnc://your-mac.local`
- **SSH:** Terminal → `ssh clara@your-mac.local`

### 1.7: Install Tailscale (Optional)

Only needed if you want to access the Mac from outside your home WiFi (coffee shop, traveling, etc.). The Slack bot itself works from anywhere via Cloudflare Tunnel regardless.

1. Open the **Mac App Store**, install **Tailscale**
1. Open Tailscale, sign in
1. Enable "Start Tailscale when I log in"
1. Note the Tailscale IP (like `100.x.x.x`)
1. Install Tailscale on your laptop/phone too — same account

```bash
# Find the Tailscale IP later
tailscale ip -4
```

-----

## Phase 2: Install Core Software

From here on, you can do everything via SSH from your laptop.

```bash
ssh clara@your-mac.local   # same WiFi
# or: ssh clara@100.x.x.x  # anywhere, if Tailscale installed
```

### 2.1: Install Homebrew

**What this is:** Homebrew is like an app store for command-line tools. Instead of downloading installers from websites, you just type `brew install toolname`.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

It will ask for your password. After it finishes, **it shows you two commands to run.** They look like:

```bash
echo >> ~/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

> **BUG WARNING:** If you close the Terminal window before running those two commands, just open a new Terminal and run them. They're always the same on Apple Silicon Macs. Verify with `brew --version`.

### 2.2: Install Required Tools

```bash
brew install git node python3 cloudflared gh tmux mosh
```

**What these are:**

- **git** — version control (managing code)
- **node** — JavaScript runtime (needed for Claude Code)
- **python3** — Python runtime (the Slack bot runs on Python)
- **cloudflared** — Cloudflare's tunnel client (connects your Mac to the internet securely)
- **gh** — GitHub's command-line tool
- **tmux** — lets you run terminal sessions that persist after you disconnect
- **mosh** — mobile-friendly SSH that reconnects if your WiFi drops

### 2.3: Install Claude Code

**What this is:** Claude Code is Anthropic's command-line AI agent. It's the "brain" that powers your bot — when someone sends a Slack message, the bot spawns a Claude Code session to think and respond.

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

> **Note:** You'll see messages like "2 packages are looking for funding" — this is normal. It just means some open-source maintainers accept donations. Ignore it.

> **Note:** You may also see "This is not an officially supported Google product" after installing gws (below). That's a standard legal disclaimer on Google's open-source projects. It works fine.

### 2.4: Enable Passwordless Sudo

**What this does:** Claude Code needs to run admin commands (installing packages, managing services) but can't type a password interactively. This lets it run admin commands without being prompted.

```bash
# Replace YOUR_USERNAME with the actual username (e.g., clara)
echo "YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/YOUR_USERNAME
```

It will ask for your password one last time. After this, you won't be prompted again.

> **Is this safe?** On your personal laptop, you wouldn't do this. But this is a dedicated AI server that only you access. The bot needs full system access to be useful.

### 2.5: Install Google Workspace CLI

**What this is:** A command-line tool that lets the bot read and write to Gmail, Calendar, Drive, Docs, and Sheets via the terminal. This is **separate from** Claude's built-in MCP integrations (covered in Phase 9) — `gws` gives the agent direct shell access to Google services.

```bash
npm install -g @googleworkspace/cli
```

Verify:

```bash
gws --version
```

### 2.6: Install Browser Automation Tools (Optional)

**What this does:** Lets your AI agent control a real Chrome browser — clicking buttons, filling forms, logging into websites, reading page content. Skip this for now if you want to get the basics working first.

```bash
npm install -g agent-browser dev-browser
dev-browser install
```

### 2.7: Install Bun (Needed for Telegram)

**What this is:** Bun is a fast JavaScript runtime. The Telegram channel plugin requires it.

```bash
brew install oven-sh/bun/bun
```

-----

## Phase 3: Create the Agent's Online Accounts

### 3.1: Create a GitHub Account

1. Go to **github.com** → Sign up
1. Create an account for the agent (e.g., username: `hitalbot`)
1. Verify the email

**Set up an SSH key** so the Mac can push code to GitHub without a password:

```bash
# Generate the key — replace with the agent's email
ssh-keygen -t ed25519 -C "youragent@gmail.com"
# Press Enter three times (default location, no passphrase, confirm)

# Show the public key
cat ~/.ssh/id_ed25519.pub
```

**What's an SSH key?** It's a pair of cryptographic files. The private key stays on your Mac (never share it). The public key goes on GitHub. They work together so your Mac can prove its identity without a password.

Copy the output (starts with `ssh-ed25519`, all one line) and:

1. Go to **github.com → Settings → SSH and GPG keys → New SSH key**
1. Title: something like "Mac mini" (just a label)
1. Key: paste the public key
1. Click **Add SSH key**

> **BUG WARNING:** If GitHub says "Key is invalid," make sure you copied the entire line with no extra line breaks. It must be one continuous line starting with `ssh-ed25519` and ending with the email.

> **Backup tip:** Save both key files (`~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`) in a password manager. The private key is like a password — never put it in email, Slack, or Google Docs. If you lose it, just generate a new one.

**Set up GitHub CLI:**

```bash
gh auth login
# Choose: GitHub.com → SSH → Yes (use existing key) → Login with a web browser
# It will show a one-time code — enter it in the browser that opens
```

### 3.2: Create Email Accounts for the Agent

Your agent needs at least one email address. If you're setting up **multiple users** (e.g., two people each with their own agent persona), each user's agent should have its own email.

**Option A: Google Workspace (recommended if you have it)**

1. Go to **admin.google.com**
1. Create a new user for each agent (e.g., `talbot@yourcompany.com`, `atlas@stellation.vc`)
1. These integrate cleanly with Calendar, Drive, etc.

**Option B: Regular Gmail**

1. Go to **gmail.com** and create a new account for each agent
1. Free, works fine, but won't have the Google Workspace admin features

**Option C: iCloud email**

1. Create via iCloud settings
1. Works for receiving/sending but lacks native integration with Google's MCP tools
1. Only recommended if you want to stay entirely within Apple's ecosystem

> **Multi-user note:** If Person A and Person B each want their own agent, create separate emails now. Later (Phase 14) you'll configure each agent to access its own email — and optionally read the other person's inbox too.

### 3.3: Create a Cloudflare Account and Get a Domain

1. Go to **cloudflare.com** and sign up (use your personal email, not the agent's)
1. You need a domain. Two options:
- **Buy directly through Cloudflare** (recommended — skip the nameserver step entirely). Go to **Domains → Register a new domain**.
- **Use an existing domain** — add it as a site in Cloudflare and update nameservers at your registrar to point to Cloudflare's.

> **Tip:** Use your personal/company account for Cloudflare. It's infrastructure you manage — the agent doesn't need its own Cloudflare account.

### 3.4: Get a Claude Max Subscription

1. Go to **claude.ai** and subscribe to **Claude Max** ($200/month)
1. On the dedicated Mac, run:

```bash
claude login
```

Follow the prompts. It prints a URL — open it in any browser to authenticate.

> **Multi-user note:** One Claude Max subscription is enough for multiple agents on the same machine. They share the rate limit. On Max $200/month, two concurrent Opus sessions work fine. If you run into rate limits, consider upgrading to Max 5x ($100/month more).

### 3.5: Get a Gemini API Key (Optional)

For image generation:

1. Go to **aistudio.google.com/apikey**
1. Click **Create API Key**
1. Save it — you'll add it to the `.env` file later

-----

## Phase 4: Set Up Claude Code

### 4.1: Create the Directory Structure

**What these are:** Organized folders for the agent's work, logs, notes, and knowledge about teammates.

```bash
mkdir -p ~/Projects
mkdir -p ~/work
mkdir -p ~/temp
mkdir -p ~/diary
mkdir -p ~/bookmarks
mkdir -p ~/discoveries
mkdir -p ~/teammates
mkdir -p ~/logs
```

### 4.2: Clone the Slack Bot Code

**What this does:** Downloads the open-source bot code from GitHub. This is a one-way download — the repo owner has no access to your machine.

```bash
cd ~/Projects
git clone https://github.com/nityeshaga/claude-home-base.git slack-bot
cd slack-bot
```

> **Want full independence?** Fork the repo to your own GitHub account first, then clone from your fork. That way changes to the original repo won't affect you.

### 4.3: Install Python Dependencies

**What this does:** Creates an isolated Python environment and installs the libraries the bot needs (Slack SDK, Flask web server, etc.).

```bash
cd ~/Projects/slack-bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

> **What's a venv?** A virtual environment keeps your Python packages separate from the system Python. Prevents conflicts.

### 4.4: Configure Claude Code Settings

**What this does:** Tells Claude Code two things: (1) use the smartest model (Opus) with a 1 million token context window, and (2) skip confirmation prompts when running unattended (like scheduled jobs at 3am).

You can do this in Claude Code itself (since `nano` is interactive and Claude Code can't run it). Open Claude Code and tell it:

> Write a file at ~/.claude/settings.json with this content: {"model": "opus[1m]", "skipDangerousModePermissionPrompt": true}

Or in a regular Terminal:

```bash
mkdir -p ~/.claude
cat > ~/.claude/settings.json << 'EOF'
{
  "model": "opus[1m]",
  "skipDangerousModePermissionPrompt": true
}
EOF
```

-----

## Phase 5A: Build the Slack Bot

**What this phase does:** Creates a Slack app, gives it permissions to read and send messages, and connects it to the bot running on your Mac.

### 5A.1: Create a Slack App

You need admin access to your Slack workspace.

1. Go to **api.slack.com/apps** → **Create New App**
1. Choose **"From an app manifest"**

### 5A.2: Paste the App Manifest

Replace `Your Bot Name` with your bot's name and `bot.yourdomain.com` with your actual domain:

```json
{
  "display_information": {
    "name": "Your Bot Name",
    "description": "AI cofounder for your team",
    "background_color": "#0f172a"
  },
  "features": {
    "bot_user": {
      "display_name": "Your Bot Name",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read", "channels:history", "channels:join",
        "channels:read", "chat:write", "chat:write.customize",
        "commands", "files:read", "files:write",
        "groups:history", "groups:read", "im:history",
        "im:read", "im:write", "links:read",
        "mpim:history", "mpim:read", "mpim:write",
        "pins:read", "pins:write", "reactions:read",
        "reactions:write", "team:read", "users:read",
        "users:read.email", "users.profile:read",
        "bookmarks:read", "metadata.message:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://bot.yourdomain.com/slack/events",
      "bot_events": [
        "app_mention", "message.im", "message.channels",
        "message.groups", "message.mpim",
        "member_joined_channel", "reaction_added",
        "file_shared"
      ]
    },
    "org_deploy_enabled": false,
    "socket_mode_enabled": false,
    "token_rotation_enabled": false
  }
}
```

> **IMPORTANT:** Replace BOTH instances of "Your Bot Name" and the domain URL before pasting. Don't leave the placeholders.

> **URL verification will fail** at this point. That's expected — you haven't set up the tunnel yet (Phase 6). Just save the manifest and move on. You'll verify the URL after Phase 6.

### 5A.3: Get Your Tokens

After creating the app:

1. **Bot Token:** Go to **OAuth & Permissions** → **Install to Workspace** → copy the `xoxb-...` token
1. **Signing Secret:** Go to **Basic Information** → App Credentials → copy the Signing Secret

### 5A.4: Find Your Slack User ID

In Slack, click on your profile picture → three dots (⋮) → **"Copy member ID"**. Do this for every team member who should talk to the bot.

### 5A.5: Configure the Bot's Environment

The `.env` file stores all your secret tokens. The bot reads it on startup.

In Claude Code on the Mac, or in a regular Terminal:

```bash
cd ~/Projects/slack-bot
cp .env.example .env
```

Then edit `.env` with your values:

```
SLACK_BOT_TOKEN=xoxb-your-actual-bot-token
SLACK_SIGNING_SECRET=your-actual-signing-secret
AUTHORIZED_USERS=U0XXXXXXXXXX,U0YYYYYYYYYY
PROJECT_DIR=/Users/clara
CLAUDE_TIMEOUT=1800
PORT=3000
GEMINI_API_KEY=your-gemini-key-here
```

**What these mean:**

- `SLACK_BOT_TOKEN` — the token that lets the bot post messages to Slack
- `SLACK_SIGNING_SECRET` — proves incoming requests are really from Slack (not an impersonator)
- `AUTHORIZED_USERS` — comma-separated Slack member IDs of people allowed to talk to the bot
- `PROJECT_DIR` — the bot's home directory on the Mac
- `CLAUDE_TIMEOUT` — max seconds (1800 = 30 minutes) the bot waits for Claude to respond before giving up
- `PORT` — the port on your Mac where the bot listens (3000 is standard)
- `GEMINI_API_KEY` — for image generation (optional)

> **Tip:** If you're in Claude Code and can't run `nano` (it's interactive), just tell Claude Code: "Update ~/Projects/slack-bot/.env with these values: …" and paste your tokens directly.

> **Security note:** It's safe to paste API keys into Claude Code — it's running locally on your Mac and just writes them to a file. They don't leave your machine.

### 5A.6: Test the Bot Locally

```bash
cd ~/Projects/slack-bot
source venv/bin/activate
python bot.py
```

You should see: `Bot starting on port 3000...`

The bot is now listening, but only on your local network. Slack can't reach it yet — that's Phase 6.

-----

## Phase 5B: Set Up Telegram (Optional)

Anthropic officially supports Telegram as a Claude Code Channel — a built-in plugin that bridges Claude Code to Telegram. No custom bot code needed.

### 5B.1: Create a Telegram Bot

1. Open Telegram
1. Search for **@BotFather** (blue checkmark)
1. Send `/start`, then `/newbot`
1. Pick a display name (e.g., "My AI Bot")
1. Pick a username — must end in `bot` (e.g., `myai_bot`)
1. BotFather gives you a bot token. **Save this.**

### 5B.2: Install the Plugin

Open Claude Code on the Mac:

```
/plugin install telegram@claude-plugins-official
/reload-plugins
```

### 5B.3: Configure and Launch

```bash
claude --channels plugin:telegram@claude-plugins-official
```

When prompted, enter your bot token. It will show a pairing code — send this code to your bot in Telegram to link your account.

> **If the plugin shows as "failed"** in `/mcp`, try running `claude --debug --channels plugin:telegram@claude-plugins-official` to see the actual error. Common fix: make sure Bun is installed (`brew install oven-sh/bun/bun`).

### 5B.4: Make It Always On

Create a launchd service (replace `YOUR_USERNAME` with your actual username, e.g., `clara`):

```bash
nano ~/Library/LaunchAgents/com.claude.telegram-channel.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.telegram-channel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/claude</string>
        <string>--channels</string>
        <string>plugin:telegram@claude-plugins-official</string>
        <string>--dangerously-skip-permissions</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/YOUR_USERNAME</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/logs/telegram-channel.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/logs/telegram-channel.err</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.claude.telegram-channel.plist
```

No Cloudflare Tunnel needed for Telegram — the plugin polls Telegram's servers directly.

-----

## Phase 5C: Set Up iMessage (Optional)

**What this does:** Lets you text your Mac from your iPhone (or any Apple device) via iMessage and get AI responses back. The plugin reads your Mac's `chat.db` (the iMessage database) directly and sends replies via AppleScript. No external server, no tunnel, no background process beyond Claude Code itself.

**macOS only.** This only works on a Mac because it reads the native Messages database.

### 5C.1: Grant Full Disk Access

The iMessage plugin needs to read `~/Library/Messages/chat.db`, which macOS protects. You should have already done this in Phase 1.5, but verify:

1. Go to **System Settings → Privacy & Security → Full Disk Access**
1. Make sure **Terminal.app** (or whatever terminal you use) is listed and toggled **ON**

> **If you skip this**, the plugin will exit immediately with `authorization denied`. macOS may also pop up a prompt the first time asking if your terminal can access Messages — click **Allow**.

### 5C.2: Install the Plugin

Open Claude Code on the Mac:

```
/plugin install imessage@claude-plugins-official
```

No environment variables or API keys needed. It reads the local Messages database directly.

### 5C.3: Launch with the Channel Flag

Exit your current Claude Code session and start a new one with the iMessage channel:

```
claude --channels plugin:imessage@claude-plugins-official
```

Verify it's working by checking that `/imessage:configure` tab-completes.

### 5C.4: Text Yourself to Test

iMessage yourself from your phone. Self-chat bypasses access control, so it works immediately.

> **First outbound reply:** macOS will show an Automation permission prompt ("Terminal wants to control Messages"). Click **OK**. This only happens once.

### 5C.5: Allow Other People

By default, nobody else's texts reach the agent — only your self-chat works. To allow other people:

```
/imessage:access allow +15551234567
```

Handles can be phone numbers (`+15551234567`) or Apple ID emails (`them@icloud.com`). If you're not sure what handle format to use, ask Claude to review your setup.

### 5C.6: How It Works

|                    |                                                                                                                              |
|--------------------|------------------------------------------------------------------------------------------------------------------------------|
|**Inbound**         |Polls `chat.db` once per second for new messages. Old messages aren't replayed on restart — it starts from the latest message.|
|**Outbound**        |Uses AppleScript (`tell application "Messages" to send...`) to send replies through the Messages app.                         |
|**History & search**|Direct SQLite queries against `chat.db`. Full native message history, not just messages since the server started.             |
|**Attachments**     |Inbound images are surfaced as local file paths. Outbound attachments send as separate messages after the text.               |

### 5C.7: Environment Variables (Optional)

|Variable                   |Default                      |What it does                                                           |
|---------------------------|-----------------------------|-----------------------------------------------------------------------|
|`IMESSAGE_APPEND_SIGNATURE`|`true`                       |Adds "Sent by Claude" to outbound messages. Set to `false` to disable. |
|`IMESSAGE_ACCESS_MODE`     |—                            |Set to `static` to disable runtime pairing and only read `access.json`.|
|`IMESSAGE_STATE_DIR`       |`~/.claude/channels/imessage`|Where access control state is stored.                                  |

### 5C.8: Make It Always On

Create a launchd service (replace `YOUR_USERNAME`):

```bash
nano ~/Library/LaunchAgents/com.claude.imessage-channel.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.imessage-channel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/claude</string>
        <string>--channels</string>
        <string>plugin:imessage@claude-plugins-official</string>
        <string>--dangerously-skip-permissions</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/YOUR_USERNAME</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/logs/imessage-channel.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/logs/imessage-channel.err</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.claude.imessage-channel.plist
```

No Cloudflare Tunnel needed. Like Telegram, iMessage works locally — it reads the database on your Mac and sends via AppleScript.

### 5C.9: Limitations

AppleScript can send messages but cannot tapback (react), edit, or thread. If you need those features, look into [BlueBubbles](https://bluebubbles.app) (requires disabling SIP — not recommended for this setup).

-----

## Phase 6: Connect Slack to the Mac via Cloudflare Tunnel

**What this does in plain language:** Right now the bot is only reachable on your home WiFi. Slack's servers are on the internet and can't reach your Mac. Cloudflare Tunnel creates a secure bridge: when Slack sends a message to `bot.yourdomain.com`, Cloudflare routes it through an encrypted tunnel directly to port 3000 on your Mac. No open ports, no static IP, no exposing your home network. Free.

### 6.1: Authenticate Cloudflare

> **IMPORTANT: Do this in a regular Terminal, not inside Claude Code.** It opens a browser window you need to click.

```bash
cloudflared tunnel login
```

A browser opens. Log into Cloudflare and authorize. You'll see a "Success" message.

### 6.2: Create a Tunnel

```bash
cloudflared tunnel create my-bot
```

This outputs a **tunnel ID** — a long string like `e87ee4c9-111d-4159-9625-d083981285c9`. **Copy it.** You'll also see a path to a credentials JSON file. **Note both.**

### 6.3: Create the Tunnel Config

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Paste this (replace with YOUR tunnel ID, YOUR username, and YOUR domain):

```yaml
tunnel: YOUR-TUNNEL-ID
credentials-file: /Users/YOUR_USERNAME/.cloudflared/YOUR-TUNNEL-ID.json

ingress:
- hostname: bot.yourdomain.com
  service: http://localhost:3000
- service: http_status:404
```

### 6.4: Create the DNS Record

```bash
cloudflared tunnel route dns my-bot bot.yourdomain.com
```

This tells Cloudflare's global DNS that `bot.yourdomain.com` should point to your tunnel.

### 6.5: Test the Tunnel

```bash
cloudflared tunnel run my-bot
```

Leave it running. In another Terminal, test:

```bash
curl https://bot.yourdomain.com/slack/events
```

You should see: `405 Method Not Allowed`. **This is correct!** It means the request made it through the tunnel to your bot. The 405 happens because `curl` sends a GET request but the bot only accepts POST (which is what Slack sends).

> **If you see `error code: 1033`**, the tunnel isn't routing correctly. See the troubleshooting section below.

### 6.6: Install as a Permanent System Service

Stop the test tunnel (Ctrl+C), then:

```bash
sudo cloudflared service install
```

> **BUG WARNING: This is where we hit a major issue.** The `cloudflared service install` command creates a system plist at `/Library/LaunchDaemons/com.cloudflare.cloudflared.plist`, but it does NOT include the `tunnel run` arguments. It just runs `cloudflared` with no arguments, which does nothing useful. It also looks for config in `/etc/cloudflared/` not `~/.cloudflared/`.

**The fix — do BOTH of these:**

**Step 1: Copy your config and credentials to the system location:**

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo cp ~/.cloudflared/YOUR-TUNNEL-ID.json /etc/cloudflared/
```

Then update the credentials path in the system config:

```bash
sudo nano /etc/cloudflared/config.yml
```

Change the `credentials-file` line to point to the system location:

```yaml
credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID.json
```

**Step 2: Fix the system plist to include `tunnel run`:**

```bash
sudo nano /Library/LaunchDaemons/com.cloudflare.cloudflared.plist
```

Find the `ProgramArguments` section. It will look like:

```xml
<key>ProgramArguments</key>
<array>
<string>/opt/homebrew/bin/cloudflared</string>
</array>
```

Replace it with:

```xml
<key>ProgramArguments</key>
<array>
<string>/opt/homebrew/bin/cloudflared</string>
<string>tunnel</string>
<string>run</string>
</array>
```

Save, then restart:

```bash
sudo launchctl unload /Library/LaunchDaemons/com.cloudflare.cloudflared.plist
sudo launchctl load /Library/LaunchDaemons/com.cloudflare.cloudflared.plist
sleep 5
curl https://bot.yourdomain.com/slack/events
```

You should see the `405 Method Not Allowed` again. If so, the tunnel is permanent.

> **How to check if the tunnel is broken:** Run `sudo cat /Library/Logs/com.cloudflare.cloudflared.err.log | tail -20`. If you see repeated lines saying `Use cloudflared tunnel run to start tunnel`, it means the plist is missing the `tunnel run` arguments. Apply the fix above.

### 6.7: Verify the Slack URL

1. Go to **api.slack.com/apps** → your app → **Event Subscriptions**
1. The URL should be `https://bot.yourdomain.com/slack/events`
1. Click **Retry** (or re-enter the URL)
1. It should say **Verified ✓**
1. **Click Save Changes** at the bottom

> **IMPORTANT:** If you changed any scopes since first installing the app, go to **Install App** → **Reinstall to Workspace** → **Allow**. Slack won't send events until you do this.

### 6.8: Test It

Go to Slack, find your bot under **Apps**, and send it a DM. It should respond!

-----

## Phase 7: Make It Always On (launchd)

**What this does:** Right now the bot runs because you started it manually. If the Mac reboots or the bot crashes, it's dead. launchd is macOS's built-in system for keeping programs alive forever — it starts the bot on boot and restarts it if it ever crashes.

### 7.1: Understand launchd

- **Plist files** go in `~/Library/LaunchAgents/` (per-user services)
- **`RunAtLoad: true`** — starts when you log in
- **`KeepAlive: true`** — auto-restarts if it crashes
- **`StartCalendarInterval`** — runs at specific times (like a scheduler)
- Times are in your **local timezone** (not UTC)

### 7.2: Create the Slack Bot Service

Replace `YOUR_USERNAME` with your actual username (e.g., `clara`) everywhere:

```bash
nano ~/Library/LaunchAgents/com.claude.homebase.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.homebase</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot-stderr.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

**Why EnvironmentVariables?** launchd services don't inherit your shell's PATH. Without this, the bot can't find `claude` or brew-installed tools.

> **Tip:** You can also have Claude Code create this file for you. Tell it: "Create the file ~/Library/LaunchAgents/com.claude.homebase.plist with this content: …" and paste the XML.

### 7.3: Load, Verify, and Test

```bash
# Start the service
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist

# Verify it's running
launchctl list | grep homebase

# Test crash recovery — kill the bot, watch launchd bring it back
kill $(pgrep -f bot.py)
sleep 3 && pgrep -f bot.py

# If a new PID appears, it's working!

# Test reboot survival
sudo reboot
# Wait 2-3 min, then DM the bot in Slack
```

-----

## Phase 8: Give It an Identity

This is what makes your agent feel like a team member, not just a tool.

### 8.1: Create the CLAUDE.md File

**What this is:** The operations manual. Claude Code reads it at the start of every conversation. It tells the agent who it is, what tools it has, and how to behave.

Create `~/CLAUDE.md` and include:

- The agent's name and role
- Your company context
- Team roster (names, Slack user IDs, roles)
- How to communicate (Slack, email)
- Privacy rules (who can see what)
- File paths and workspace structure
- Instructions for every tool (gws, Cloudflare, Tailscale, etc.)
- Proactive messaging instructions

Use the template at [github.com/nityeshaga/claude-home-base](https://github.com/nityeshaga/claude-home-base).

> **Shortcut:** Open Claude Code and tell it: "Ask me questions about my team, my company, and how I want you to behave — then write your own CLAUDE.md, identity.md, and origin story."

### 8.2: Create the Identity File

`~/identity.md` — personality and principles. Keep it short:

- What role does it play?
- How does it communicate?
- What does it care about?

### 8.3: Create the Teammates Directory

```bash
mkdir -p ~/teammates
```

Create `~/teammates/teammates.md` with authority levels and privacy rules. Create individual files for each person (e.g., `~/teammates/natalia.md`) with their name, Slack ID, role, and communication preferences.

### 8.4: Set Up Memory

```bash
mkdir -p ~/.claude/projects/-Users-YOUR_USERNAME/memory
```

Create a `MEMORY.md` file there. Starts empty and builds over time as the agent learns.

-----

## Phase 9: Connect Google Services

There are **two separate systems** for connecting Google services. You should set up both:

1. **Claude's built-in MCP integrations** — Gmail, Calendar, Notion, Slack, etc. connected through claude.ai. These give your agent structured tool access (search emails, create events, etc.) directly within conversations.
2. **Google Workspace CLI (`gws`)** — a command-line tool the agent can call via bash. Covers Drive, Docs, Sheets, and can also do Gmail/Calendar. Useful for batch operations and scripts.

### 9.1: Connect Built-in MCP Integrations (Recommended — Do This First)

These are the easiest to set up and the most useful for day-to-day agent work.

1. Open Claude Code on the Mac
2. Go to **claude.ai/settings** in a browser (logged into the same account)
3. Navigate to **Integrations** or **Connected Apps**
4. Connect:
   - **Gmail** — authorize with the agent's Google account
   - **Google Calendar** — authorize (same account)
   - **Slack** — authorize with your workspace
   - **Notion** — authorize if you use it

Once connected, Claude Code automatically gets MCP tools like `gmail_search_messages`, `gcal_list_events`, `slack_send_message`, etc. These work inside any Claude Code conversation — no `gws` needed.

> **IMPORTANT — Single account limitation:** The built-in Gmail MCP connects to **one Gmail account**. If you need to read multiple inboxes (your work email, personal email, someone else's email), see Phase 9.5 below for the multi-account solution.

> **Calendar is already multi-account:** Unlike Gmail, the Calendar integration can see ANY calendar that's been shared with or added to the connected account's calendar list. So if Peter shares his calendar with natalia@every.to, the agent can see both.

### 9.2: Create a Google Cloud Project (Needed for `gws` and email push notifications)

1. Go to **console.cloud.google.com**
1. Click **New Project** — name it "AI Agent"

### 9.3: Enable APIs

In **APIs & Services → Library**, enable:

- Gmail API
- Google Calendar API
- Google Drive API
- Google Docs API
- Google Sheets API
- Google Slides API (if needed)
- Cloud Pub/Sub API (for email notifications)

### 9.4: Create OAuth Credentials

1. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
1. Configure the consent screen first:
   - Internal (Google Workspace) or External (personal Gmail)
   - Add scopes for Gmail, Calendar, Drive
1. Application type: **Desktop app**
1. Click **Create**, then **Download JSON**

> **Tip:** "Internal" means only your org's users. Skips verification hassle. For personal Gmail, choose "External" and add your email as a test user.

### 9.5: Authenticate `gws`

**Step 1:** Place the downloaded OAuth credentials file:

```bash
mkdir -p ~/.config/gws
cp ~/Downloads/client_secret_*.json ~/.config/gws/client_secret.json
```

> **IMPORTANT:** The file MUST be named `client_secret.json` and placed at `~/.config/gws/client_secret.json`. The `gws` CLI looks for it there. If this file is missing, `gws auth login` will fail silently or give a cryptic error.

**Step 2:** Authenticate (do this in a regular Terminal, not Claude Code — it opens a browser):

```bash
gws auth login --full
```

This opens a browser for OAuth consent. Select the agent's Google account and approve all scopes. The `--full` flag requests all scopes including Pub/Sub (needed for email notifications).

**Step 3:** Verify it worked:

```bash
gws auth status
```

You should see `auth_method: "oauth"` and your email address. Test with:

```bash
gws gmail messages list --max-results 3
gws calendar events list
```

### 9.6: Multi-Account Email Access

The built-in Gmail MCP only connects to one account. If you need to read multiple inboxes (e.g., work email + personal email, or your inbox + your partner's inbox), you have options:

**Option A: Community multi-account MCP server (recommended)**

Install a multi-account Gmail MCP server that lets the agent query any connected inbox by name:

```bash
npm install -g google-workspace-mcp
```

Configure it as a custom MCP server in Claude Code's settings. Each account gets its own OAuth credentials and a label (e.g., "natalia-work", "natalia-personal", "peter-work"). The agent specifies which account to use on each request.

Popular options:
- `google-workspace-mcp` by aaronsb — Gmail + Calendar + Drive, per-account credential isolation
- `gmail-mcp-multi` by dmorrill — Gmail-specific, supports unlimited accounts with aliases

**Option B: Gmail delegation**

Add the agent's email as a delegate on other Gmail accounts. The agent can then access delegated inboxes through the primary account.

**Option C: Forwarding rules**

Set up auto-forwards from secondary inboxes to the agent's primary inbox. Simple but you lose the ability to reply from the original address.

### 9.7: Set Up Pub/Sub for Email Notifications

1. In Google Cloud Console → **Pub/Sub** → Create a topic (e.g., `gmail-notifications`)
1. Create a subscription for that topic
1. Grant `gmail-api-push@system.gserviceaccount.com` the **Pub/Sub Publisher** role
1. The email watcher script handles the rest

-----

## Phase 10: Set Up the Email Watcher

### 10.1: Verify Scripts Exist

```bash
ls ~/Projects/slack-bot/email-watcher.py
ls ~/Projects/slack-bot/email-sweep.py
```

- **email-watcher.py** — watches for new emails in real time
- **email-sweep.py** — runs every 12 hours as a safety net

### 10.2: Test

```bash
cd ~/Projects/slack-bot
source venv/bin/activate
python email-watcher.py
```

Send a test email to the agent's address.

### 10.3: Create launchd Services

Create both plist files (replace `YOUR_USERNAME`):

**Email Watcher** (`~/Library/LaunchAgents/com.claude.email-watcher.plist`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.email-watcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher-stderr.log</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

**Email Sweep** (`~/Library/LaunchAgents/com.claude.email-sweep.plist`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.email-sweep</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep-stderr.log</string>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>43200</integer>
</dict>
</plist>
```

Load both:

```bash
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-sweep.plist
```

-----

## Phase 11: Set Up the File Browser

**What this does:** Gives your team a web-based way to browse files on the Mac.

```bash
mkdir -p ~/work/file-browser
```

Have Claude Code generate it:

```bash
claude -p "Create a Python HTTP server at ~/work/file-browser/server.py that serves files from my home directory. Nice web UI with file browsing, markdown rendering, and dark theme. Accept a port number as a command-line argument."
```

Create a launchd service for it (same pattern as above, using port 8889), then load it.

Accessible at:

- Same WiFi: `http://your-mac.local:8889`
- Anywhere (Tailscale): `http://TAILSCALE_IP:8889`

-----

## Phase 12: Set Up Scheduled Jobs

Each scheduled job is a launchd plist that runs Claude Code with a prompt at a specific time.

**Key points:**

- Always use `--dangerously-skip-permissions` (no human to click "approve")
- The `--dangerously-skip-permissions` flag must come **before** `-p` in the arguments
- Times are **local timezone** (not UTC)
- Weekday numbers: 0 = Sunday, 1 = Monday, …, 6 = Saturday
- The agent can notify via Slack using `bot.py --send`

Example daily task at 10:30 AM (replace `YOUR_USERNAME`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.daily-task</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/claude</string>
        <string>--dangerously-skip-permissions</string>
        <string>-p</string>
        <string>Your prompt here. When done, DM the team a summary.</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>10</integer>
        <key>Minute</key>
        <integer>30</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/logs/daily-task.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/logs/daily-task.err</string>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/Users/YOUR_USERNAME/.claude/local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/YOUR_USERNAME</string>
    </dict>
</dict>
</plist>
```

-----

## Phase 13: Optional Extras

### 13.1: Disable Automatic macOS Updates

Updates can restart your Mac unexpectedly.

1. **System Settings → General → Software Update → Automatic Updates**
1. Turn OFF "Install macOS updates" and "Install Security Responses"
1. Apply updates manually when convenient

### 13.2: Browser Automation

1. Open Chrome on the Mac and log into relevant accounts
1. The agent uses `dev-browser --connect` to control Chrome

### 13.3: MCP Servers

MCP (Model Context Protocol) servers give the agent additional tool access for richer integration.

-----

## Phase 14: Multi-User Setup (Second Agent)

**What this does:** Sets up a second agent persona on the same Mac for a different person. Each agent has its own name, personality, memory, email, and Telegram bot — but they share the same Claude Max subscription and hardware.

**Example:** Person A has "Talbot" (Chief of Staff). Person B gets "Atlas" (VC Analyst). Same Mac, same account, completely different behavior per person.

### 14.1: How It Works

Each Claude Code session reads its `CLAUDE.md` from its **working directory**. So if Talbot runs from `~/` and Atlas runs from `~/peter/`, they load different instructions and behave differently.

```
~/                          ← Person A's agent (Talbot)
├── CLAUDE.md               ← Chief of Staff instructions
├── identity.md             ← Talbot's personality
├── teammates/
└── .claude/projects/-Users-clara/memory/   ← Talbot's memory

~/peter/                    ← Person B's agent (Atlas)
├── CLAUDE.md               ← VC Analyst instructions
├── identity.md             ← Atlas's personality
├── teammates/
└── .claude/projects/-Users-clara-peter/memory/  ← Atlas's memory
```

Both agents share:
- The same Claude Max subscription (same rate limit)
- The same Google Calendar (both can see shared calendars)
- The same household items (Leo, bills, logistics)

Each agent has:
- Its own name and persona
- Its own CLAUDE.md with different instructions
- Its own memory directory
- Its own Telegram bot
- Access to different email accounts (if using multi-account MCP)

### 14.2: Create the Second Agent's Directory

```bash
mkdir -p ~/peter
mkdir -p ~/peter/teammates
mkdir -p ~/peter/work
mkdir -p ~/peter/logs
```

### 14.3: Create the Second Agent's CLAUDE.md

Create `~/peter/CLAUDE.md` with Person B's instructions. Example framing:

```markdown
# Atlas — VC Analyst for Peter Boyce II

You are Atlas, a VC analyst supporting Peter Boyce II, Managing Partner at Stellation Venture Capital.

## How You Operate
- [Peter's preferred working style]
- [Deal flow tracking, portfolio monitoring, fund ops]
- [Briefing format and priorities]

## Morning Brief
- [Deal-flow-heavy format]
- [Portfolio updates]
- [LP communications due]

## Shared Items
- Household logistics are shared with Talbot (~/CLAUDE.md)
- Google Calendar is shared — flag conflicts with Natalia's schedule
- Leo coverage: only flag when there's a gap
```

### 14.4: Create a Second Telegram Bot

1. Open Telegram → @BotFather → `/newbot`
2. Pick a name and username for Person B's bot
3. Save the token

Configure the plugin for the second agent. The second agent's Telegram channel needs its own state directory:

```bash
mkdir -p ~/.claude/channels/telegram-peter
```

### 14.5: Create the Second Agent's launchd Service

```bash
nano ~/Library/LaunchAgents/com.claude.peter-telegram.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.peter-telegram</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/claude</string>
        <string>--channels</string>
        <string>plugin:telegram@claude-plugins-official</string>
        <string>--dangerously-skip-permissions</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/peter</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/YOUR_USERNAME</string>
        <key>TELEGRAM_STATE_DIR</key>
        <string>/Users/YOUR_USERNAME/.claude/channels/telegram-peter</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/peter/logs/telegram-channel.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/peter/logs/telegram-channel.err</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

The key difference: `WorkingDirectory` points to `~/peter/`, so this agent reads `~/peter/CLAUDE.md` instead of `~/CLAUDE.md`.

### 14.6: Configure the Second Telegram Bot

Start Claude Code in the second agent's directory:

```bash
cd ~/peter
claude --channels plugin:telegram@claude-plugins-official
```

When prompted, enter Person B's Telegram bot token. Complete the pairing flow. Then exit and load the launchd service:

```bash
launchctl load ~/Library/LaunchAgents/com.claude.peter-telegram.plist
```

### 14.7: Shared vs. Separate Email

With a multi-account Gmail MCP (Phase 9.6), both agents can access all connected inboxes. Each agent's CLAUDE.md defines which inbox is "primary" for that agent:

- Talbot's CLAUDE.md: "Your primary inbox is natalia@every.to. Also monitor natalia.personal@gmail.com for bills and travel."
- Atlas's CLAUDE.md: "Your primary inbox is peter@stellation.vc. Also monitor peter.personal@gmail.com."

### 14.8: Rate Limit Considerations

Both agents share one Claude Max subscription. At $200/month:
- Two concurrent Opus sessions work fine for normal use
- Heavy parallel use (both agents doing long tasks simultaneously) may hit rate limits
- If this happens regularly, upgrade to Max 5x ($100/month more) for 5x the throughput

### 14.9: Shared Context Protocol

Define how the two agents communicate about shared items. Add to both CLAUDE.md files:

```markdown
## Shared Items Protocol
- Shared calendar: check P&N calendar for joint events
- Leo coverage: if both are at evening events, flag immediately
- Bills: household bills tracked in [shared location]
- When in doubt about the other person's schedule, check their calendar directly
```

-----

## Maintenance & Troubleshooting

### Check if services are running

```bash
launchctl list | grep com.claude    # All Claude services
ps aux | grep bot.py                 # Bot process
ps aux | grep cloudflared            # Tunnel process
```

### Read logs

```bash
tail -50 ~/Projects/slack-bot/bot.log           # Bot logs
tail -50 ~/Projects/slack-bot/bot-stderr.log     # Bot errors
tail -50 ~/Projects/slack-bot/email-watcher.log  # Email watcher
sudo cat /Library/Logs/com.cloudflare.cloudflared.err.log | tail -30  # Tunnel logs
```

### Restart a service

```bash
launchctl unload ~/Library/LaunchAgents/com.claude.homebase.plist
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist
```

### Bot not responding — debugging checklist

1. **Is the bot process running?** `ps aux | grep bot.py`
1. **Is the tunnel running?** `ps aux | grep cloudflared`
1. **Can the bot be reached locally?** `curl http://localhost:3000/slack/events` (should return 405)
1. **Can it be reached from outside?** `curl https://bot.yourdomain.com/slack/events` (should return 405)
1. **Is the Slack URL verified?** Check Event Subscriptions in api.slack.com
1. **Did you reinstall the app after changing scopes?** Go to Install App → Reinstall to Workspace
1. **Check logs** for errors: `tail -50 ~/Projects/slack-bot/bot-stderr.log`

### Tunnel shows error 1033

This means Cloudflare's tunnel is running but not routing. Check:

```bash
sudo cat /Library/Logs/com.cloudflare.cloudflared.err.log | tail -20
```

If you see "Use `cloudflared tunnel run` to start tunnel" repeated, the system plist is missing the `tunnel run` arguments. Fix it per Phase 6.6 instructions.

### Gmail watch expired

Renews on restart:

```bash
launchctl unload ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
```

### Google auth token expired

For `gws` CLI:

```bash
gws auth login --full
```

For Claude's built-in MCP: re-authorize through claude.ai/settings → Integrations.

### After a reboot

Everything should restart automatically. If not:

```bash
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-sweep.plist
launchctl load ~/Library/LaunchAgents/com.claude.file-browser.plist
launchctl load ~/Library/LaunchAgents/com.claude.telegram-channel.plist
```

-----

## Architecture Overview

```
PERSON A (Slack / Telegram / iMessage)    PERSON B (Telegram)
      |                                          |
      | Messages                                 | Messages
      v                                          v
CLOUDFLARE TUNNEL    TELEGRAM (polling)    TELEGRAM (polling)
      |                   |                      |
      v                   v                      v
+--------------------------------------------------+
|              YOUR MAC (always on)                 |
|                                                   |
|  AGENT A (e.g. "Talbot")                          |
|  WorkingDirectory: ~/                             |
|  ├── Slack bot (port 3000)                        |
|  ├── Telegram channel                             |
|  ├── iMessage channel                             |
|  └── CLAUDE.md → Chief of Staff persona           |
|                                                   |
|  AGENT B (e.g. "Atlas")                           |
|  WorkingDirectory: ~/peter/                       |
|  ├── Telegram channel (separate bot)              |
|  └── CLAUDE.md → VC Analyst persona               |
|                                                   |
|  SHARED SERVICES:                                 |
|  ├── Claude Code (the AI brain)                   |
|  ├── Google MCP (multi-account email access)      |
|  ├── Google Calendar (shared calendars)           |
|  ├── gws CLI (Drive, Docs, Sheets)                |
|  ├── GitHub (gh)                                  |
|  └── Gemini (images)                              |
|                                                   |
|  BACKGROUND SERVICES (launchd):                   |
|  ├── Slack bot (always on, auto-restart)          |
|  ├── Agent A Telegram (always on)                 |
|  ├── Agent B Telegram (always on)                 |
|  ├── iMessage channel (always on)                 |
|  ├── Email watcher (always on)                    |
|  ├── Email sweep (every 12 hours)                 |
|  ├── File browser (port 8889)                     |
|  └── Scheduled jobs (daily reports, etc.)         |
|                                                   |
|  CLOUDFLARE TUNNEL (system service)               |
|  └── Routes internet traffic to localhost          |
+--------------------------------------------------+
```

### Cost Summary

|Component                |Cost                       |
|-------------------------|---------------------------|
|Mac (mini or MacBook Air)|~$500-1,100 one-time       |
|Claude Max               |$200/month (covers all agents)|
|Cloudflare               |Free                       |
|Tailscale                |Free                       |
|Google Workspace         |Existing plan or free Gmail|
|GitHub                   |Free                       |
|Google Cloud (Pub/Sub)   |Free tier                  |
|Domain                   |~$10-15/year               |

### Key Files

|File                                                     |What it does                                              |
|---------------------------------------------------------|----------------------------------------------------------|
|`~/CLAUDE.md`                                            |Agent A operations manual — read every session            |
|`~/peter/CLAUDE.md`                                      |Agent B operations manual — read every session            |
|`~/identity.md`                                          |Agent A personality and principles                        |
|`~/peter/identity.md`                                    |Agent B personality and principles                        |
|`~/teammates/`                                           |Team roster and preferences                               |
|`~/Projects/slack-bot/bot.py`                            |The Slack bot server                                      |
|`~/Projects/slack-bot/.env`                              |Tokens and configuration                                  |
|`~/Projects/slack-bot/email-watcher.py`                  |Real-time email handler                                   |
|`~/.claude/settings.json`                                |Claude Code configuration                                 |
|`~/.cloudflared/config.yml`                              |Tunnel config (user copy)                                 |
|`/etc/cloudflared/config.yml`                            |Tunnel config (system copy — this is the one that matters)|
|`/Library/LaunchDaemons/com.cloudflare.cloudflared.plist`|Tunnel system service                                     |
|`~/Library/LaunchAgents/com.claude.*.plist`              |All bot/agent services                                    |

-----

## Known Bugs & Gotchas Summary

1. **Homebrew post-install commands:** If you close Terminal before running the two commands Homebrew shows after installation, just open a new Terminal and run `eval "$(/opt/homebrew/bin/brew shellenv)"` and add it to your `.zprofile`.
1. **npm "packages looking for funding":** Ignore it. It's just open-source maintainers accepting donations.
1. **"This is not an officially supported Google product":** Standard legal disclaimer on Google's open-source tools. Works fine.
1. **GitHub SSH key "Key is invalid":** Make sure you copied the entire key as one continuous line with no extra line breaks.
1. **Slack URL verification fails:** Expected until the tunnel is running AND the bot is running. Don't panic — come back to it after Phase 6.
1. **Slack bot not responding after setup:** Most likely you need to **Reinstall to Workspace** after changing scopes/manifest. Go to Install App → Reinstall to Workspace → Allow.
1. **Cloudflare tunnel error 1033:** The `sudo cloudflared service install` command creates a broken plist. You must (a) copy config to `/etc/cloudflared/`, (b) update credentials path, and (c) add `tunnel run` arguments to the system plist. This is the biggest bug in the setup.
1. **Cloudflare tunnel logs showing "Use cloudflared tunnel run":** Same as above — the system plist is missing the `tunnel run` arguments.
1. **Claude Code can't run `nano`:** It's interactive. Either use Claude Code to write files directly ("Write this content to this path") or exit Claude Code and use nano in a regular Terminal.
1. **Bot running but no messages reaching it:** Check that the Cloudflare tunnel is actually routing (curl the public URL). Check that Event Subscriptions URL is verified in Slack. Check that you clicked Save Changes after verification.
1. **`gws auth login` fails silently:** Make sure `~/.config/gws/client_secret.json` exists. You must download the OAuth client JSON from Google Cloud Console and place it there BEFORE running `gws auth login`. The CLI does not prompt you for Client ID/Secret — it reads from this file.
1. **Built-in MCP vs. gws confusion:** Claude's built-in Gmail/Calendar MCP (connected via claude.ai settings) is completely separate from the `gws` CLI. You can use both. The built-in MCP gives structured tools inside conversations; `gws` gives bash-level access for scripts and batch ops.

-----

*Built from an actual setup session at 2am-5am. Every bug in this guide was hit in real time. Updated with multi-user architecture and multi-account email support.*
