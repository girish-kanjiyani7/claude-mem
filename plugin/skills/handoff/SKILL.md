---
name: handoff
description: Generate a HANDOFF.md grounded in claude-mem's recorded observation data — recent changes, bugfixes attempted, files touched, and current session context — so a fresh Claude session can continue exactly where this one left off. Use when the user wants to stop and resume later, or when Claude is stuck repeating the same failing approach.
---

# Handoff

Generate a `HANDOFF.md` file grounded in claude-mem's actual recorded data for this project. Unlike `/clear` + "continue" (which relies on automatic timeline injection), `/handoff` produces a portable, human-readable briefing doc that works even when stepping away for hours, switching machines, or when the current session is so degraded that you need a clean break with explicit direction about what NOT to try again.

## When to Use

- Stepping away and resuming later (no session continuity via injection)
- Claude is stuck retrying the same failing approach — the handoff explicitly flags what NOT to try again
- The user wants a reviewable, shareable record of where things stand
- The user says "handoff", "generate a handoff", "write a handoff doc", "I want to start fresh"

## Step 1: Resolve the Worker Port

```bash
WORKER_PORT="${CLAUDE_MEM_WORKER_PORT:-$(node -e "const fs=require('fs'),p=require('path'),os=require('os');const uid=(typeof process.getuid==='function'?process.getuid():77);const fallback=String(37700+(uid%100));try{const s=JSON.parse(fs.readFileSync(p.join(os.homedir(),'.claude-mem','settings.json'),'utf-8'));process.stdout.write(String(s.CLAUDE_MEM_WORKER_PORT||fallback));}catch{process.stdout.write(fallback);}" 2>/dev/null)}"
```

Verify the worker is running:

```bash
curl -s "http://localhost:${WORKER_PORT}/api/health" | head -c 100
```

If the worker is not running, note it and fall back to conversation context only (see Notes).

## Step 2: Determine the Project Name

Use the current working directory's basename. Check for git worktrees first:

```bash
git_dir=$(git rev-parse --git-dir 2>/dev/null)
git_common_dir=$(git rev-parse --git-common-dir 2>/dev/null)
if [ "$git_dir" != "$git_common_dir" ]; then
  PROJECT=$(basename "$(dirname "$git_common_dir")")
else
  PROJECT=$(basename "$PWD")
fi
echo "$PROJECT"
```

## Step 3: Pull Data from claude-mem

Run these in parallel to gather the raw material for the handoff:

**Recent context — what claude-mem knows about the current state:**
```bash
curl -s "http://localhost:${WORKER_PORT}/api/context/recent?project=${PROJECT}&limit=20"
```

**Bugfix observations — what was tried and failed:**
```bash
curl -s "http://localhost:${WORKER_PORT}/api/search/by-type?type=bugfix&project=${PROJECT}&limit=15&orderBy=date_desc"
```

**Recent changes — what files and code actually changed:**
```bash
curl -s "http://localhost:${WORKER_PORT}/api/changes?project=${PROJECT}&limit=15"
```

**Recent decisions — architecture choices, approach selections:**
```bash
curl -s "http://localhost:${WORKER_PORT}/api/decisions?project=${PROJECT}&limit=10"
```

**Current git state:**
```bash
git status --short && echo "---" && git log --oneline -5
```

## Step 4: Synthesize and Write HANDOFF.md

Using the data from Step 3 plus current conversation context, write `HANDOFF.md` to the project root. The doc is addressed to an incoming Claude agent that has never seen this conversation.

```markdown
# Handoff

> Generated: [ISO timestamp]
> Project: [project name]
> Based on [N] recorded observations from claude-mem

## Goal

[What the user is trying to accomplish — the actual end state, not the current sub-task.
One paragraph. Specific enough that a fresh agent can orient in 10 seconds.]

## Current State

**Working:**
- [Confirmed working — be specific, cite files if relevant]

**Broken / Blocked:**
- [Exact symptom or error. Paste error messages verbatim if available from observations.]

**Git status:**
- [Uncommitted changes, current branch, last 3 commits]

## Files in Play

| File | Status | Why It Matters |
|------|--------|---------------|
| `path/to/file` | modified / created / read | [one line] |

*Derived from claude-mem change observations and current git status.*

## What Has Been Tried (Do Not Repeat)

*Sourced from claude-mem bugfix observations for this project.*

### [Attempt name from observation title]
- **What:** [what was done]
- **Why it failed:** [root cause from observation narrative/facts]

### [Next attempt...]

## Current Best Theory

[The most promising path forward based on recent decisions and discoveries.
Include the reasoning — what evidence points here.]

## Next Steps

1. [Specific action — file path, function name, command to run]
2. [Next step]
3. ...

## Key Constraints

*From claude-mem observations and this session:*
- [Non-obvious constraint or user preference]
- [Things explicitly ruled out — reference the failed attempt above]
```

## Step 5: Tell the User

After writing:

> `HANDOFF.md` written to `[path]`.
> 1. Run `/clear` or start a new Claude Code session
> 2. Say: **"Read HANDOFF.md and continue from where we left off"**
>
> The "What Has Been Tried" section is sourced from your claude-mem bugfix observations — the fresh agent won't repeat the same mistakes.

## Notes

- **Worker unavailable:** Fall back to synthesizing from conversation context alone. Add this banner to the top of the doc: `> ⚠️ claude-mem worker unavailable — based on conversation context only, not recorded observations.`
- Keep the handoff under ~150 lines. Dense and specific beats long and vague.
- The "What Has Been Tried" section is the most important — it's what separates this from a plain `/clear` + "continue".
