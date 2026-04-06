# Quick Start — Setup & Self-Diagnosis

> Practical steps to minimize token drain. For the technical analysis behind these recommendations, see [01_BUGS.md](01_BUGS.md).

---

## Installation

### New Users (recommended: npm)

```bash
# 1. Install via npm (no Sentinel bug)
npm install -g @anthropic-ai/claude-code

# 2. Disable auto-update
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
EOF

# 3. Verify
claude --version   # should show 2.1.91 or later
file $(which claude)   # should show symbolic link to cli.js
```

### Existing Standalone Users

```bash
# 1. Update to v2.1.91
claude update

# 2. Disable auto-update (add to existing settings.json)
# Add "DISABLE_AUTOUPDATER": "1" to the "env" section

# 3. Verify
claude --version   # should show 2.1.91 or later
```

---

## Behaviors to Avoid

| Behavior | Why | Impact |
|----------|-----|--------|
| `--resume` | Replays entire conversation history as billable input | 500K+ tokens per resume |
| `/dream`, `/insights` | Background API calls consume tokens without visible output | Silent drain |
| v2.1.89 or earlier standalone | Sentinel bug causes sustained 4-17% cache read | 3-4x token waste |
| Enabling auto-update | Future versions may reintroduce regressions | Pin v2.1.91 until Bugs 3-5 are fixed |

## Behaviors to Use with Caution

| Behavior | Why | Recommendation |
|----------|-----|----------------|
| Parallel sub-agents (single terminal) | Each agent starts with fresh context, but warms up | **Safe** — agents warm to 94-99% after 3-5 requests |
| Multiple terminals simultaneously | No cache sharing, parallel quota drain | **Limit to one active terminal** |
| Large CLAUDE.md / context files | Sent on every turn | Keep lean |
| Session start / compaction | `cache_creation` spikes are structural | Normal — budget for it |

---

## Self-Diagnosis

### Check Your Session JSONL

Session files in `~/.claude/projects/` contain usage data for each turn:

- **Healthy session:** `cache_read` >> `cache_creation` (read ratio > 80%)
- **Affected session:** `cache_creation` >> `cache_read` (read ratio < 40%)

If most sessions show low read ratios, you're likely on an affected version. Update to v2.1.91.

### Optional: Monitor Cache via Proxy

```bash
# Route through local proxy for monitoring
ANTHROPIC_BASE_URL=http://localhost:8080 claude

# Parse cache_creation_input_tokens / cache_read_input_tokens from responses
# Healthy: read ratio > 80%  |  Affected: read ratio < 40%
```

---

*See [01_BUGS.md](01_BUGS.md) for technical root causes, [04_BENCHMARK.md](04_BENCHMARK.md) for npm vs standalone comparison.*
