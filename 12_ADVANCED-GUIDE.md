> **🇰🇷 [한국어 버전](ko/12_ADVANCED-GUIDE.md)**

# Claude Code Advanced Guide — Power User Practices

> **TL;DR:** Master CLAUDE.md architecture to control cost and context quality. Use proxy monitoring to see the quota numbers Claude Code hides. Automate guardrails with hooks and multi-project isolation to scale across codebases.
>
> **Audience:** Agent builders, multi-project operators, teams.
> **Prerequisite:** [09_QUICKSTART.md](09_QUICKSTART.md) for setup. The bug analysis in [01_BUGS.md](01_BUGS.md) and quota architecture in [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) for technical context.

---

## 1. CLAUDE.md Architecture

Claude Code loads instruction files at every turn and concatenates them into the system prompt. Understanding this architecture is key to controlling both behavior and cost.

### Three-tier hierarchy

| Tier | Location | Scope | Loaded when |
|------|----------|-------|-------------|
| Global | `~/.claude/CLAUDE.md` | All projects, all sessions | Always |
| Project (root) | `<project>/CLAUDE.md` | This project only | When working in `<project>/` |
| Project (dotdir) | `<project>/.claude/CLAUDE.md` | This project only (alternative) | When working in `<project>/` |

All tiers are concatenated and sent as part of the system prompt on **every API call**. There is no lazy loading or conditional inclusion.

### Design principles

- **Global tier:** Only truly universal rules. Coding style, git conventions, security policies. Target: **under 50 lines**.
- **Project tier:** What makes this codebase unique. Tech stack, key file paths, naming conventions. Target: **under 150 lines**.
- **Combined target:** Under 200 lines across all tiers.
- **Write for the model, not for humans.** Concise imperatives work better than explanatory prose. "Use `os.getenv()` for secrets, never hardcode" beats a paragraph about why environment variables matter.

### Cost analysis

A 200-line CLAUDE.md is roughly 2,000 tokens. At typical cache rates (96-99% `cache_read`), this costs cache-read price per turn — cheap individually, but it compounds:

| CLAUDE.md size | Tokens | Per-turn cost at cache rate | Risk |
|---------------|--------|---------------------------|------|
| 50 lines | ~500 | Negligible | None |
| 200 lines | ~2,000 | Low | None |
| 500 lines | ~5,000 | Moderate | Cache-miss risk increases |
| 1,000+ lines | ~10,000+ | High | Diminishing returns, cache fragility |

The system prompt (including CLAUDE.md) forms the cache prefix. Larger prefixes mean more bytes to hash and match. Beyond ~500 lines, you increase the chance of cache misses from minor changes, and the instruction-following benefit plateaus.

### MEMORY.md pattern

Claude Code auto-manages a `MEMORY.md` file under `~/.claude/projects/<project-path>/memory/`. This persists across sessions.

**Good uses:**
- User preferences (language, git email, style)
- Project status tracking and pointers to detail files
- Feedback rules that apply across sessions

**Bad uses:**
- Code patterns or architecture docs (read the code instead — it's always fresher)
- Temporary state (use `/compact` summaries or working files)
- Large content blocks (MEMORY.md is loaded every turn, same as CLAUDE.md)

**Index pattern:** Keep MEMORY.md as an index of pointers to individual memory files. Each file covers one topic. This keeps the per-turn load small while allowing deep storage.

### Anti-patterns

| Anti-pattern | Why it hurts | Fix |
|-------------|-------------|-----|
| Pasting library docs into CLAUDE.md | Bloats every turn, model already knows popular libraries | Link to docs or let the model read them on demand |
| Including file trees that change often | Any change to the tree invalidates the cache prefix | Use glob patterns or key-file lists instead |
| Session-specific state in CLAUDE.md | Stale across sessions, pollutes global instructions | Use PLAN files or session-local notes |
| Duplicating rules across tiers | Wastes tokens, risks contradictions | Put each rule in exactly one tier |

---

## 2. Subagent Strategy

Claude Code can spawn independent AI instances ("subagents") to handle subtasks in parallel. Each subagent is a full Claude Code session with its own context window.

### Cost profile

| Phase | Cache ratio | Note |
|-------|------------|------|
| Cold start (1st request) | 54-80% | Full cache build — expensive |
| Warming (requests 2-5) | 80-94% | Gradually improves |
| Steady state (5+ requests) | 94-99% | Normal operating cost |

Each subagent builds its own cache independently. There is no cache sharing between parent and child sessions.

### When to use subagents

**Good candidates:**
- Searching multiple directories simultaneously (3-5 agents across different subtrees)
- Isolated investigations that shouldn't pollute the main session's context
- Long-running background tasks (code generation, test writing)
- Tasks where you need the result but not the intermediate reasoning

**Bad candidates:**
- Simple single-file operations (the cold start overhead exceeds the benefit)
- Tasks that require the main session's accumulated context (subagents start fresh)
- When you are near the rate limit (each subagent consumes from the same quota pool)
- Short tasks that finish in one turn (paying cold start for a single response)

### Context passing

Subagents read CLAUDE.md (paying the token cost per agent), but they do **not** inherit the parent session's conversation history. Effective patterns:

- **Explicit prompts:** Give the subagent the exact file paths, search terms, and expected output format. Don't assume it knows what you discussed earlier.
- **PLAN files:** For complex multi-step tasks, write a plan to a file and have the subagent read it. This costs one `Read` call instead of stuffing context into the prompt.
- **Scoped instructions:** If a subagent needs project-specific rules, those are already in CLAUDE.md. Don't repeat them in the prompt.

### Parallel dispatch

Launching 3-5 agents simultaneously is efficient for exploration tasks. Each warms its own cache independently. The total cost is 3-5x the cold start overhead, but the wall-clock time savings usually justify it.

Be aware that all agents in a session draw from the same rate limit quota. Five parallel agents drain five times faster than sequential work.

---

## 3. Hooks & Automation

Hooks are shell commands that run automatically when Claude Code uses tools. They enable guardrails, linting, and policy enforcement without manual intervention.

### Configuration

Hooks live in `~/.claude/settings.json` under the `"hooks"` key. Project-level overrides go in `<project>/.claude/settings.json`.

### Available hook points

| Hook | When it runs | Can block? | Use case |
|------|-------------|-----------|----------|
| `PreToolUse` | Before a tool executes | Yes (exit 1) | Block dangerous commands |
| `PostToolUse` | After a tool executes | No | Lint, validate, log |
| `Notification` | When Claude sends a notification | No | Custom alerting |

### Environment variables available in hooks

| Variable | Content |
|----------|---------|
| `CC_TOOL_INPUT` | JSON string of the tool's input parameters |
| `CC_TOOL_NAME` | Name of the tool being invoked |

### Production example: Auto-lint Python files after edit

This hook runs `ruff` on any Python file that Claude edits or writes, using the project's local virtual environment:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "filepath=$(echo \"$CC_TOOL_INPUT\" | grep -oP '\"file_path\"\\s*:\\s*\"\\K[^\"]+'); if [ -z \"$filepath\" ] || [ ! -f \"$filepath\" ]; then exit 0; fi; case \"$filepath\" in *.py) d=$(dirname \"$filepath\"); while [ \"$d\" != \"/\" ]; do if [ -f \"$d/.venv/bin/ruff\" ] && [ -f \"$d/pyproject.toml\" ]; then \"$d/.venv/bin/ruff\" check --no-fix \"$filepath\" 2>&1 | head -20; exit 0; fi; d=$(dirname \"$d\"); done ;; esac",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

Key design choices:
- Walks up the directory tree to find the nearest `.venv/bin/ruff` — works across multi-project setups
- Only triggers on `.py` files (the `case` statement)
- Exits cleanly (exit 0) for non-Python files — no false failures
- `--no-fix` reports issues without modifying the file
- `head -20` caps output to avoid flooding the context

### Example: Block dangerous commands

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$CC_TOOL_INPUT\" | grep -qiE 'rm\\s+-rf\\s+/|drop\\s+table|git\\s+push\\s+--force\\s+(origin\\s+)?main' && echo 'BLOCKED: Dangerous command detected' && exit 1 || exit 0",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

This blocks `rm -rf /`, `DROP TABLE`, and force pushes to main. Adjust the regex to your risk tolerance.

### Global environment variables

The `"env"` key in settings.json sets environment variables for every Claude Code session:

```json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1",
    "ANTHROPIC_BASE_URL": "http://localhost:8080"
  }
}
```

`DISABLE_AUTOUPDATER` prevents surprise version changes mid-work. `ANTHROPIC_BASE_URL` routes all API traffic through a local proxy (see Section 4).

### Hook design guidelines

- **Keep hooks fast.** The `timeout` field is in milliseconds. A 10-second timeout is generous for a linter; keep blocking hooks under 5 seconds.
- **Exit 0 for success, exit 1 to block.** Only `PreToolUse` hooks can block — `PostToolUse` exit codes are ignored.
- **Don't modify files in hooks.** Hooks that write to the filesystem can cause race conditions with Claude's own writes.
- **Test hooks in isolation first.** Run the command manually with sample `CC_TOOL_INPUT` before adding it to settings.json.

---

## 4. Monitoring & Proxy Setup

Claude Code's built-in usage bar shows a rough percentage but hides the actual quota mechanics. A transparent proxy reveals the full picture.

### Why monitor

- See exact `cache_creation_input_tokens` and `cache_read_input_tokens` per request
- Capture rate limit headers that Claude Code discards (it only reads `representative-claim`)
- Detect context mutations (microcompact, budget enforcement) before they hit the API
- Correlate visible token counts with utilization percentage

### ANTHROPIC_BASE_URL proxy

`ANTHROPIC_BASE_URL` is an official environment variable. Set it to route all API requests through a local proxy:

```bash
# In settings.json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:8080"
  }
}
```

The proxy logs every request and response without modifying them. It captures headers, token usage, and request bodies.

### Rate limit headers

Every API response includes `anthropic-ratelimit-unified-*` headers (from [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)):

| Header | Meaning | Example |
|--------|---------|---------|
| `5h-utilization` | Current 5-hour window usage (0.0-1.0) | `0.26` |
| `7d-utilization` | Current 7-day window usage | `0.19` |
| `representative-claim` | Which window is the bottleneck | `five_hour` |
| `5h-reset` | Unix timestamp when the 5h window resets | `1775455200` |
| `7d-reset` | Unix timestamp when the 7d window resets | `1775973600` |
| `fallback-percentage` | Capacity allocation ratio | `0.5` |
| `overage-status` | Extra-usage billing state | `allowed` |

**Key insight from 3,702 measured requests:** The 5-hour window is always the binding constraint (`representative-claim` = `five_hour` in 100% of requests). Each 1% of the 5h window costs roughly 1.5M-2.1M visible tokens — but 96-99% of that is `cache_read`. Visible output per 1% is only 9K-16K tokens, suggesting that thinking tokens (invisible in the API response) account for a substantial portion of quota consumption.

### JSONL session logs

Claude Code writes session logs to `~/.claude/projects/<project-path>/` as JSONL files. These are the client-side record of every API interaction.

**Critical caveat:** Extended thinking generates PRELIM entries that duplicate token counts. In measured data, JSONL records **1.93x** the `cache_read` tokens compared to what the proxy sees. Always filter by `stop_reason: "end_turn"` for FINAL entries when calculating real costs.

Subagent sessions have separate JSONL files in the same directory. Their filenames include the subagent's session ID.

---

## 5. Multi-Project Workflow

### Context isolation

Each project directory gets its own:
- CLAUDE.md (project-level instructions)
- `.claude/settings.json` (project-level config overrides)
- Session logs (JSONL files under `~/.claude/projects/<project-path>/`)
- MEMORY.md (persistent memory)

### Settings hierarchy

| Level | File | Overrides |
|-------|------|-----------|
| Global | `~/.claude/settings.json` | Base config |
| Project | `<project>/.claude/settings.json` | Extends/overrides global |

Project settings merge with (not replace) global settings. Hooks from both levels run. Environment variables from the project level override the global level.

### Project switching

- **Start a new session when switching projects.** Don't carry context from project A into project B — it wastes tokens and confuses the model.
- **Each project should be self-contained.** Its CLAUDE.md should have enough context for Claude to work independently without needing the global file to explain project-specific conventions.
- **Share standards, not details.** Global CLAUDE.md: "Use type hints in all Python functions." Project CLAUDE.md: "This project uses FastAPI with SQLAlchemy async, Python 3.12."

### Git workflow across projects

- **Don't auto-commit instruction files.** CLAUDE.md and PLAN files are local working documents. Commit them deliberately if you want them in the repo.
- **Review diffs before committing.** Claude follows patterns it sees in `git log`, so maintaining clean commit history improves future commit quality.
- **Separate concerns.** If Claude makes changes across multiple subsystems, consider splitting into focused commits rather than one large commit.

---

## 6. Rate Limit Architecture & Tactics

Understanding the quota system lets you plan work sessions and avoid surprises. All data here is from proxy-captured headers on a Max 20x ($200/mo) plan — other tiers will differ in absolute numbers but the architecture is the same.

### Dual window system

Two independent sliding windows control your quota:

| Window | Duration | Role | Resets |
|--------|----------|------|--------|
| **5-hour** | ~5 hours | Always the bottleneck | Every ~5 hours (check `5h-reset`) |
| **7-day** | 7 days | Accumulator | Fixed weekly timestamp |

The 7-day counter accumulates roughly 12-17% of each 5h window's peak usage. In practice, the 5h window is always the constraint that triggers rate limiting.

### Per-1% cost (Max 20x, measured)

| Metric | Range | Note |
|--------|-------|------|
| Output per 1% | 9K-16K tokens | Visible output only |
| Cache Read per 1% | 1.5M-2.1M tokens | 96-99% of visible volume |
| Total Visible per 1% | 1.6M-2.1M tokens | Sum of all measured fields |

A full 5h window (100%) corresponds to only **0.9M-1.6M visible output tokens**. For several hours of Opus-class work, this is low — the gap is consistent with extended thinking tokens being counted against the quota at output-token rates, invisible to the client.

### The thinking token blind spot

Extended thinking tokens are **not** included in `output_tokens` from the API response. You cannot see or control this cost from the client side. The practical effect:

- You see 15K output tokens used for a complex reasoning turn
- The actual quota cost may be 50K-200K+ (thinking + output combined)
- The usage bar jumps by an amount that doesn't match your visible token counts
- There is no client-side mitigation

### Practical tactics

| Tactic | Effect | Tradeoff |
|--------|--------|----------|
| Spread heavy work across 5h windows | Stay under limit longer | Requires planning around reset times |
| Avoid multiple terminals | Prevents parallel drain | Less parallelism |
| Keep sessions lean | Less `cache_read` per turn | More frequent fresh starts |
| Check reset times via proxy | Know when quota recovers | Requires proxy setup |
| Fresh sessions over long ones | Avoid Bug 4/5 context degradation | Lose accumulated context |

### When you hit the limit

1. **Check if it's real or Bug B3** (false rate limiter). Look for `"model": "<synthetic>"` in session JSONL. If present, it's a client-generated fake error.
2. **If real:** Check the `5h-reset` header timestamp (via proxy logs). Wait for the reset.
3. **Don't restart Claude Code repeatedly.** It won't help — the limit is server-side and tied to your account, not your session.
4. **Don't open new terminals.** Each new session pays a cold start cost, draining quota faster.

---

## 7. MCP Server Considerations

MCP (Model Context Protocol) servers extend Claude Code with external tools. Understanding their cost profile helps you manage context budget.

### Context cost

Every enabled MCP tool's schema is included in the system prompt on every turn. More tools = more tokens per turn = faster cache growth.

| MCP configuration | Schema overhead per turn |
|------------------|----------------------|
| 0 MCP tools | Baseline |
| 5 MCP tools | +500-2,000 tokens |
| 20 MCP tools | +2,000-8,000 tokens |
| 57 MCP tools (observed) | +10,000+ tokens |

### Tool result size limits

Two different caps apply depending on the tool type:

| Tool type | Per-result limit | Aggregate limit | Override available |
|-----------|-----------------|----------------|-------------------|
| Built-in (Read, Bash, Grep, Glob, Edit) | 30K-50K chars (per GrowthBook flags) | **200K chars** (Bug 5) | No |
| MCP tools | Up to 500K chars | Same 200K aggregate | Yes (`_meta["anthropic/maxResultSizeChars"]`) |

The 200K aggregate budget (`tengu_hawthorn_window`) applies across all tool results in the conversation. After ~15-20 file reads, older results are silently truncated to 1-41 characters. This is Bug 5 — there is no environment variable to disable it.

### Best practices

- **Only enable MCP servers you actively use.** Disable unused servers to reduce per-turn schema overhead.
- **Be aware of the 200K aggregate cap.** Large MCP responses consume the same budget as built-in tool results.
- **Large MCP responses trigger compaction faster.** A single 100K-char response eats half the aggregate budget.
- **Check schema size.** If your MCP server exposes many tools, each one adds to every turn's token count. Consider splitting into focused servers.

---

## 8. GrowthBook Feature Flags

Claude Code uses [GrowthBook](https://www.growthbook.io/) for server-side feature control. Understanding this system explains why behavior can change without a client update.

### What GrowthBook controls

| Flag | Controls | Current default |
|------|----------|----------------|
| `tengu_hawthorn_window` | Aggregate tool result budget (chars) | `200000` |
| `tengu_pewter_kestrel` | Per-tool result size caps | `{global: 50000, Bash: 30000, Grep: 20000}` |
| `tengu_slate_heron` | Time-based microcompact | `enabled: false` |
| `tengu_session_memory` | Session memory compact | `false` |
| `tengu_sm_compact` | SM compact gate | `false` |
| `tengu_cache_plum_violet` | Unknown (only enabled flag) | `true` |
| `tengu_summarize_tool_results` | System prompt flag for model | `true` |

### Why it matters

- Anthropic can change these values **server-side** without pushing a client update
- Docker-pinned old versions (v2.1.74/86) started experiencing drain when flag values changed remotely ([#37394](https://github.com/anthropics/claude-code/issues/37394))
- The disk cache in `~/.claude.json` (`cachedGrowthBookFeatures`) is a snapshot — runtime values may differ based on session attributes
- Feature flags enable A/B testing, so your experience may differ from another user's on the same plan and version

### Inspecting your flags

```bash
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
for k in ['tengu_slate_heron', 'tengu_session_memory', 'tengu_sm_compact',
          'tengu_sm_compact_config', 'tengu_cache_plum_violet',
          'tengu_hawthorn_window', 'tengu_pewter_kestrel']:
    print(f'{k}: {json.dumps(gb.get(k, \"NOT PRESENT\"), indent=2)}')
"
```

---

## 9. Session Management Patterns

### Session lifecycle

| Phase | Typical length | Cache ratio | Context quality |
|-------|---------------|------------|----------------|
| Cold start | 1-3 turns | 54-80% | Full (nothing cleared) |
| Warm | 4-50 turns | 94-99% | Full |
| Mature | 50-150 turns | 96-99% | Degrading (Bug 4/5 active) |
| Late | 150+ turns | 96-99% | Significantly degraded |

Cache ratio stays high throughout because microcompact substitutes the same marker consistently. But context **quality** degrades: the model can no longer see original tool results from earlier in the session.

### When to start a new session

- After completing a self-contained task (don't carry stale context forward)
- When Claude starts repeating approaches or can't reference earlier work (sign of Bug 4)
- When you've done 15-20+ file reads (Bug 5's 200K aggregate cap is likely active)
- Before switching to a different project or subsystem
- After hitting a rate limit (waiting for the 5h reset)

### Compaction control

| Method | What it does | When to use |
|--------|-------------|-------------|
| `/compact` | Summarizes conversation history | Before a topic shift within a session |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Sets autocompact threshold | Delay compaction if you need context |
| `DISABLE_AUTO_COMPACT=true` | Disables autocompact | When you want full manual control |

Note: None of these control microcompact (Bug 4) or budget enforcement (Bug 5). Those operate on a separate code path.

---

## 10. Community Tools & Resources

### Monitoring

| Tool | Purpose | Link |
|------|---------|------|
| CUStats | Web-based session stats viewer | [custats.info](https://custats.info) |
| BudMon | Rate limit header monitor | [github.com/weilhalt/budmon](https://github.com/weilhalt/budmon) |
| context-stats | Cache miss analysis from JSONL | [github.com/luongnv89/cc-context-stats](https://github.com/luongnv89/cc-context-stats) |

### Optimization

| Tool | Purpose | Link |
|------|---------|------|
| tokenlean | 54 CLI tools for agent token optimization | [github.com/edimuj/tokenlean](https://github.com/edimuj/tokenlean) |
| tokpress | Intelligent tool output compression (23 filters) | [pypi.org/project/tokpress](https://pypi.org/project/tokpress/) |

### Diagnosis

| Tool | Purpose | Link |
|------|---------|------|
| ccdiag | CLI diagnostic tool | [github.com/kolkov/ccdiag](https://github.com/kolkov/ccdiag) |
| llm-relay | Multi-CLI proxy + session diagnostics (8 detectors) | [pypi.org/project/llm-relay](https://pypi.org/project/llm-relay/) |

### This repository

| Document | Topic |
|----------|-------|
| [01_BUGS.md](01_BUGS.md) | 7 confirmed bugs — technical root cause analysis |
| [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) | Dual 5h/7d quota architecture from 3,702 proxy requests |
| [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) | Session log analysis methodology (110 main + 279 subagent sessions) |
| [04_BENCHMARK.md](04_BENCHMARK.md) | Performance benchmarks |
| [05_MICROCOMPACT.md](05_MICROCOMPACT.md) | Silent context mutation deep dive (Bugs 4 & 5) |
| [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md) | Systematic test results from April 3 |
| [07_TIMELINE.md](07_TIMELINE.md) | 14-month issue chronicle (91+ issues, 4 escalation cycles) |
| [08_UPDATE-LOG.md](08_UPDATE-LOG.md) | Document update log |
| [09_QUICKSTART.md](09_QUICKSTART.md) | Setup and self-diagnosis |
| [10_ISSUES.md](10_ISSUES.md) | Related issues, community tools, and contributors |

---

## Appendix A: settings.json Reference

A complete `settings.json` combining the patterns from this guide:

```json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$CC_TOOL_INPUT\" | grep -qiE 'rm\\s+-rf\\s+/|drop\\s+table|git\\s+push\\s+--force\\s+(origin\\s+)?main' && echo 'BLOCKED: Dangerous command detected' && exit 1 || exit 0",
            "timeout": 5000
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "filepath=$(echo \"$CC_TOOL_INPUT\" | grep -oP '\"file_path\"\\s*:\\s*\"\\K[^\"]+'); if [ -z \"$filepath\" ] || [ ! -f \"$filepath\" ]; then exit 0; fi; case \"$filepath\" in *.py) d=$(dirname \"$filepath\"); while [ \"$d\" != \"/\" ]; do if [ -f \"$d/.venv/bin/ruff\" ] && [ -f \"$d/pyproject.toml\" ]; then \"$d/.venv/bin/ruff\" check --no-fix \"$filepath\" 2>&1 | head -20; exit 0; fi; d=$(dirname \"$d\"); done ;; esac",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

## Appendix B: Quick Reference Card

### Environment variables

| Variable | Purpose | Recommended |
|----------|---------|-------------|
| `DISABLE_AUTOUPDATER=1` | Prevent auto-updates | Yes |
| `ANTHROPIC_BASE_URL=http://...` | Route through proxy | For monitoring |
| `DISABLE_AUTO_COMPACT=true` | Disable autocompact | Situational |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Set compact threshold | Situational |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=true` | Disable SM compact only | Limited effect |
| `DISABLE_COMPACT=true` | Disable all compaction | Risky (kills manual too) |

### What you can't control

| Mechanism | Why not | Details |
|-----------|---------|---------|
| Microcompact (Bug 4) | Server-controlled via GrowthBook, no env var | [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| Tool result budget (Bug 5) | Server-controlled, 200K aggregate cap | [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| Thinking token cost | Not exposed in API response | [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |
| GrowthBook flag changes | Server-side, can change without client update | Section 8 |
| PRELIM log entries (Bug 8) | Extended thinking behavior | [01_BUGS.md](01_BUGS.md) |

### Session health checks

```bash
# Check your GrowthBook flags
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
print('Budget cap:', gb.get('tengu_hawthorn_window', 'NOT SET'))
print('Microcompact:', gb.get('tengu_slate_heron', {}).get('enabled', 'N/A'))
"

# Count PRELIM vs FINAL entries in a session log
grep -c '"stop_reason":"end_turn"' ~/.claude/projects/*/YOUR_SESSION.jsonl
grep -c 'PRELIM' ~/.claude/projects/*/YOUR_SESSION.jsonl

# Check for synthetic rate limits
grep -c '"model":"<synthetic>"' ~/.claude/projects/*/YOUR_SESSION.jsonl
```

---

*This guide reflects findings as of April 2026 (Claude Code v2.1.91). Claude Code is actively developed — verify current behavior before relying on specific version details. For the underlying data and methodology, see the linked analysis documents in this repository.*
