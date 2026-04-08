> **🇰🇷 [한국어 버전](ko/11_USAGE-GUIDE.md)**

# Claude Code Usage Guide — Essential Practices

> **TL;DR:**
> - Update to v2.1.91+, disable auto-updates, and start fresh sessions for each task.
> - Keep CLAUDE.md under 200 lines — it is sent with every single API call.
> - After 15-20 file reads in one session, older tool results are silently truncated. Start a new session before that happens.
>
> **Audience:** Anyone using Claude Code — developers and non-developers alike.
> **Baseline:** v2.1.91 or later. See [09_QUICKSTART.md](09_QUICKSTART.md) for installation and self-diagnosis.

---

## 1. Version & Setup

| Step | What to Do |
|------|-----------|
| **Check your version** | Run `claude --version` in your terminal. You want **2.1.91** or later. |
| **Update if needed** | `claude update` (standalone) or `npm install -g @anthropic-ai/claude-code` (npm) |
| **Disable auto-updates** | Add `"DISABLE_AUTOUPDATER": "1"` to `~/.claude/settings.json` under the `"env"` section (see below) |
| **Verify installation type** | `file $(which claude)` — shows "symbolic link" (npm) or "ELF 64-bit" (standalone) |

**Why pin your version?** Auto-updates can introduce regressions. v2.1.89 had a cache bug that caused 4-17% cache efficiency (should be 95%+), meaning every token was billed at near-full price. Pinning lets you update on your own schedule after others have tested new releases.

**Settings file example** (`~/.claude/settings.json`):
```json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

**npm vs standalone:** On v2.1.91+, both work equally well. Benchmarks show they converge to 94-99% cache efficiency once warmed (measured data in [04_BENCHMARK.md](04_BENCHMARK.md)). Use whichever you prefer.

---

## 2. Session Management

A "session" is one continuous conversation with Claude Code. Every time you run `claude` in your terminal, that is a new session. The `/clear` command also starts a fresh context within the same terminal.

### Start Fresh Sessions Often

This is the single most impactful habit you can build. Fresh sessions are cheap — stale sessions get bloated and lose information silently.

**When to start a new session:**

| Situation | Why |
|-----------|-----|
| Switching to a different task or project | Accumulated context from task A is irrelevant noise for task B |
| After 15-20 file reads | Older tool results get silently truncated after the 200K character aggregate cap (Bug 5) |
| After 30+ tool uses total | Context quality degrades as more results are cleared |
| When Claude seems to "forget" what it read earlier | This is likely silent truncation, not forgetfulness |
| After a compaction event (you see a summary message) | Post-compaction, Claude only has the summary — not the raw data |

### Avoid --resume and --continue

Both `--resume` and `--continue` replay the entire conversation history as billable input. On a 500K token conversation, a single resume costs the equivalent of processing all that context again. The cache fix in v2.1.90+ helps, but the fundamental cost remains high.

**Instead:** Copy any critical context (file paths, decisions, error messages) into your new session's first message. This is faster and cheaper than resuming.

### Understanding Compaction

When a conversation gets long, Claude Code compresses older context to stay within limits. There are two types:

**Manual compaction** (`/compact`): You trigger this yourself. You can provide a custom summary focus — for example, `/compact focus on the database migration we discussed`. Useful when you want to keep working in the same session but shed irrelevant context.

**Automatic compaction:** Three separate mechanisms run silently on every API call, replacing old tool outputs with placeholder text like `[Old tool result content cleared]`. These run regardless of any environment variable settings — even `DISABLE_AUTO_COMPACT=true` does not stop all of them (see [05_MICROCOMPACT.md](05_MICROCOMPACT.md) for technical details).

**After any compaction:** Claude can no longer see the original file contents, command outputs, or search results from earlier in the session. It only sees the summary or placeholder. If you need Claude to reference specific earlier content, you will need to re-read those files.

---

## 3. Understanding Context (Why Claude "Gets Dumb")

### What Is the Context Window?

Everything Claude can "see" in one conversation: your messages, its responses, file contents it read, command outputs, search results, and system instructions. Claude Code supports up to 1M tokens, but the effective usable context is much less due to silent truncation mechanisms.

### Token Intuition

Tokens are the unit of measurement for how much text Claude processes. You do not need to count them precisely, but a rough sense helps:

| Content | Approximate Token Count |
|---------|------------------------|
| 1 English word | ~1.3 tokens |
| 1 Korean character | ~1 token |
| A typical source file (200 lines) | 2,000-5,000 tokens |
| A large CLAUDE.md (500 lines) | 5,000-10,000 tokens |
| A full conversation (100 turns) | 300,000-600,000 tokens |

### Why Claude Suddenly Seems Worse

The model is not getting dumber — it is literally losing access to information it read earlier.

**What happens:** After approximately 15-20 file reads in a single session, the total tool result text exceeds a 200K character aggregate cap. At that point, older tool results are silently truncated to as few as 1-41 characters. Claude can no longer see those file contents, grep results, or command outputs — only a tiny stub remains.

**Measured example:** In one tested session, 261 budget truncation events were detected. Tool results that originally contained thousands of characters were reduced to single-digit lengths. The threshold was crossed at 242,094 total characters of tool output (see [01_BUGS.md](01_BUGS.md#bug-5--tool-result-budget-enforcement-all-versions)).

**What this looks like to you:**
- Claude contradicts something it said earlier (it can no longer see the data it based that statement on)
- Claude re-reads files it already read (it does not remember the earlier result)
- Claude suggests approaches it already tried (it cannot see that they failed)
- Claude's answers become more generic and less grounded in your codebase

### What You Can Do

| Action | Effect |
|--------|--------|
| Start fresh sessions every 15-20 file reads | Resets the tool result budget |
| Read only the parts of files you need | Use line ranges: "Read lines 50-80 of server.py" instead of the whole file |
| Be specific in your requests | "Fix the login validation in auth.py line 42" reads fewer files than "Fix my auth system" |
| Keep CLAUDE.md lean | Reduces per-turn overhead (see Section 4) |

---

## 4. CLAUDE.md Best Practices

### What CLAUDE.md Does

CLAUDE.md is an instruction file that Claude Code reads at the start of every conversation turn. Think of it as a briefing document — Claude sees it before processing each of your messages. It is the right place for project-specific rules, conventions, and constraints.

### The Cost

CLAUDE.md content is sent with every API call as cached input. A 10,000-token CLAUDE.md costs roughly 10,000 tokens of `cache_read` per turn. Over a 50-turn session, that is 500,000 tokens spent just on instructions. Our measured data shows `cache_read` accounts for 88-98% of all visible token volume ([03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md)) — and CLAUDE.md is a fixed part of every cache read.

### File Hierarchy

Claude Code loads all matching CLAUDE.md files and concatenates them:

| Location | Scope | Example |
|----------|-------|---------|
| `~/.claude/CLAUDE.md` | Global — applies to all projects | Personal coding preferences |
| `<project-root>/CLAUDE.md` | Project-specific | Project conventions, tech stack |
| `<project-root>/.claude/CLAUDE.md` | Also project-specific | Alternative location |

All matching files are loaded together. If you have a global CLAUDE.md (500 tokens) and a project CLAUDE.md (2,000 tokens), every turn sends 2,500 tokens of instructions.

### Writing a Good CLAUDE.md

**DO:**
- Keep it under 200 lines / 2,000 tokens
- Focus on project-specific rules and constraints
- Use bullet points and short phrases
- Reference external documents instead of inlining them
- Include only what Claude needs to know for THIS project

**DON'T:**
- Paste entire API documentation or library guides
- Include lengthy code examples or tutorials
- Duplicate information already present in the codebase
- Add generic coding advice (Claude already knows how to code)
- Include personal notes, planning documents, or task lists

### Examples

**Good (compact, actionable — ~150 tokens):**
```markdown
# Project Rules
- Python 3.12, ruff for linting, pytest for tests
- Never hardcode secrets — use os.getenv() with safe defaults
- PostgreSQL: parameterized queries only, no string concatenation
- Run tests: pytest tests/ -x
- API responses: always return {data, error, timestamp}
```

**Bad (bloated, generic — ~5,000+ tokens):**
```markdown
# Complete Python Development Guide
## What is Python?
Python is a high-level programming language...
## Setting Up Your Environment
First, install Python from python.org...
## Best Practices for Clean Code
1. Use meaningful variable names...
[400 more lines of generic content]
```

The good example gives Claude exactly the constraints it needs in 5 lines. The bad example wastes thousands of tokens on information Claude already has.

---

## 5. Token-Saving Habits

Small habits compound across hundreds of turns per day. Each row below is a concrete change you can make today.

| Habit | Why It Helps | How to Do It |
|-------|-------------|--------------|
| **Read file ranges, not whole files** | A 1,000-line file = ~5,000 tokens. Lines 50-80 = ~200 tokens. | Ask: "Read lines 50-80 of server.py" |
| **One active terminal** | Multiple Claude Code sessions do not share cache — each builds and pays for its own. | Close unused Claude sessions before starting a new one. |
| **Use specific requests** | Vague requests cause Claude to read many files searching for context. | "Fix the null check in auth.py:42" > "Fix my auth system" |
| **Lean CLAUDE.md** | Sent every turn. 5,000 extra tokens x 50 turns = 250,000 wasted tokens. | Keep under 200 lines. Reference docs instead of inlining. |
| **Fresh sessions for new tasks** | Old sessions carry irrelevant context and may have truncated tool results. | Close and restart when switching tasks. |
| **Avoid background commands** | `/dream` and `/insights` make background API calls that consume quota silently. | Simply do not use them. |
| **State your goal upfront** | Lets Claude plan before acting, reducing unnecessary file reads. | Start with "I need to add rate limiting to the /api/users endpoint" not "Show me the API code." |

---

## 6. Behaviors to Avoid

| Behavior | What Happens | What to Do Instead |
|----------|-------------|-------------------|
| `--resume` / `--continue` | Replays entire conversation history as billable input. A 500K-token conversation costs ~500K tokens just to resume. | Start a fresh session. Paste key context in your first message. |
| `/dream`, `/insights` | Background API calls that drain quota without visible output. | Do not use these commands. |
| Multiple terminals simultaneously | Each terminal builds its own cache independently. No sharing, parallel quota drain. | Use one terminal at a time. |
| Huge CLAUDE.md files (500+ lines) | 5,000-10,000+ tokens sent on every single turn. In a 50-turn session, that is 250K-500K tokens on instructions alone. | Keep under 200 lines. Move details to referenced docs. |
| "Read the entire codebase" | Claude reads dozens of files, hits the 200K budget cap fast, and older results get truncated. | Be specific: "Read the auth module" or "Find where rate limiting is implemented." |
| Running an old version | Versions before v2.1.91 have cache bugs that cause 4-17% efficiency (vs. 95%+ normal). Every token costs 5-25x more. | Run `claude --version` and update to v2.1.91+. |
| Long sessions without breaks | After 50+ tool uses, effective context drops to ~40-80K despite the 1M window. Claude is working with 4-8% of what you think it sees. | Start a new session every 15-20 file reads or 30-50 tool uses. |

---

## 7. Self-Diagnosis

### Quick Health Check

Run these three checks to verify your setup is healthy:

```bash
# 1. Version — should be 2.1.91 or later
claude --version

# 2. Installation type — "symbolic link" (npm) or "ELF 64-bit" (standalone)
file $(which claude)

# 3. Active terminals — only one Claude Code session should be active at a time
# Check if other terminals have claude running
ps aux | grep -c '[c]laude'
```

### Reading Your Session Logs

Session JSONL files live at `~/.claude/projects/**/*.jsonl`. Each API call is logged as a JSON entry containing token usage. The two key fields:

| Field | Meaning | What You Want |
|-------|---------|---------------|
| `cache_read_input_tokens` | Tokens reused from the server's prompt cache | **High** (cheap — discounted) |
| `cache_creation_input_tokens` | Tokens sent fresh, creating a new cache entry | **Low** (expensive — full price) |

**How to interpret:**

| Cache Read Ratio | Meaning | Action |
|-----------------|---------|--------|
| **> 80%** | Healthy. Cache is working as intended. | None needed. |
| **40-80%** | Moderate. Possible cold starts or session churn. | Normal for session beginnings — check if it stabilizes. |
| **< 40%** | Problem. Cache is not being reused. | Check your version (should be 2.1.91+). Check for multiple terminals. |

**Note on JSONL accuracy:** Local JSONL logs show approximately 1.93x the actual token usage due to PRELIM entry duplication (Bug 8). Do not rely on local logs for precise cost estimates. The ratio is useful for comparing sessions, but the absolute numbers are inflated. See [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) for the cross-validation data.

### Community Monitoring Tools

Several community members have built tools that make monitoring easier:

| Tool | What It Does | Link |
|------|-------------|------|
| **CUStats** | Web-based visual session statistics and usage tracking | [custats.info](https://custats.info) |
| **ccdiag** | Go-based CLI tool for JSONL session log recovery and DAG analysis | [github.com/kolkov/ccdiag](https://github.com/kolkov/ccdiag) |
| **context-stats** | Per-interaction cache metrics export and analysis | [github.com/luongnv89/cc-context-stats](https://github.com/luongnv89/cc-context-stats) |
| **BudMon** | Desktop dashboard for real-time rate-limit header monitoring | [github.com/weilhalt/budmon](https://github.com/weilhalt/budmon) |
| **claude-usage-dashboard** | Standalone Node.js dashboard with forensic analysis and multi-host aggregation | [github.com/fgrosswig/claude-usagage-dashboard](https://github.com/fgrosswig/claude-usagage-dashboard) |
| **tokenlean** | 54 CLI tools that extract symbols/snippets instead of full file reads, reducing token waste | [github.com/edimuj/tokenlean](https://github.com/edimuj/tokenlean) |

For the full list of community tools and analysis resources, see [10_ISSUES.md](10_ISSUES.md#community-analysis--tools).

---

## 8. Known Issues Quick Reference

Detailed technical analysis in [01_BUGS.md](01_BUGS.md). This table is a practical summary of what each bug means for your daily usage.

| Bug | What Happens | Status | Your Action |
|-----|-------------|--------|-------------|
| **B1** Sentinel | Standalone binary breaks prompt cache — 4-17% efficiency instead of 95%+ | **Fixed** (v2.1.91) | Update to v2.1.91+. |
| **B2** Resume | `--resume` causes full cache miss, rebilling entire conversation | **Fixed** (v2.1.90) | Avoid `--resume` and `--continue`. Start fresh. |
| **B3** False Rate Limit | Client generates fake "Rate limit reached" errors without contacting the API. 24 synthetic events measured across 6 days. | **Unfixed** | Restart Claude Code if you hit a rate limit immediately after idle time. |
| **B4** Microcompact | Old tool results silently replaced with `[Old tool result content cleared]`. 327 events measured. Cache stays at 99%+, but the model loses access to earlier data. | **Unfixed** | Start fresh sessions periodically. No env var prevents this. |
| **B5** Budget Cap | After 200K aggregate characters of tool results, older results are truncated to 1-41 characters. 261 events measured in one session. | **Unfixed** | Start fresh sessions after 15-20 file reads. Read only the lines you need. |
| **B8** Log Inflation | Extended thinking causes 2-3x duplicate entries in JSONL logs, inflating local token counts. JSONL shows 1.93x actual usage. | **Unfixed** | Do not trust local JSONL token counts as absolute values. Use them for relative comparisons only. |

### Server-Side Factors

Beyond client bugs, server-side changes also affect your experience:

- **Dual quota windows:** Your usage is tracked in both a 5-hour and a 7-day sliding window. The 5-hour window is almost always the bottleneck ([02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)).
- **Thinking tokens:** Extended thinking tokens do not appear in the `output_tokens` API field, but likely count against your quota. Visible output explains less than half the observed utilization.
- **Organization sharing:** Accounts under the same organization share rate limit pools. If a colleague is using Claude Code heavily, it reduces your available quota too.

---

## Summary: The Five-Minute Checklist

If you remember nothing else from this guide, do these five things:

1. **Version 2.1.91+** — Run `claude --version`. Update if needed.
2. **Auto-update off** — Set `DISABLE_AUTOUPDATER=1` in settings.json.
3. **Fresh sessions** — New task = new session. Do not use `--resume`.
4. **Lean CLAUDE.md** — Under 200 lines. Reference docs, do not inline them.
5. **Be specific** — Tell Claude exactly what you need. Fewer files read = more effective session.

---

*For advanced topics (hooks, subagents, proxy monitoring, multi-project workflows), see [12_ADVANCED-GUIDE.md](12_ADVANCED-GUIDE.md).*
*For setup and installation instructions, see [09_QUICKSTART.md](09_QUICKSTART.md).*
*For the full technical bug analysis, see [01_BUGS.md](01_BUGS.md).*
*For community tools and related issues, see [10_ISSUES.md](10_ISSUES.md).*
