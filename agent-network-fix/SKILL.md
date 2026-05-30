---
name: agent-network-fix
description: Fix network access issues when opencode's bash tool cannot reach external URLs. Use when curl/git commands in bash return 000 or timeout, when GitHub clone fails, or when the user reports that webfetch works but bash network requests fail. This skill diagnoses macOS system proxy misconfiguration and adds proxy environment variables to ~/.zshenv so every new opencode session inherits correct proxy settings.
---

# Agent Network Fix

Diagnose and fix network connectivity issues in opencode sessions where the `bash` tool cannot access external resources while `webfetch`/`websearch` work fine.

## Root Cause

opencode's `bash` tool runs in a **non-interactive shell** that does not load `~/.zshrc` or `~/.bashrc`. If the user's macOS has a system proxy configured (e.g., `127.0.0.1:6789` for Clash/Surge/Shadowsocks), but no proxy environment variables are exported in `~/.zshenv`, all `curl`/`git`/`wget` calls from the bash tool will fail with HTTP status `000` or timeout.

Meanwhile, `webfetch` and `websearch` have their own built-in network routing and work independently.

## Diagnostic Steps

### Step 1: Confirm the symptom

Run these tests in parallel:

```bash
curl -s -o /dev/null -w "%{http_code}" https://github.com
curl -s -o /dev/null -w "%{http_code}" https://google.com
```

If both return `000`, the bash tool has no network access. Proceed.

### Step 2: Check for system proxy

```bash
networksetup -getwebproxy Wi-Fi 2>/dev/null
networksetup -getsecurewebproxy Wi-Fi 2>/dev/null
```

Look for `Enabled: Yes` with a `Server` and `Port`. If both say `Enabled: No`, the issue is not proxy-related — check firewall, DNS, or other network problems instead.

### Step 3: Check current env var state

```bash
echo "http_proxy=$http_proxy"
echo "https_proxy=$https_proxy"
echo "all_proxy=$all_proxy"
```

If all are empty, this confirms the root cause: proxy env vars are missing from the non-interactive shell.

### Step 4: Check existing shell config

```bash
cat ~/.zshenv 2>/dev/null
```

If `~/.zshenv` already contains proxy exports, the issue may be a stale proxy process (port not listening). Verify with:

```bash
lsof -i :<PORT> 2>/dev/null | head -3
```

If nothing is listening on the proxy port, the proxy app is not running. Tell the user to start their proxy software.

## Fix Steps

### Step 1: Add proxy to ~/.zshenv

Read the current `~/.zshenv` file. If proxy variables are not already present, append them:

```bash
# Proxy settings for opencode bash tool
export http_proxy=http://127.0.0.1:<PORT>
export https_proxy=http://127.0.0.1:<PORT>
export all_proxy=http://127.0.0.1:<PORT>
```

Replace `<PORT>` with the port from Step 2 (commonly `6789`, `7890`, or `1080`).

**Important:**
- Use `~/.zshenv`, NOT `~/.zshrc`. The `.zshenv` file is sourced for ALL zsh invocations, including non-interactive shells.
- Do not duplicate entries. If the file already has proxy lines, update them instead of adding more.
- Preserve any existing content in `~/.zshenv`.

### Step 2: Apply immediately for current session

Export the variables in the current shell so subsequent commands work without restart:

```bash
export http_proxy=http://127.0.0.1:<PORT>
export https_proxy=http://127.0.0.1:<PORT>
export all_proxy=http://127.0.0.1:<PORT>
```

### Step 3: Verify

```bash
curl -s -o /dev/null -w "%{http_code}" https://github.com
```

Should return `200`. Then test git:

```bash
git ls-remote --heads https://github.com/opencode-ai/opencode.git main 2>&1 | head -1
```

Should return a commit hash.

## Edge Cases

### Ethernet instead of Wi-Fi

If `networksetup -getwebproxy Wi-Fi` returns "Error: Invalid service name", try:

```bash
networksetup -listallnetworkservices
```

Use the active service name (e.g., "Ethernet", "Thunderbolt Bridge") instead of "Wi-Fi".

### Multiple proxy apps

Some users run multiple proxy tools. The active one is whichever port has a listening process. Use `lsof -i :<PORT>` to verify which ports are active.

### SOCKS proxy

If the system uses a SOCKS proxy instead of HTTP/HTTPS:

```bash
networksetup -getsocksfirewallproxy Wi-Fi 2>/dev/null
```

If enabled, use `socks5://` scheme:

```bash
export all_proxy=socks5://127.0.0.1:<PORT>
```

Note: `curl` supports SOCKS natively, but `git` may need `GIT_SSH_COMMAND` or `core.gitProxy` configuration for SSH URLs.

### Git-specific proxy

If HTTP proxy is set but git over SSH still fails, configure git proxy:

```bash
git config --global http.proxy http://127.0.0.1:<PORT>
git config --global https.proxy http://127.0.0.1:<PORT>
```

## Summary for User

After fixing, tell the user:

> The issue was that opencode's bash tool runs in a non-interactive shell that doesn't load `~/.zshrc`. Your macOS system proxy (`127.0.0.1:<PORT>`) was not being inherited by bash commands.
>
> I've added proxy environment variables to `~/.zshenv`, which is sourced for ALL zsh sessions. This fix persists across opencode restarts and new sessions.
