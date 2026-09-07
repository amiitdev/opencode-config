# OpenCode AI Configuration

My personal **OpenCode** setup - a powerful AI coding assistant with custom providers, models, and MCP integrations.

![OpenCode](https://img.shields.io/badge/OpenCode-AI-blue?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTEyIDJMMyA3djEwbDkgNSA5LTVIN0wxMiAyeiIgZmlsbD0iI2ZmZiIvPjwvc3ZnPg==)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Termux-lightgrey?style=flat-square)

---

## What is OpenCode?

OpenCode is an AI-powered coding assistant that runs in your terminal. It connects to various LLM providers, supports MCP (Model Context Protocol) servers, and helps you code, debug, and build projects faster.

---

## Table of Contents

- [Quick Start](#-quick-start)
- [OmniRoute Setup (Step by Step)](#-omniroute-setup-step-by-step)
- [Config Files](#-config-files)
- [Providers & Models](#-providers--models)
- [MCP Servers](#-mcp-servers)
- [Commands Reference](#-commands-reference)
- [OpenCode Setup Guide](#-opencode-setup-guide)
- [Termux Setup](#-termux-setup)
- [Keyboard Shortcuts](#-keyboard-shortcuts)

---

## Quick Start

```bash
# 1. Install OpenCode
npm install -g opencode

# 2. Clone this repo
git clone https://github.com/amiitdev/opencode-config.git
cd opencode-config

# 3. Copy config to your home directory
cp configs/opencode.json ~/opencode.json

# 4. Start OpenCode
opencode
```

---

## OmniRoute Setup (Step by Step)

OmniRoute is a free AI proxy that runs locally on your machine. It routes requests to multiple free LLM providers automatically. This is the **core** of the setup - OpenCode connects to OmniRoute to access free models.

### What is OmniRoute?

```
Your OpenCode  ──>  OmniRoute (localhost:20128)  ──>  Free AI Models
   (client)            (proxy/gateway)                (GPT, Claude, Gemini, etc.)
```

- **One URL** for all models: `http://localhost:20128/v1`
- **Auto-routing**: Picks the best free provider automatically
- **No API keys needed** for free providers (Kiro, OpenCode Free, Pollinations)
- **Dashboard** at `http://localhost:20128` to manage everything

---

### Step 1: Install OmniRoute

#### Option A: npm (Recommended)

```bash
# Install globally
npm install -g omniroute

# Verify installation
omniroute --version
```

#### Option B: Docker

```bash
docker run -d \
  --name omniroute \
  --restart unless-stopped \
  -p 127.0.0.1:20128:20128 \
  -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

#### Option C: From Source

```bash
git clone https://github.com/diegosouzapw/OmniRoute.git
cd OmniRoute
npm install
npm run dev
```

#### Option D: Termux (Android)

```bash
pkg install nodejs npm
npm install -g omniroute
```

---

### Step 2: Start OmniRoute

```bash
# Start the server
omniroute
```

You'll see output like:
```
OmniRoute v3.8.x
API:       http://localhost:20128/v1
Dashboard: http://localhost:20128
```

The dashboard opens automatically in your browser.

> **Tip:** Keep this terminal running. OmniRoute must be running whenever you use OpenCode.

---

### Step 3: Connect Providers

Open the dashboard at **http://localhost:20128** and go to **Providers**.

#### Option A: OpenRouter (What We Use - Recommended)

1. Dashboard → **Providers** → **Add Provider**
2. Select **OpenRouter**
3. Enter your OpenRouter API key (get one at https://openrouter.ai)
4. Click **Connect**
5. You now have access to 100+ models (GPT-4o, Claude, Gemini, Llama, etc.)

> **Our setup uses OpenRouter** - it gives access to all major models through one API key.

#### Option B: Kiro AI (Free Claude - No Credit Card)

1. Dashboard → **Providers** → **Add Provider**
2. Select **Kiro AI**
3. Click **Connect** (no API key needed!)
4. Free access to Claude models (~50 credits/month)

#### Option C: OpenCode Free (No Auth)

1. Dashboard → **Providers** → **Add Provider**
2. Select **OpenCode Free**
3. Click **Connect** (no API key needed!)
4. Access to multiple free models instantly

#### Option D: Pollinations (No Key Needed)

1. Dashboard → **Providers** → **Add Provider**
2. Select **Pollinations**
3. Click **Connect**
4. Access to GPT-5, Claude, Gemini, and more

#### Option E: Add Other Paid Providers

If you have API keys for OpenAI, Anthropic, etc.:

1. Dashboard → **Providers** → **Add Provider**
2. Select the provider (OpenAI, Anthropic, Google, etc.)
3. Enter your API key
4. Click **Connect**

---

### Step 4: Create an API Key

1. Open Dashboard → **Endpoints** (or **API Keys**)
2. Click **Create New Key**
3. Copy and save the key (it won't be shown again!)

> This key is for tools to access OmniRoute, not for accessing upstream providers.

---

### Step 5: Verify It Works

Test with curl:

```bash
# List available models
curl http://localhost:20128/v1/models \
  -H "Authorization: Bearer YOUR_KEY"

# Test a chat completion
curl http://localhost:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_KEY" \
  -d '{
    "model": "auto",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

You should see models listed and get a response. 

---

### Step 6: Point OpenCode to OmniRoute

Edit your `~/opencode.json`:

```json
{
  "provider": {
    "omniroute": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OmniRoute (Free)",
      "options": {
        "baseURL": "http://localhost:20128/v1"
      },
      "models": {
        "auto/best-coding": {
          "name": "Auto Best Coding",
          "tool_call": true,
          "reasoning": true,
          "limit": {
            "context": 1048576,
            "output": 384000
          }
        }
      }
    }
  },
  "model": "omniroute/auto/best-coding"
}
```

Key points:
- `baseURL` must be `http://localhost:20128/v1`
- Model names use the `omniroute/` prefix
- `auto` tells OmniRoute to pick the best provider automatically

#### Our Actual Config (with OpenRouter)

This is what our working config looks like with OpenRouter connected:

```json
{
  "provider": {
    "omniroute": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OmniRoute (Free)",
      "options": {
        "baseURL": "http://localhost:20128/v1"
      },
      "models": {
        "oc/mimo-v2.5-free": { "name": "mimo-v2.5-free" },
        "oc/deepseek-v4-flash-free": { "name": "deepseek-v4-flash-free" },
        "oc/hy3-free": { "name": "hy3-free" },
        "oc/nemotron-3-ultra-free": { "name": "nemotron-3-ultra-free" },
        "oc/north-mini-code-free": { "name": "north-mini-code-free" },
        "oc/big-pickle": { "name": "big-pickle" },
        "felo/felo-chat": { "name": "felo-chat" },
        "felo/felo-search": { "name": "felo-search" },
        "felo/felo-scholar": { "name": "felo-scholar" },
        "felo/felo-social": { "name": "felo-social" },
        "felo/felo-document": { "name": "felo-document", "attachment": true },
        "mcode/mimo-auto": { "name": "mimo-auto" }
      }
    }
  },
  "model": "omniroute/auto/best-coding",
  "mcp": {
    "gmail": {
      "type": "local",
      "command": ["npx", "-y", "@gongrzhe/server-gmail-autoauth-mcp"],
      "enabled": true
    }
  }
}
```

The OpenRouter models available through OmniRoute include:
- **GPT-4o / GPT-4o-mini** - OpenAI models
- **Claude Opus 4 / Sonnet 4** - Anthropic models
- **Gemini 2.5 Pro / Flash** - Google models
- **Llama 4 Maverick / Scout** - Meta models
- **DeepSeek R1 / V3** - DeepSeek models
- **Qwen3 235B / Coder** - Alibaba models
- And 100+ more models

---

### Step 7: Start Using OpenCode

```bash
# Make sure OmniRoute is running first!
omniroute &

# Then start OpenCode
opencode
```

Inside OpenCode, switch models with:
```
/model omniroute/auto/best-coding
/model omniroute/auto/best-fast
/model omniroute/auto/best-free
```

---

### OmniRoute CLI Commands

| Command | Description |
|---------|-------------|
| `omniroute` | Start server (port 20128) |
| `omniroute setup` | Guided first-run wizard |
| `omniroute doctor` | Health checks |
| `omniroute providers` | List/test providers |
| `omniroute config` | Manage configuration |
| `omniroute status` | Show status info |
| `omniroute logs` | Stream usage logs |
| `omniroute update` | Check for updates |
| `omniroute --help` | Show all options |

---

### OmniRoute Auto-Models

These models auto-route to the best available provider:

| Model | What It Does |
|-------|--------------|
| `auto` | Best available model (smart routing) |
| `auto/best-coding` | Best for code generation |
| `auto/best-reasoning` | Best for complex reasoning |
| `auto/best-fast` | Fastest response |
| `auto/best-free` | Best free model |
| `auto/best-vision` | Best for image analysis |
| `auto/coding` | General coding |
| `auto/chat` | Casual conversation |
| `auto/cheap` | Budget-friendly |

---

### OmniRoute + OpenCode Config Flow

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR MACHINE                         │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ OpenCode │───>│  OmniRoute   │───>│ Free Models  │  │
│  │ (client) │    │  :20128/v1   │    │ (cloud)      │  │
│  └──────────┘    └──────────────┘    └──────────────┘  │
│       │                │                               │
│       │                ├── Kiro (free Claude)          │
│       │                ├── OpenCode Free               │
│       │                ├── Pollinations                │
│       │                └── Your paid APIs              │
│       │                                                │
│  ┌────┴─────┐                                         │
│  │ MCP      │                                         │
│  │ Servers  │                                         │
│  │ (Gmail,  │                                         │
│  │  GitHub) │                                         │
│  └──────────┘                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Troubleshooting OmniRoute

**OmniRoute won't start:**
```bash
# Check if port is in use
lsof -i :20128

# Kill existing process
kill $(lsof -t -i :20128)

# Try again
omniroute
```

**Check OmniRoute status:**
```bash
# Full status check
omniroute status

# List configured providers
omniroute providers list

# Test all providers
omniroute providers test-all

# Check available providers in catalog
omniroute providers available
```

**Provider shows "credits_exhausted":**
```bash
# Check provider status
omniroute providers status

# You may need to:
# 1. Add more credits to your OpenRouter account
# 2. Switch to a free provider (Kiro, OpenCode Free)
# 3. Use a different provider
```

**No models available:**
```bash
# Check what providers are connected
omniroute providers list

# Test the connection
omniroute providers test-all

# Re-run setup wizard
omniroute setup
```

**OpenCode can't connect:**
```bash
# Verify OmniRoute is running
curl http://localhost:20128/v1/models

# Check your config has correct baseURL
cat ~/opencode.json | grep baseURL

# Verify OpenCode config
omniroute config list
```

**Dashboard not opening:**
```bash
# Open manually
open http://localhost:20128
# or
xdg-open http://localhost:20128
```

**Check logs:**
```bash
# Stream live logs
omniroute logs --follow

# Check application logs
ls -la ~/.omniroute/logs/application/
```

---

## Config Files

| File | Description | Location |
|------|-------------|----------|
| `opencode.json` | Main config (desktop/Linux) | `~/opencode.json` |
| `opencode-termux.json` | Config for Termux/Android | `~/opencode.json` (on Termux) |
| `.opencode/` | OpenCode runtime directory | `~/.opencode/` |

### Config Structure

```
~/opencode.json          # Main configuration
~/.opencode/             # Runtime files
  ├── package.json       # Node dependencies
  ├── node_modules/      # Installed packages
  └── bin/               # Binary files
```

---

## Providers & Models

### OmniRoute (Free Proxy) - Primary Provider

**Base URL:** `http://localhost:20128/v1`

| Model | Use Case | Context | Output |
|-------|----------|---------|--------|
| `auto/best-coding` | **Default** - Best for coding tasks | 1M | 384K |
| `auto/best-reasoning` | Complex reasoning & analysis | 1M | 384K |
| `auto/best-fast` | Quick tasks, fast responses | 1M | 384K |
| `auto/best-vision` | Image analysis & vision tasks | 1M | 384K |
| `auto/coding` | General coding assistance | 1M | 384K |
| `auto/fast` | Speed-optimized responses | 1M | 384K |
| `auto/chat` | Casual conversation | 1M | 384K |
| `auto/cheap` | Budget-friendly option | 1M | 384K |
| `auto/smart` | High intelligence tasks | 1M | 384K |
| `auto/pro-coding` | Professional-grade coding | 1M | 384K |
| `auto/pro-reasoning` | Advanced reasoning | 1M | 384K |
| `auto/claude-opus` | Claude Opus via proxy | 1M | 384K |
| `auto/claude-sonnet` | Claude Sonnet via proxy | 1M | 384K |
| `auto/best-free` | Best free model available | 1M | 384K |
| `auto/offline` | Offline/local models | 1M | 384K |
| `auto/vision` | Vision-capable models | 1M | 384K |
| `auto/multimodal` | Multimodal tasks | 1M | 384K |
| `auto/glm` | GLM models | 1M | 384K |
| `auto/minimax` | MiniMax models | 1M | 384K |
| `auto/mimo` | Mimo models | 1M | 384K |
| `auto/gemma` | Google Gemma models | 1M | 384K |
| `auto/llama` | Meta Llama models | 1M | 384K |
| `auto/gemini` | Google Gemini models | 1M | 384K |

### OmniRoute Free Models

| Model | Description |
|-------|-------------|
| `oc/mimo-v2.5-free` | Mimo v2.5 Free |
| `oc/deepseek-v4-flash-free` | DeepSeek v4 Flash Free |
| `oc/hy3-free` | HY3 Free |
| `oc/nemotron-3-ultra-free` | Nemotron 3 Ultra Free |
| `oc/north-mini-code-free` | North Mini Code Free |
| `oc/big-pickle` | Big Pickle |

### Felo Models

| Model | Description |
|-------|-------------|
| `felo/felo-chat` | General chat |
| `felo/felo-search` | Search-augmented |
| `felo/felo-scholar` | Academic/research |
| `felo/felo-social` | Social media focused |
| `felo/felo-document` | Document analysis (supports attachments) |

### Ollama Cloud

**Base URL:** `https://ollama.com/v1`

| Model | Description |
|-------|-------------|
| `gpt-oss:120b` | GPT-OSS 120B parameter |
| `gpt-oss:20b` | GPT-OSS 20B parameter |
| `gemma4:31b` | Gemma 4 31B |
| `minimax-m3` | MiniMax M3 |
| `nemotron-3-nano:30b` | Nemotron 3 Nano 30B |
| `nemotron-3-super` | Nemotron 3 Super |
| `nemotron-3-ultra` | Nemotron 3 Ultra |

### Ollama (Local)

**Base URL:** `http://localhost:11434/v1`

| Model | Description |
|-------|-------------|
| `qwen3.5` | Qwen 3.5 (local) |

---

## MCP Servers

### Gmail (Local)

Auto-authenticates with your Google account.

```json
{
  "gmail": {
    "type": "local",
    "command": ["npx", "-y", "@gongrzhe/server-gmail-autoauth-mcp"],
    "enabled": true
  }
}
```

**Capabilities:**
- Read/send emails
- Search inbox
- Manage labels
- Draft messages

### GitHub (Remote)

Connects to GitHub API via personal access token.

```json
{
  "github": {
    "type": "remote",
    "url": "https://api.githubcopilot.com/mcp/",
    "enabled": true,
    "headers": {
      "Authorization": "Bearer {env:GITHUB_PERSONAL_ACCESS_TOKEN}"
    }
  }
}
```

**Setup:**
1. Go to GitHub Settings > Developer Settings > Personal Access Tokens
2. Create a token with `repo`, `gist`, `read:org`, `workflow` scopes
3. Set environment variable:
   ```bash
   export GITHUB_PERSONAL_ACCESS_TOKEN="ghp_your_token_here"
   ```

---

## Commands Reference

### Starting OpenCode

```bash
# Start in current directory
opencode

# Start in specific directory
opencode /path/to/project

# With specific config
opencode --config ~/opencode-termux.json
```

### Within OpenCode Terminal

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel current operation |
| `Ctrl+D` | Exit OpenCode |
| `Ctrl+L` | Clear terminal |
| `Tab` | Autocomplete |
| `↑/↓` | Navigate history |
| `Ctrl+R` | Search history |

### Slash Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/clear` | Clear conversation |
| `/model` | Switch model |
| `/compact` | Compress conversation |
| `/cost` | Show token usage |
| `/config` | Open configuration |
| `/quit` | Exit OpenCode |

### Model Switching

```bash
# Switch to best coding model
/model omniroute/auto/best-coding

# Switch to fast model
/model omniroute/auto/best-fast

# Switch to vision model
/model omniroute/auto/best-vision

# Switch to free model
/model omniroute/auto/best-free
```

---

## Setup Guide

### Prerequisites

```bash
# Node.js (v18+)
node --version

# npm
npm --version

# Git
git --version

# GitHub CLI (optional)
gh --version
```

### Installation

```bash
# Install OpenCode globally
npm install -g opencode

# Verify installation
opencode --version
```

### Configuration

1. **Copy the config file:**
   ```bash
   cp configs/opencode.json ~/opencode.json
   ```

2. **Set up GitHub token (optional):**
   ```bash
   # Add to ~/.bashrc or ~/.zshrc
   export GITHUB_PERSONAL_ACCESS_TOKEN="ghp_xxxxxxxxxxxx"
   ```

3. **Start OmniRoute proxy (required for free models):**
   ```bash
   # Start the OmniRoute local proxy
   omniroute start
   # Or if using a different proxy:
   # Your proxy command here
   ```

4. **Run OpenCode:**
   ```bash
   opencode
   ```

---

## Termux Setup

For Android devices using Termux:

```bash
# Install Node.js in Termux
pkg install nodejs npm

# Install OpenCode
npm install -g opencode

# Copy the Termux-specific config
cp configs/opencode-termux.json ~/opencode.json

# Start OpenCode
opencode
```

### Termux Config Differences

The Termux config (`opencode-termux.json`) includes:
- Same OmniRoute models
- Additional OpenRouter free models (vision-capable)
- Optimized for mobile use

---

## Keyboard Shortcuts

### Navigation
| Key | Action |
|-----|--------|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+U` | Clear line |
| `Ctrl+K` | Kill to end of line |
| `Ctrl+W` | Delete word backward |

### OpenCode Specific
| Key | Action |
|-----|--------|
| `Ctrl+C` | Cancel/Interrupt |
| `Ctrl+D` | Exit |
| `Ctrl+L` | Clear screen |
| `Tab` | Autocomplete |

---

## Environment Variables

```bash
# GitHub Integration
export GITHUB_PERSONAL_ACCESS_TOKEN="ghp_xxxxxxxxxxxx"

# OmniRoute (if not using localhost)
export OMNIROUTE_URL="http://your-proxy:20128/v1"

# Ollama (if not default)
export OLLAMA_URL="http://localhost:11434/v1"
```

---

## File Locations

| Platform | Config Path | Runtime Path |
|----------|-------------|--------------|
| Linux | `~/opencode.json` | `~/.opencode/` |
| macOS | `~/opencode.json` | `~/.opencode/` |
| Termux | `~/opencode.json` | `~/.opencode/` |

---

## Troubleshooting

### Common Issues

**1. "Provider not found" error**
```bash
# Ensure OmniRoute is running
curl http://localhost:20128/v1/models
```

**2. "MCP server failed to start"**
```bash
# Reinstall Gmail MCP
npx -y @gongrzhe/server-gmail-autoauth-mcp
```

**3. "GitHub authentication failed"**
```bash
# Check your token
echo $GITHUB_PERSONAL_ACCESS_TOKEN

# Re-authenticate
gh auth login
```

**4. Model not responding**
```bash
# Switch to a different model
/model omniroute/auto/best-coding
```

---

## License

MIT License - Use freely and modify as needed.

---

## Author

**amiitdev** - [GitHub](https://github.com/amiitdev)

---

> Built with OpenCode AI | Configured for maximum productivity
