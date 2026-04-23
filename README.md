# Hermes Agent - Key Setup Troubleshooting Guide

Complete documentation of API key setup issues encountered and solutions applied when configuring Hermes Agent.

## Problem Overview

Hermes Agent was experiencing API authentication and model compatibility issues during initial setup.

## Initial Issues

### 1. No Working LLM Provider
- OpenRouter API key was set but returning 401 "User not found" error
- Hermes gateway couldn't establish connection to any LLM provider
- Agent was partially running but unable to process requests

### 2. Crypto Integration Required
- Hermes needed ATXP connection for cryptocurrency operations
- No ATXP_CONNECTION string configured

## Troubleshooting Steps

### Step 1: Fix ATXP Connection

**Issue:** Missing ATXP configuration

**Solution:**
```bash
# Added to ~/.hermes/.env
export ATXP_CONNECTION="https://accounts.atxp.ai?connection_token=YOUR_TOKEN_HERE"
```

**Notes:** ATXP connection string was received separately and added to both `.env.hermes` (workspace) and actual `~/.hermes/.env` file.

---

### Step 2: Fix OpenRouter Model Endpoint (404 Error)

**Error:**
```
❌ Non-retryable error (HTTP 404): No endpoints found for meta-llama/llama-3.1-8b-instruct:free
```

**Root Cause:**
- Model string included `:free` suffix
- OpenRouter doesn't support `:free` variant in public API

**Solution:**
```yaml
# ~/.hermes/config.yaml - BEFORE
model:
  default: "meta-llama/llama-3.1-8b-instruct:free"
  provider: "openrouter"

# ~/.hermes/config.yaml - AFTER
model:
  default: "meta-llama/llama-3.1-8b-instruct"
  provider: "openrouter"
```

**Action:** Removed `:free` from model identifier, restarted gateway.

---

### Step 3: Fix Context Window Issue (ValueError)

**Error:**
```
ValueError: Model meta-llama/llama-3.1-8b-instruct has a context window of 16,384 tokens,
which is below the minimum 64,000 required by Hermes Agent.
Choose a model with at least 64K context, or set model.context_length in config.yaml to override.
```

**Root Cause:**
- Llama 3.1 8B has 16,384 token context window
- Hermes requires minimum 64K tokens by default

**Temporary Solution (Context Override):**
```yaml
model:
  default: "meta-llama/llama-3.1-8b-instruct"
  provider: "openrouter"
  base_url: "https://openrouter.ai/api/v1"
  max_tokens: 4096
  temperature: 0.7
  context_length: 16384  # Override Hermes requirement
```

**Better Solution (Model Upgrade):**
Later upgraded to larger model with better context (see Step 6).

---

### Step 4: Switch to Groq (Provider Error)

**Goal:** Use Groq's free Llama 3.3 70B model with 128K context

**Initial Attempt:**
```yaml
model:
  default: "llama-3.3-70b-versatile"
  provider: "groq"
  base_url: "https://api.groq.com/openai/v1"
```

**Error:**
```
⚠️ Provider authentication failed: Unknown provider 'groq'
```

**Root Cause:**
- Hermes doesn't have built-in Groq provider
- Custom base URL override doesn't automatically register new providers

**Failed Alternative - OpenAI Provider:**
```yaml
model:
  default: "llama-3.3-70b-versatile"
  provider: "openai"
  base_url: "https://api.groq.com/openai/v1"
```

**Error:**
```
⚠️ Provider authentication failed: Unknown provider 'openai'
```

**Root Cause:**
- Hermes requires explicit provider authentication
- OpenAI provider doesn't work with OpenAI-compatible endpoints

---

### Step 5: Save Groq API Key (Not Used)

**Configuration:**
```bash
# ~/.hermes/.env
GROQ_API_KEY=gsk_YOUR_KEY_HERE
OPENAI_API_KEY=$GROQ_API_KEY
```

**Note:** Groq key was saved in `.env` but couldn't be used due to provider issues. Kept as backup for future attempts.

---

### Step 6: Final Working Configuration (OpenRouter)

**Solution:**
- Reverted to OpenRouter (supported provider)
- Upgraded to larger model with better context
- Kept ATXP configuration active

```yaml
# ~/.hermes/config.yaml
model:
  default: "meta-llama/llama-3.1-70b-instruct"
  provider: "openrouter"
  base_url: "https://openrouter.ai/api/v1"
  max_tokens: 8192
  temperature: 0.7
```

```bash
# ~/.hermes/.env
# Telegram
TELEGRAM_BOT_TOKEN=YOUR_TELEGRAM_BOT_TOKEN
TELEGRAM_ENABLED=true
TELEGRAM_ALLOWED_USERS=YOUR_USER_ID

# LLM Provider
OPENROUTER_API_KEY=sk-or-v1-YOUR_OPENROUTER_KEY_HERE

# Crypto Integration
ATXP_CONNECTION="https://accounts.atxp.ai?connection_token=YOUR_TOKEN_HERE"

# Groq (Backup, not currently used)
GROQ_API_KEY=sk_YOUR_GROQ_KEY_HERE
OPENAI_API_KEY=$GROQ_API_KEY
```

**Gateway Commands:**
```bash
# Restart Hermes gateway
kill $(cat ~/.hermes/gateway.pid | jq -r '.pid')
cd ~/.hermes && source .env && ~/.hermes/hermes-agent/venv/bin/hermes gateway run > /dev/null 2>&1 &

# Update gateway.pid
echo '{"pid": '$(pgrep -f "hermes gateway run" | head -1)', "kind": "hermes-gateway", "argv": ["~/.hermes/hermes-agent/venv/bin/hermes", "gateway", "run"], "start_time": '$(date +%s)'}' > ~/.hermes/gateway.pid

# Check status
hermes status
hermes doctor
```

---

## Current Working Status

### Gateway Status
```
◆ Environment
  Project:      /home/openclaw/.hermes/hermes-agent
  Python:       3.11.15
  .env file:    ✓ exists
  Model:        meta-llama/llama-3.1-70b-instruct
  Provider:     OpenRouter

◆ API Keys
  OpenRouter    ✓ sk-o...6cd0

◆ Platform Connection
  Telegram      ✓ configured

◆ Gateway Service
  Status:       ✓ running
```

### Session Activity
- **Session ID:** 20260423_114423_8b2fd712
- **Created:** 2026-04-23 11:44
- **Last Activity:** 2026-04-23 17:24
- **Tokens Used:** 0 (fresh install)
- **Platform:** Telegram connected and active

---

## Lessons Learned

### 1. Model Context Window Requirements
Hermes has strict model requirements. Always verify:
- Context window size (minimum 64K unless overridden)
- Model availability on selected provider
- Endpoint format (avoid `:free` suffixes unless verified)

### 2. Provider Compatibility
Hermes has a fixed provider list:
- `openrouter` ✅ (working)
- `openai` ❌ (doesn't support custom endpoints)
- `groq` ❌ (not built-in)
- `nvidia` ✅ (potentially supported if configured)

For non-built-in providers, use `openrouter` as proxy or configure custom integration via skills.

### 3. Configuration File Locations
- **Gateway config:** `~/.hermes/.env` (primary)
- **Workspace config:** `~/.hermes/config.yaml` (model settings)
- **PID tracking:** `~/.hermes/gateway.pid` (for restarts)

### 4. Gateway Restart Process
```bash
# 1. Stop existing process
kill $(cat ~/.hermes/gateway.pid | jq -r '.pid')

# 2. Source environment variables
cd ~/.hermes && source .env

# 3. Start gateway
~/.hermes/hermes-agent/venv/bin/hermes gateway run > /dev/null 2>&1 &

# 4. Update PID tracking
# ...
```

### 5. Troubleshooting Commands
```bash
# Check gateway status
hermes status

# Diagnose issues
hermes doctor

# Check running processes
ps aux | grep hermes

# View configured models/providers
hermes model  # (Interactive, requires TTY)
```

---

## Future Improvements

1. **Groq Integration:** If Hermes adds native Groq provider support, can migrate back to Groq:
   ```yaml
   model:
     default: "llama-3.3-70b-versatile"
     provider: "groq"  # If supported in future version
   ```

2. **Free Model Options:** Explore alternative free tier models with 64K+ context on OpenRouter.

3. **Multi-provider Setup:** Configure failover providers in case primary fails.

---

## Environment Details

- **System:** Linux 5.15.0-173-generic (x64)
- **Python:** 3.11.15
- **Hermes Version:** v22
- **Node:** v22.22.2
- **Location:** `/home/openclaw/.hermes/hermes-agent`

---

## Additional Resources

- **Hermes Documentation:** Check local docs or `hermes setup`
- **OpenRouter Models:** https://openrouter.ai/models
- **Groq Console:** https://console.groq.com/keys
- **Configuration Template:** See `~/.hermes/config.yaml` for detailed options

---

## Change Log

| Date | Change | Status |
|------|--------|--------|
| 2026-04-23 14:05 | Started key setup troubleshooting | 🔄 |
| 2026-04-23 14:07 | Added ATXP_CONNECTION configuration | ✅ |
| 2026-04-23 14:09 | Fixed OpenRouter model endpoint (404) | ✅ |
| 2026-04-23 14:13 | Added context_length override for 8B model | ✅ |
| 2026-04-23 17:15 | Attempted Groq provider setup | ❌ |
| 2026-04-23 17:18 | Fallback to OpenRouter with 70B model | ✅ |
| 2026-04-23 17:24 | Gateway fully operational | ✅ |
| 2026-04-23 17:27 | Created GitHub repository with documentation | ✅ |

---

## GitHub Repository

This documentation has been published to GitHub for easy reference and collaboration.

**Repository:** https://github.com/panditvirchandra77-lgtm/hermes-key-setup-troubleshooting

**Purpose:**
- Live documentation of Hermes Agent setup issues
- Reference for future troubleshooting
- Community contribution welcome

---

**Last Updated:** 2026-04-23

---

*This documentation was created during live troubleshooting session. Use as reference for similar setups.*