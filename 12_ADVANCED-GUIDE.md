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

### Real-world tiering example

Across months of daily usage with 14+ projects, the following three-tier split has proven effective:

**Global (~40 lines)** covers what never changes between projects:
- Coding language preference (Korean conversations, English code/config)
- Git commit conventions (no auto-commits of instruction files, no co-author tags on local working docs)
- Security policy (never hardcode secrets — `os.getenv()` with safe defaults, parameterized SQL only)
- A one-line pointer to a shared coding standards document

**Project A — backend API (~80 lines):**
- Python 3.12, FastAPI + SQLAlchemy async
- Test runner command (`pytest -x tests/`)
- Key module paths (`api/routes/`, `api/models/`, `api/services/`)
- Error log location (`docs/error-log.md`)

**Project B — data pipeline (~60 lines):**
- Python 3.12, CLI-only architecture (no web server, no API)
- Data quality rules (source deduplication, encoding normalization)
- Source format specs (JSON lines, UTF-8, specific date formats)

Combined: ~180 lines across 3 tiers — well under budget. Every line earns its place.

### Cost analysis

A 200-line CLAUDE.md is roughly 2,000 tokens. At typical cache rates (96-99% `cache_read`), this costs cache-read price per turn — cheap individually, but it compounds:

| CLAUDE.md size | Tokens | Per-turn cost at cache rate | Risk |
|---------------|--------|---------------------------|------|
| 50 lines | ~500 | Negligible | None |
| 200 lines | ~2,000 | Low | None |
| 500 lines | ~5,000 | Moderate | Cache-miss risk increases |
| 1,000+ lines | ~10,000+ | High | Diminishing returns, cache fragility |

The system prompt (including CLAUDE.md) forms the cache prefix. Larger prefixes mean more bytes to hash and match. Beyond ~500 lines, you increase the chance of cache misses from minor changes, and the instruction-following benefit plateaus.

In practice, one CLAUDE.md grew to 800+ lines over 4 months because new rules were added but old ones were never removed. It included full API docs for a library Claude already knew, ASCII architecture diagrams that changed monthly, and a task backlog that was 3 months stale. Trimming it to 150 lines had zero measurable impact on Claude's output quality — but saved ~6,500 tokens per turn. The lesson: schedule periodic reviews of your instruction files. If a rule hasn't influenced Claude's behavior in the last 2 weeks, it probably does not need to be there.

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

Early attempts at using MEMORY.md put everything in one file — project status, user preferences, feedback, references, career notes, system configuration. It grew to 300+ lines and was loaded every turn. Restructuring it as a pure index (one-line pointers to individual topic files) cut the per-turn load to ~50 lines while preserving all the stored knowledge. The individual files are only read when relevant — Claude sees the index entry "GPU freeze debugging notes" and only reads the full file when actually working on that problem. This is the difference between paying 3,000 tokens per turn unconditionally versus paying 500 tokens per turn plus 2,500 tokens only when needed.

### PLAN.txt separation

Planning documents — current approach, task breakdowns, error logs, what was tried and failed — should never go in CLAUDE.md. They change every session and would invalidate the cache prefix on every update. Instead, keep them in a PLAN.txt file that Claude reads on demand.

Don't commit PLAN files to git — they're session-specific working documents, not source code. CLAUDE.md is a judgment call: commit it if the team needs shared instructions, keep it local if it contains personal workflow preferences.

### Anti-patterns

| Anti-pattern | Why it hurts | Fix |
|-------------|-------------|-----|
| Pasting library docs into CLAUDE.md | Bloats every turn, model already knows popular libraries | Link to docs or let the model read them on demand |
| Including file trees that change often | Any change to the tree invalidates the cache prefix | Use glob patterns or key-file lists instead |
| Session-specific state in CLAUDE.md | Stale across sessions, pollutes global instructions | Use PLAN files or session-local notes |
| Duplicating rules across tiers | Wastes tokens, risks contradictions | Put each rule in exactly one tier |
| Never pruning old rules | Instruction bloat with zero benefit | Review monthly, delete what the model follows by default |

---

## 2. Subagent Strategy

Claude Code can spawn independent AI instances ("subagents") to handle subtasks in parallel. Each subagent is a full Claude Code session with its own context window.

### Cost profile

| Phase | Cache ratio | Note |
|-------|------------|------|
| Cold start (1st request) | 54-87% | Full cache build — expensive (varies by version/model) |
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

The cold start overhead (~5,000-15,000 tokens for cache warmup) means subagents only pay off for tasks that take 5+ turns of work. A single file search or a targeted edit is always cheaper done directly in the parent session.

### Context passing

Subagents read CLAUDE.md (paying the token cost per agent), but they do **not** inherit the parent session's conversation history. Effective patterns:

- **Explicit prompts:** Give the subagent the exact file paths, search terms, and expected output format. Don't assume it knows what you discussed earlier.
- **PLAN files:** For complex multi-step tasks, write a plan to a file and have the subagent read it. This costs one `Read` call instead of stuffing context into the prompt.
- **Scoped instructions:** If a subagent needs project-specific rules, those are already in CLAUDE.md. Don't repeat them in the prompt.

In practice, early attempts at subagent dispatch launched agents with minimal context and expected them to "figure it out" from CLAUDE.md. This led to repeated work — subagents re-reading files the parent had already analyzed, or taking approaches the parent had already ruled out. The fix: write a brief working file with what has been tried and what specifically needs investigation, then have the subagent read it as its first action. This small investment (one file write, ~30 seconds) routinely saves 3-5 turns of wasted subagent work.

### Parallel dispatch

For a project ecosystem spanning 14+ repositories, dispatching 5-8 parallel search agents across different repos is a common pattern. Each agent costs a cold start (~54-87% cache on first request), but the wall-clock speedup for cross-repo exploration is 3-5x compared to sequential searching.

The key lesson: give each agent a specific, scoped question — "Find where rate limiting is configured in the API module" not "explore the codebase." Vague prompts lead to agents wandering through irrelevant files, burning tokens on exploration that produces nothing actionable.

Be aware that all agents in a session draw from the same rate limit quota. Five parallel agents drain five times faster than sequential work. This is a deliberate tradeoff — you are spending quota to buy wall-clock time.

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

This hook went through 3 iterations before reaching its current form. The first version was just `ruff check $filepath` — it broke on non-Python files. The second added a `case *.py` filter — it broke on projects without ruff installed, failing with cryptic "command not found" errors that Claude tried to fix by installing ruff globally. The third added the venv tree-walk detection — it finds the nearest project's ruff binary automatically and handles multi-project setups cleanly. The final version took 10 minutes to develop but saves hours of catching lint errors late in a session.

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

In practice, this hook caught 3 actual dangerous commands in the first month of use — all from Claude misinterpreting ambiguous instructions. One was a `DROP TABLE` generated when the prompt said "clean up the test data" (Claude interpreted "clean up" as "delete the table"). PreToolUse hooks are the seatbelt you do not think you need until you do.

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

Running multiple projects simultaneously with Claude Code requires strict context hygiene. Without it, instructions leak between projects, stale context wastes tokens, and the model gets confused about which codebase it is working in.

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

The pattern that works across months of running 14+ projects daily: one terminal, one project, one session. When switching projects, close the session entirely. The 30 seconds to restart is cheaper than the confusion from leaked context.

Cross-project pattern sharing works through the global CLAUDE.md tier. Security rules, git conventions, and language preferences go there once. Project-specific tech stack, conventions, and key paths go in the project CLAUDE.md. This avoids duplication while keeping each project self-contained.

A common failure mode: editing Project A's CLAUDE.md while Claude is working on Project B. The change propagates on the next turn and can confuse the model — it suddenly sees instructions for a different tech stack injected into its system prompt. Keep instruction file edits separate from coding work.

### Git workflow across projects

- **Don't auto-commit instruction files.** CLAUDE.md and PLAN files are local working documents. Commit them deliberately if you want them in the repo — and think twice before doing so.
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
- There is no known client-side mitigation as of v2.1.91

### Practical tactics

| Tactic | Effect | Tradeoff |
|--------|--------|----------|
| Spread heavy work across 5h windows | Stay under limit longer | Requires planning around reset times |
| Avoid multiple terminals | Prevents parallel drain | Less parallelism |
| Keep sessions lean | Less `cache_read` per turn | More frequent fresh starts |
| Check reset times via proxy | Know when quota recovers | Requires proxy setup |
| Fresh sessions over long ones | Avoid Bug 4/5 context degradation | Lose accumulated context |

In practice, a typical productive day on a Max 20 plan uses 60-80% of the 5-hour window during peak work. Spreading heavy work (code generation, large refactors) across multiple 5h windows is more effective than cramming everything into one sprint. Knowing that quota refreshes at a specific time — visible through proxy-captured `5h-reset` headers — lets you schedule intensive work right after a reset and save lighter tasks (code review, documentation, planning) for the tail end of a window.

### Organization-level quota pooling

If you are on a team plan, watch for unexplained rate limits. Organization-level accounts share the quota pool — the rate limiting is keyed by `organizationUuid`, not by individual user. A teammate running heavy agent workloads can drain your available capacity without warning. This was initially mistaken for a personal-level bug before the `organizationUuid`-keyed cache was discovered in proxy analysis. If your usage bar jumps when you have not been working, check whether a colleague is running parallel sessions under the same org account.

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
| Cold start | 1-3 turns | 27-85% | Full (nothing cleared) |
| Warm | 4-50 turns | 94-99% | Full |
| Mature | 50-150 turns | 96-99% | Degrading (Bug 4/5 active) |
| Late | 150+ turns | 96-99% | Significantly degraded |

Cache ratio stays high throughout because microcompact substitutes the same marker consistently. But context **quality** degrades: the model can no longer see original tool results from earlier in the session.

### Session startup ritual

An effective session startup pattern, refined over months of daily use:

1. **Read the error log** from the last session (if one exists). This prevents repeating known failures.
2. **Read the current PLAN file.** This gives Claude the full picture of what has been tried, what worked, and what is next.
3. **State the specific goal** for this session in the first message.

This 3-step pattern prevents Claude from repeating past mistakes and gives it focused direction immediately. Sessions started this way are consistently more productive than "continue where we left off" approaches, which force Claude to infer the current state from stale context.

### When to start a new session

- After completing a self-contained task (don't carry stale context forward)
- When Claude starts repeating approaches or can't reference earlier work (sign of Bug 4)
- When you've done 15-20+ file reads (Bug 5's 200K aggregate cap is likely active)
- Before switching to a different project or subsystem
- After hitting a rate limit (waiting for the 5h reset)

### Recognizing a dying session

The clearest signal that a session is past its useful life: Claude starts suggesting approaches you already tried earlier in the session, cannot quote specific lines from files it read 20+ turns ago, or gives increasingly generic advice where it was previously specific. These are symptoms of Bug 4/5 — the information is literally gone from its context, not "forgotten." The model is not getting worse; it is working with less data. No amount of prompting will recover information that has been silently truncated from tool results.

When you see these signs, start a fresh session. Do not try to "remind" Claude by re-explaining — that just wastes tokens in a degraded context. A clean restart with a focused PLAN file gets better results in fewer turns.

### Why /compact is not recommended

For the essential reasoning behind this recommendation, see [11_USAGE-GUIDE.md](11_USAGE-GUIDE.md). This section expands on the implementation details for power users.

`/compact` performs lossy compression — it drops specific variable names, exact error messages, line numbers, and decision rationale. The resulting summary gives Claude a vague outline of "what was discussed" but lacks the precision needed to continue work accurately. In practice, post-compact sessions produce more hallucinations and redundant work than fresh sessions that read well-maintained documents.

**The better pattern: document-based session handoff.**

Instead of compacting, update your working documents at the end of each session while the full conversation is still available for cross-validation:

1. **Before closing:** Ask Claude to update PLAN files, error logs, or status documents based on the current session's work. Cross-check the updates against the actual conversation — Claude can verify its own document updates while it still has access to the full context.
2. **Start the next session fresh.** The new session reads the updated documents (step 1 of the session startup ritual) and has clean, verified context.

This pattern works because you control what is preserved — not an automated summarizer. Each session's learnings are captured in durable, human-verified files rather than trapped in a lossy AI summary. Over months of multi-project operation, this approach consistently produces better session continuity than any combination of `/compact` and `--resume`.

This is an instance of [Andrej Karpathy's "LLM Wiki" pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): rather than re-deriving context on every query, incrementally maintain structured documents that the LLM reads and updates. In this context, CLAUDE.md is the schema (behavioral instructions), MEMORY.md is the index (pointers to topic files), and PLAN files are the working log (session-specific state). The three-tier CLAUDE.md architecture (Section 1) maps directly to the wiki's layered structure. This workflow should ideally be automated by the tool — but until it is, manual document maintenance is the most reliable approach to preserving context quality across sessions.

### Compaction environment variables

| Variable | What it does | Practical note |
|----------|-------------|----------------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Sets autocompact threshold | Delays compaction, buys more time before context loss |
| `DISABLE_AUTO_COMPACT=true` | Disables autocompact | Only affects the main autocompact — not microcompact |
| `DISABLE_COMPACT=true` | Disables all compaction | Risky: disables manual `/compact` too |

**Important:** None of these control microcompact (Bug 4) or budget enforcement (Bug 5). Those operate on a separate code path controlled by server-side GrowthBook flags.

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
| [01_BUGS.md](01_BUGS.md) | 6 confirmed client-side bugs — technical root cause analysis |
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
