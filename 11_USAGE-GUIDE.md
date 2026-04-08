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

This is the single most impactful habit you can build. Think of sessions like browser tabs: fresh ones are cheap and fast, but stale ones accumulate invisible debt — bloated context, truncated tool results, and degraded recall — even when every metric looks healthy.

**In practice:** In one real workflow managing a data pipeline project, a single session that lasted 150+ turns maintained a cache ratio at 99%. The numbers looked perfectly healthy. But Claude could not accurately reference any file it had read before turn 40. It would paraphrase earlier findings incorrectly, miss key details from files it had "seen," and suggest approaches that contradicted its own earlier analysis. The metrics said everything was fine; the experience said otherwise.

**When to start a new session:**

| Situation | Why |
|-----------|-----|
| Switching to a different task or project | Accumulated context from task A is irrelevant noise for task B |
| After 15-20 file reads | Older tool results get silently truncated after the 200K character aggregate cap (Bug 5) |
| After 30+ tool uses total | Context quality degrades as more results are cleared |
| When Claude seems to "forget" what it read earlier | This is likely silent truncation, not forgetfulness |
| After a compaction event (you see a summary message) | Post-compaction, Claude only has the summary — not the raw data |

### The Session Rotation Pattern

After completing each self-contained task — fixing a bug, adding a feature, reviewing code — close the session and start a new one. Before closing, copy 2-3 key context lines into a scratch note. Then paste them into your new session's first message. For example:

```
Working on src/api/auth.py — added rate limiting in validate_token().
Tests in tests/test_auth.py pass. Next: wire up the /logout endpoint.
```

This costs a few seconds but gives Claude a clean context window with exactly the information it needs. In a multi-project workflow, this pattern is the difference between sessions that stay sharp for hours and sessions that silently degrade after the first 20 minutes.

### Avoid --resume and --continue

Both `--resume` and `--continue` replay the entire conversation history as billable input. On a 500K token conversation, a single resume costs the equivalent of processing all that context again. The cache fix in v2.1.90+ helps, but the fundamental cost remains high.

**Instead:** Copy any critical context (file paths, decisions, error messages) into your new session's first message. This is faster and cheaper than resuming.

### Understanding Compaction

When a conversation gets long, Claude Code compresses older context to stay within limits. There are two types:

**Manual compaction** (`/compact`): You trigger this yourself. You can provide a custom summary focus — for example, `/compact focus on the database migration we discussed`.

**Automatic compaction:** Three separate mechanisms run silently on every API call, replacing old tool outputs with placeholder text like `[Old tool result content cleared]`. These run regardless of any environment variable settings — even `DISABLE_AUTO_COMPACT=true` does not stop all of them (see [05_MICROCOMPACT.md](05_MICROCOMPACT.md) for technical details).

**After any compaction:** Claude knows WHAT it discussed (from the summary) but not the exact code, exact error messages, or exact line numbers. If you ask "what was the error on line 42?" after compaction, it will either hallucinate a plausible-sounding answer or admit it does not know — and the hallucination case is more dangerous because it looks confident. If you need Claude to reference specific earlier content, you will need to re-read those files.

### Why /compact Is Not Recommended

In practice, `/compact` loses far more detail than you expect. The compression is lossy — specific variable names, exact error messages, line numbers, and the reasoning behind decisions are routinely dropped. After compaction, Claude retains a vague sense of "what was discussed" but loses the precision needed to continue work accurately. This leads to subtle bugs: Claude confidently references code that no longer matches reality, or suggests approaches that were already ruled out in the pre-compaction conversation.

**The better alternative: document-based session handoff.** Instead of compressing a conversation into an unreliable summary, explicitly update your working documents (PLAN files, CLAUDE.md, error logs) at the end of each session — while you can still verify the content against the full conversation context. Then start a fresh session that reads those documents.

This "document cross-validation" pattern works because:
- **You control what's preserved.** You decide which details matter, not an automated summarizer.
- **Updates are verified.** You can ask Claude to cross-check the document updates against the actual conversation before closing the session.
- **New sessions start clean.** The fresh session reads authoritative, human-verified documents rather than a lossy AI summary.
- **Knowledge compounds across sessions.** Each session's learnings are captured in durable files, not trapped in a conversation that will eventually be compacted or lost.

In a workflow managing multiple projects over months, this pattern consistently outperforms `/compact`. Sessions that start by reading updated documents are more focused and accurate than sessions that rely on compacted conversation history — because the documents contain exactly what you chose to preserve, with nothing silently dropped.

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

**In practice:** During a code review session reading 25 files across 3 modules, the tool result budget silently kicked in around file 18. Claude started giving generic suggestions instead of file-specific feedback — because it could no longer see the contents of the first 12 files it had read. The review went from "this null check on line 73 conflicts with the guard clause on line 31 of handler.py" to "consider adding error handling to your functions." The quality drop was abrupt, not gradual.

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

**Good (compact, actionable — ~30 lines covering a multi-project setup):**
```markdown
# Project Rules
- Python 3.12, ruff for linting, pytest for tests
- Never hardcode secrets — use os.getenv() with safe defaults
- PostgreSQL: parameterized queries only, no string concatenation
- Run tests: pytest tests/ -x
- API responses: always return {data, error, timestamp}

# Git
- Never commit CLAUDE.md or PLAN.txt
- Commit message: imperative mood, <72 chars
- git email: user@users.noreply.github.com

# Key Paths
- API entry: src/api/main.py
- Pipeline config: pipeline/config.yaml
- Error log: docs/error-log.md (read on demand, not here)

# Security
- .env patterns only. No hardcoded keys.
- DELETE/DROP queries: always confirm with user first
```

This gives Claude the tech stack, security policy, git conventions, and key file pointers — all in under 30 lines. Everything Claude needs to work correctly on the project, nothing it does not.

**Bad (organically grown — 800+ lines):**
```markdown
# Complete Project Reference
## Architecture
[ASCII diagram of 6 services, 40 lines]
## Full API Reference
### GET /api/v1/users
Returns a list of users. Parameters: limit (int), offset (int)...
### POST /api/v1/users
Creates a new user. Body: {name: string, email: string}...
[200 more lines of API docs]
## Meeting Notes (2025-03)
- Decided to switch from MySQL to PostgreSQL
- Sarah will handle the migration script
## Current Sprint Tasks
- [ ] Fix login bug (#423)
- [ ] Add rate limiting
[150 more lines of planning notes and task lists]
```

In practice, one configuration that grew like this over months reached 800+ lines. It included a full API reference, architecture diagrams in ASCII, meeting notes, and a task backlog. Every single API call sent all 800 lines. Trimming it to 150 lines had zero impact on Claude's code quality but reduced per-turn overhead by 6,000+ tokens — tokens that were being billed on every turn for months.

### Companion Files: MEMORY.md and PLAN Files

**MEMORY.md:** For information that changes frequently — project status, current tasks, feedback from past sessions — use MEMORY.md instead of CLAUDE.md. MEMORY.md also loads every turn, but it is auto-managed and easier to keep lean. The key practice: use it as an **index of pointers** rather than dumping everything into one file. Each entry is a one-line reference to a separate file that Claude can read on demand. This keeps the per-turn cost low while making detailed context available when needed.

**PLAN files:** Keep planning documents (current task, approach decisions, error logs) in separate files like PLAN.txt rather than in CLAUDE.md. Claude can read them on demand, and they do not cost tokens every turn. Do not commit these to version control — they are working documents that change constantly and have no value in git history.

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
| **Never say "explore the codebase"** | Triggers dozens of file reads that burn through the tool result budget. | Point Claude at the 2-3 specific files relevant to your task. |

**The hidden cost of vague exploration:** One of the biggest unnoticed drains is asking Claude to "look at the codebase" or "understand the project structure." In one measured session, a vague "explore the API" request caused 34 file reads in 8 minutes — consuming the entire tool result budget. By the time the actual task started, every file read from the exploration phase had already been truncated. The exploration itself became invisible, and Claude had to re-read the files it needed. The net result: double the token cost, zero benefit.

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
| Growing CLAUDE.md without pruning | Instruction files that grow organically over months — adding rules but never removing outdated ones. Old rules accumulate, contradict each other, and inflate every turn. | Review and prune quarterly. In one case, reducing a 600-line file to 120 lines had no loss of Claude's effectiveness — most of the removed content was stale or redundant. |

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

### The Most Useful Diagnostic

Beyond metrics, there is one behavioral signal that is more reliable than any log file: **if Claude starts suggesting approaches you already tried earlier in the session, or cannot quote specific lines from files it read 10+ turns ago, your session is past its useful life.** This is the earliest detectable sign of context degradation. Do not troubleshoot it — just start fresh. The new session will cost far less than the circular turns you would spend in the degraded one.

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
