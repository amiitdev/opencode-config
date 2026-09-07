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
- [Antigravity Integration & Multi-Device Setup](#-antigravity-integration--multi-device-setup)
- [IP Rate-Limiting Flaw & Account Isolation](#-ip-rate-limiting-flaw--account-isolation)
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
- **Auto-routing**: Picks the best free provider automatically (`auto/best-coding`)
- **No API keys needed** for free providers (Antigravity OAuth, Kiro, OpenCode Free)
- **Dashboard** at `http://localhost:20128` to manage everything

---

### Step 1: Install OmniRoute

```bash
# Install globally
npm install -g omniroute

# Verify installation
omniroute --version
```

---

### Step 2: Start OmniRoute

```bash
# Start the server
omniroute
```

The dashboard opens automatically at `http://localhost:20128` (default password: `change me`).

---

## ⚡ Antigravity Integration & Multi-Device Setup

**Antigravity** provides access to Google DeepMind's Gemini models (`gemini-3.6-flash-high`, `gemini-3.5-pro`, etc.) with high speed (~60–70ms latency), full tool-calling support, and zero token cost via Google OAuth.

### How to Connect Antigravity in OmniRoute

1. Open OmniRoute Dashboard at `http://localhost:20128/dashboard`.
2. Go to **Providers** → **Antigravity**.
3. Click **Add Connection** → Sign in with your Google/Gmail account.
4. Click **Auto Sync** → **Test All Models**.

---

## ⚠️ IP Rate-Limiting Flaw & Account Isolation

### The Flaw with Default Free Tier Models (`oc/*`)
* Default unauthenticated free-tier models (like `oc/*`) enforce rate limiting **by public IPv4 address** (`49.47.132.137`).
* When multiple devices (e.g. laptop and phone) are connected to the same Wi-Fi network, they share the **exact same public IP address**.
* If laptop requests hit the free-tier rate limit, OpenCode on your phone **also gets rate-limited instantly** (`400: Error from provider / 429 Rate Limit Exceeded`).

### The Solution: Per-Device Google Account Isolation
Antigravity solves this vulnerability because quota is tracked **per Google Account**, not per IP address:

```
┌─────────────────────────────────────────────────────────┐
│               SAME WI-FI (IP: 49.47.132.137)            │
│                                                         │
│  Laptop (OmniRoute) ──> Google Account A               │
│                        (amitkumar.devnode@gmail.com)    │
│                        [Separate Quota]                │
│                                                         │
│  Phone  (OmniRoute) ──> Google Account B               │
│                        (igac1924@gmail.com)             │
│                        [Separate Quota]                │
└─────────────────────────────────────────────────────────┘
```

* **Laptop setup:** Uses Google Account `amitkumar.devnode@gmail.com`.
* **Phone setup:** Uses Google Account `igac1924@gmail.com`.
* **Result:** Exhausting limits on one device does **NOT** block or affect the other device, providing complete multi-device isolation.

---

## Config Files

| File | Description | Location |
|------|-------------|----------|
| `opencode.json` | Main active config (full 51 models) | `~/opencode.json` |
| `configs/opencode.json` | 15-model template separating `opencode-free` and `omniroute` | `opencode-config/configs/opencode.json` |
| `configs/opencode-github-mcp.json` | Standalone GitHub Copilot MCP configuration | `opencode-config/configs/opencode-github-mcp.json` |

---

## Providers & Models

### OmniRoute (Free Proxy) - Primary Provider

**Base URL:** `http://localhost:20128/v1`

| Model | Use Case | Context | Output |
|-------|----------|---------|--------|
| `auto/best-coding` | **Default** - Best for coding tasks (routes to Antigravity Gemini) | 1M | 384K |
| `auto/best-reasoning` | Complex reasoning & analysis | 1M | 384K |
| `auto/best-fast` | Quick tasks, fast responses | 1M | 384K |
| `auto/best-vision` | Image analysis & vision tasks | 1M | 384K |
| `auto/coding` | General coding assistance | 1M | 384K |
| `auto/fast` | Speed-optimized responses | 1M | 384K |
| `auto/chat` | Casual conversation | 1M | 384K |
| `auto/cheap` | Budget-friendly option | 1M | 384K |
| `auto/smart` | High intelligence tasks | 1M | 384K |
| `auto/pro-coding` | Professional-grade coding | 1M | 384K |

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

---

## Commands Reference

### Model Switching inside OpenCode
```bash
/model omniroute/auto/best-coding
/model omniroute/auto/best-fast
/model omniroute/auto/best-free
```

---

## Termux Setup

For Android devices using Termux:

```bash
pkg install nodejs npm
npm install -g omniroute opencode

# Start OmniRoute and connect Antigravity with your second Google account
omniroute
```

---

## License

MIT License - Use freely and modify as needed.

---

## Author

**amiitdev** - [GitHub](https://github.com/amiitdev)
