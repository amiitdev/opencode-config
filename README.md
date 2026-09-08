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

- [What is OpenCode?](#what-is-opencode)
- [Quick Start](#quick-start)
- [OmniRoute Setup (Step by Step)](#omniroute-setup-step-by-step)
- [Antigravity Integration & Multi-Device Setup](#antigravity-integration--multi-device-setup)
- [IP Rate-Limiting Flaw & Account Isolation](#ip-rate-limiting-flaw--account-isolation)
- [Config Files](#config-files)
- [Providers & Models](#providers--models)
- [MCP Servers](#mcp-servers)
  - [Gmail](#gmail-local)
  - [Reddit](#reddit-mcp-buddy)
  - [Playwright (Browser Agent)](#playwright-browser-agent)
  - [Render (Cloud Infrastructure)](#render-remote)
  - [GitHub](#github-remote)
- [Browser Agent Authentication](#browser-agent-authentication)
- [Commands Reference](#commands-reference)
- [Termux Setup](#termux-setup)
- [License](#license)
- [Author](#author)

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

## Antigravity Integration & Multi-Device Setup

**Antigravity** provides access to Google DeepMind's Gemini models (`gemini-3.6-flash-high`, `gemini-3.5-pro`, etc.) with high speed (~60–70ms latency), full tool-calling support, and zero token cost via Google OAuth.

### How to Connect Antigravity in OmniRoute

1. Open OmniRoute Dashboard at `http://localhost:20128/dashboard`.
2. Go to **Providers** → **Antigravity**.
3. Click **Add Connection** → Sign in with your Google/Gmail account.
4. Click **Auto Sync** → **Test All Models**.

---

## IP Rate-Limiting Flaw & Account Isolation

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

---

### Reddit (MCP Buddy)
Browse Reddit, search posts, get comments, and analyze users. Zero setup required — works anonymously (10 rpm). Add free Reddit API credentials for 100 rpm.

| Feature | Description |
|---------|-------------|
| `browse_subreddit` | Browse any subreddit (hot/new/top/rising) |
| `search_reddit` | Search across Reddit or specific subreddits |
| `get_post_details` | Fetch post with full comment threads |
| `user_analysis` | Analyze any Reddit user's karma and activity |
| `reddit_explain` | Explain Reddit terms and slang |

**Anonymous mode** (no credentials needed):
```json
{
  "reddit": {
    "type": "local",
    "command": ["npx", "-y", "reddit-mcp-buddy"],
    "enabled": true
  }
}
```

**Authenticated mode** (100 rpm, optional):
```json
{
  "reddit": {
    "type": "local",
    "command": ["npx", "-y", "reddit-mcp-buddy"],
    "enabled": true,
    "environment": {
      "REDDIT_CLIENT_ID": "your_client_id",
      "REDDIT_CLIENT_SECRET": "your_client_secret",
      "REDDIT_USERNAME": "your_username",
      "REDDIT_PASSWORD": "your_password"
    }
  }
}
```

**Get free Reddit credentials:**
1. Go to https://www.reddit.com/prefs/apps
2. Click "create another app..."
3. Select type: **script** (critical for 100 rpm)
4. Redirect URI: `http://localhost:8080`
5. Copy Client ID and Secret

---

### Playwright (Browser Agent)
Control a web browser — click buttons, fill forms, navigate websites. Supports persistent sessions so you can log into sites once and reuse the session.

| Feature | Description |
|---------|-------------|
| `browser_navigate` | Open any URL |
| `browser_click` | Click buttons and links |
| `browser_type` | Type into text boxes |
| `browser_snapshot` | See what's on the page |
| `browser_screenshot` | Take screenshots |
| `browser_evaluate` | Run JavaScript on the page |

```json
{
  "playwright": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp", "--user-data-dir", "/home/amit/.playwright-auth"],
    "enabled": true
  }
}
```

**How it works:**
```
┌─────────────────────────────────────────┐
│  First time: Manual login               │
│  Open browser → Log into sites          │
│  → Cookies saved to ~/.playwright-auth/ │
├─────────────────────────────────────────┤
│  After restart: Auto-login              │
│  OpenCode starts Playwright MCP         │
│  → Reads saved cookies                  │
│  → Already logged into Reddit/Twitter   │
│  → AI can now browse as you             │
└─────────────────────────────────────────┘
```

**See [Browser Agent Authentication](#browser-agent-authentication) below for setup.**

---

### Render (Remote)
Manage your Render cloud infrastructure — web services, databases, deploys, logs, and metrics — directly from OpenCode using natural language.

| Feature | Description |
|---------|-------------|
| `list_services` | List all services in your workspace |
| `create_web_service` | Create new web services (Node, Python, Go, Docker, etc.) |
| `trigger_deploy` | Deploy or redeploy services |
| `list_logs` | View service logs with filters |
| `get_metrics` | CPU, memory, instance count, HTTP metrics |
| `query_render_postgres` | Run SQL queries on your databases |
| `create_postgres` | Create new Postgres databases |
| `create_key_value` | Create Redis-compatible key-value stores |

**Setup:**

1. Get your API key from https://dashboard.render.com → Account Settings → API Keys
2. Set environment variable:
```bash
export RENDER_API_KEY="rnd_your_api_key_here"
```

3. Add to `~/.zshrc` for persistence:
```bash
echo 'export RENDER_API_KEY="rnd_your_api_key_here"' >> ~/.zshrc
```

**Config:**
```json
{
  "render": {
    "type": "remote",
    "url": "https://mcp.render.com/mcp",
    "enabled": true,
    "headers": {
      "Authorization": "Bearer {env:RENDER_API_KEY}"
    }
  }
}
```

**Example prompts:**
- "List my Render services"
- "Create a new Postgres database named user-db with 5 GB storage"
- "Show me the error logs for my API service"
- "Deploy my web service and clear the build cache"
- "What was the busiest traffic day for my service this month?"

---

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

## Browser Agent Authentication

To log into sites like Reddit, Twitter, or LinkedIn (which require 2FA/CAPTCHAs), use the **persistent browser session** approach:

### Step 1: Open browser for manual login

```bash
# Install Playwright if not already installed
cd /tmp && npm install playwright

# Create login script
cat > /tmp/login-browser.js << 'EOF'
const { chromium } = require('playwright');

(async () => {
  const context = await chromium.launchPersistentContext('/home/amit/.playwright-auth', {
    headless: false,
    viewport: { width: 1280, height: 800 }
  });
  
  const page = context.pages()[0] || await context.newPage();
  await page.goto('https://www.reddit.com/login');
  
  console.log('='.repeat(50));
  console.log('BROWSER OPENED - Log into your sites:');
  console.log('  1. Reddit is open - log in now');
  console.log('  2. Then open Twitter, LinkedIn etc.');
  console.log('  3. When DONE, press Ctrl+C');
  console.log('='.repeat(50));
  
  process.on('SIGINT', async () => {
    console.log('\nSaving session...');
    await context.close();
    console.log('Session saved to /home/amit/.playwright-auth/');
    process.exit(0);
  });
  
  await new Promise(() => {});
})();
EOF

# Run it
node /tmp/login-browser.js
```

### Step 2: Log into sites manually
- Reddit login page opens
- Log in normally (complete 2FA, CAPTCHAs, etc.)
- Open new tabs for Twitter, LinkedIn, etc.

### Step 3: Save session
- Press `Ctrl+C` in terminal
- Session saves to `~/.playwright-auth/`

### Step 4: Use in OpenCode
After restart, ask OpenCode:
- "Open Reddit and show my homepage"
- "Go to Twitter and check my notifications"
- "Fill out the login form on example.com"

**Note:** Google blocks automated browsers. Use the Gmail MCP server for Google/Gmail instead.

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
