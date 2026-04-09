> **🇰🇷 [한국어 버전](ko/09_QUICKSTART.md)**

# Quick Start — Setup & Self-Diagnosis

> Practical steps to minimize token drain. Two approaches: **stay current** (v2.1.91+) or **downgrade** (v2.1.63). Both are valid — choose based on your workflow. For the technical analysis behind these recommendations, see [01_BUGS.md](01_BUGS.md).

---

## Which Approach Is Right for You?

| | **Option A: Stay Current (v2.1.91+)** | **Option B: Downgrade (v2.1.63)** |
|---|---|---|
| **Cache bugs** | B1/B2 fixed — 95-99% cache hit in stable sessions | No cache bugs (predates them), but also no fixes for future regressions |
| **System prompt** | Includes "Output efficiency" instructions (v2.1.64+) — model may take shortcuts | Pre-"Output efficiency" — multiple users report deeper analysis and better code quality |
| **Features** | Full access: hooks, skills, plugins, 1M context, Agent SDK | No hooks, no skills, limited plugin support. 1M context may not be available |
| **Token cost** | Higher baseline (~280K avg context per @wpank) but cache offsets most of it | Lower baseline (~68K avg context per @wpank) — 3.3x cheaper per request in head-to-head test |
| **New bugs** | Exposed to B3-B11. Workarounds exist for most | Not exposed to B8a-B11, but also no protection against server-side issues (B4, B5, server quota) |
| **Best for** | Users who need hooks/skills/plugins, or who use the cache-fix interceptor | Users who prioritize code quality and cost efficiency over latest features |

**Independent confirmations for v2.1.63 downgrade:**
- @wpank: 47,810 requests tracked — 2.2x more output per dollar, 68% cheaper per request (caveat: session length mismatch in comparison)
- @janstenpickle: "1.5 hours with latest version to go nowhere and 5 minutes with downgraded version"
- @diabelko: "token usage dropped to 1-3% of limits"
- @breno-ribeiro706: "better results but still faster token consumption than a month ago" — **server-side issues persist regardless of CLI version**

---

## Option A: Stay Current (v2.1.91+)

### npm Installation (strongly recommended)

```bash
# Install via npm
npm install -g @anthropic-ai/claude-code

# Verify
claude --version   # should show 2.1.91 or later
file $(which claude)   # should show symbolic link to cli.js
```

### npm vs Standalone — Which to Choose?

The standalone binary is a bundled Bun executable. The Sentinel cache bug (B1) is fixed in v2.1.91, so both installations perform comparably out of the box. The difference is in **what community fixes you can apply**.

Community tools use three interception approaches, each with different compatibility:

| Approach | npm | Standalone | Representative Tools |
|----------|-----|-----------|---------------------|
| **`ANTHROPIC_BASE_URL` proxy** | **Works** | **Works** | [X-Ray Interceptor](https://github.com/Renvect/X-Ray-Claude-Code-Interceptor), [Pruner](https://github.com/OneGoToAI/Pruner), [cc-trace](https://github.com/alexfazio/cc-trace) (mitmproxy) |
| **Claude Code Hooks API** (settings.json) | **Works** | **Works** | [OpenWolf](https://github.com/cytostack/openwolf) (★363), cc-trace (Go/Shell hooks) |
| **`NODE_OPTIONS --import`** preload | **Works** | **Does not work** | [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) |

**The key trade-off:** cnighswonger's cache-fix interceptor — which normalizes block positions, sorts tool definitions, and strips image carry-forward — is the most impactful single fix (98.3% cache hit). It requires `NODE_OPTIONS`, so it is **npm-only**. If you want this specific fix, you need npm.

Proxy-based tools and Hooks API tools (including OpenWolf) work with both installations. If you are on standalone and don't need the cache-fix interceptor, you can stay.

| Metric | npm (v2.1.91) | Standalone (v2.1.91) |
|--------|--------------|---------------------|
| Cache cold start | 84.5% | 27.8-84.7% (varies) |
| Cache stable session | 98-99.6% | 94-99% |
| Community fix support | All 3 approaches | Proxy + Hooks only |

### Pin Your Version

```bash
# Disable auto-update (add to ~/.claude/settings.json)
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

### Recommended Environment Variables

```bash
# Add to ~/.claude/settings.json "env" section:

# Disable adaptive thinking zero-reasoning bug (B11, Anthropic acknowledged)
"CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING": "1"

# Optional: set effort to high (default medium=85 may under-allocate reasoning)
# Alternative: use /effort high or /effort max per-session
```

### Optional: Cache Fix Interceptor

[@cnighswonger](https://github.com/cnighswonger)'s [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) patches three remaining cache-busting issues that v2.1.91 did NOT fix:

1. **Attachment block scatter** — skills/MCP/hooks blocks drift to wrong positions on resume (52-73% of calls affected)
2. **Non-deterministic tool ordering** — tool definitions arrive in random order (98% of calls affected)
3. **Image carry-forward** — base64 images from Read tool persist in all subsequent calls (up to 266K tokens/turn)

```bash
npm install -g claude-code-cache-fix

# Add to ~/.claude/settings.json:
{
  "env": {
    "NODE_OPTIONS": "--import claude-code-cache-fix"
  }
}
```

Reported result: **98.3% cache hit rate** on resumed sessions (4,700 calls over 8 days). The interceptor is read-only — it normalizes request structure before it hits the API, without modifying your conversation.

---

## Option B: Downgrade to v2.1.63

v2.1.63 (February 28, 2026) is the last version before the "Output efficiency" system prompt change (v2.1.64, March 3). Multiple users independently report better code quality and lower token consumption.

### Install

```bash
# Direct install of specific version
curl -fsSL https://claude.ai/install.sh | bash -s 2.1.63

# Verify
claude --version   # should show 2.1.63
```

### Pin to Prevent Auto-Update

```bash
# Same as Option A — disable auto-updater
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

### What You Lose

- **Hooks** — `PreToolUse`, `PostToolUse`, `UserSubmitPrompt` hooks are not available
- **Skills** — plugin skills and `/skill` commands are not available
- **Some MCP features** — `maxResultSizeChars` override (v2.1.91) is not available
- **Cache fixes** — B1/B2 fixes are not present (but v2.1.63 predates the bugs that B1/B2 fix, so this is not a regression)
- **Agent SDK features** — newer orchestrator features may not work

### What You Keep

- Core Claude Code functionality (Read, Edit, Write, Bash, Grep, Glob, Agent)
- API compatibility (the Messages API is backward compatible)
- CLAUDE.md and memory files
- MCP server connections (basic)

### Important Caveat

**Server-side issues persist regardless of CLI version.** @breno-ribeiro706 confirmed: "still faster than a month ago." Bugs B4 (microcompact), B5 (budget cap), and server-side quota changes are controlled server-side via GrowthBook flags — no client downgrade can fix them. The improvement from downgrading is real but partial.

---

## Behaviors to Avoid (Both Options)

| Behavior | Why | Impact |
|----------|-----|--------|
| `--resume` / `--continue` | Replays entire conversation history as billable input | 500K+ tokens per resume. Start fresh sessions instead |
| `/dream`, `/insights` | Background API calls consume tokens without visible output | Silent drain |
| `/branch` | Duplicates or un-compacts message history (B9) | Can inflate 6% → 73% context in one message |
| `/release-notes` | Injects full changelog (~40-60K tokens) into context for rest of session | Silent context bloat |
| Multiple terminals simultaneously | No cache sharing between terminals, parallel quota drain | Limit to one active terminal |
| Large CLAUDE.md / many plugins | Sent on every turn, compounds with tool definitions | Keep CLAUDE.md lean. 47 skills = ~70K overhead/turn |

## Behaviors to Adopt

| Behavior | Why | Benefit |
|----------|-----|---------|
| Start fresh sessions often | The 200K tool result cap (B5) silently truncates older results | Clean context = better model performance |
| `/effort high` or `/effort max` | Default medium (effort=85) can under-allocate reasoning (B11) | Better output quality, reduces retry waste |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` | Prevents zero-reasoning turns that produce fabrications (B11) | Anthropic acknowledged this bug |
| Monitor `/usage` and `/context` | Catch abnormal drain early | Know when to restart |
| Pin your CLI version | Future updates may reintroduce regressions | Stability |

---

## Self-Diagnosis

### Quick Check: Are You Affected?

```
Symptom                                          → Likely Cause
─────────────────────────────────────────────────────────────────
"Rate limit reached" immediately, 0% usage       → B3 (false rate limiter) or server billing bug
Session drains 50%+ in <30 min                   → Multiple bugs compounding, check cache ratio
Model takes shortcuts, "simplest fix"             → B11 (adaptive thinking) + Output efficiency prompt
Tool results seem missing mid-session             → B4/B5 (microcompact + budget cap)
--resume costs massive tokens                     → B2/B2a (cache miss on resume)
Session won't load, 400 error                     → B8a (JSONL corruption from concurrent tools)
/branch immediately fills context                 → B9 (/branch inflation)
```

### Check Your Cache Ratio (Session JSONL)

Session files in `~/.claude/projects/` contain usage data for each turn:

```bash
# Find your latest session
ls -lt ~/.claude/projects/*/sessions/ | head -5

# Quick cache ratio check (using jq)
cat <session-file>.jsonl | jq -r '
  select(.type == "assistant" and .costInfo) |
  "\(.costInfo.cacheReadInputTokens // 0) / \(.costInfo.cacheCreationInputTokens // 0)"
' | head -20
```

- **Healthy session:** `cache_read` >> `cache_creation` (read ratio > 80%)
- **Affected session:** `cache_creation` >> `cache_read` (read ratio < 40%)

If most sessions show low read ratios on v2.1.91+, the cache-fix interceptor (Option A) or a downgrade (Option B) may help.

### Optional: Monitor via Proxy

```bash
# Route through local proxy for real-time monitoring
ANTHROPIC_BASE_URL=http://localhost:8080 claude

# Parse cache_creation_input_tokens / cache_read_input_tokens from responses
# Healthy: read ratio > 80%  |  Affected: read ratio < 40%
```

Community monitoring tools:
- [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) — includes `CACHE_FIX_DEBUG=1` mode for per-request logging
- [cc-trace](https://github.com/alexfazio/cc-trace) — mitmproxy-based full traffic analysis (★152)
- [BudMon](https://github.com/weilhalt/budmon) — real-time rate-limit header monitoring
- [claude-usage-dashboard](https://github.com/fgrosswig/claude-usage-dashboard) — JSONL dashboard with forensic analysis
- [CUStats](https://custats.info) — real-time usage tracking

---

*See [01_BUGS.md](01_BUGS.md) for technical root causes, [04_BENCHMARK.md](04_BENCHMARK.md) for npm vs standalone benchmarks, [12_ADVANCED-GUIDE.md](12_ADVANCED-GUIDE.md) for advanced configuration.*
