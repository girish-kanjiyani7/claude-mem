---
name: handoff
description: Generate a HANDOFF.md grounded in claude-mem's recorded observations — using the MCP search/timeline/get_observations tools to surface recent bugfixes, changes, decisions, and files touched — so a fresh Claude session can continue exactly where this one left off without repeating failed approaches. Use when stepping away, or when Claude is stuck in a loop.
---

# Handoff

Generate a `HANDOFF.md` file grounded in claude-mem's actual recorded observations for this project. Uses the same MCP tool workflow as `/mem-search` — search → filter → fetch — to pull real data: what changed, what failed, what was decided. The result is a portable briefing doc a fresh session can act on immediately.

**Why this differs from `/clear` + "continue":** Timeline injection gives the next session context. Handoff gives it *direction* — specifically what NOT to try again, sourced from actual bugfix observations. That's the gap it fills when someone is stuck in a loop.

## When to Use

- Claude is stuck retrying the same failing approach
- Stepping away and resuming hours or days later
- The user wants a reviewable record of current state
- The user says "handoff", "write a handoff doc", "I want to start fresh"

## Workflow

### Step 1: Determine the Project Name

Check for git worktrees before using the directory basename:

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

### Step 2: Gather Observations (3-Layer MCP Workflow)

**NEVER fetch full observation details without filtering first. Follow the search → filter → fetch pattern.**

#### 2a. Search — get index tables for each observation type

Run these searches using the `search` MCP tool:

```
search(query="*", type="observations", obs_type="bugfix", project=PROJECT, limit=20, orderBy="date_desc")
```
```
search(query="*", type="observations", obs_type="change", project=PROJECT, limit=20, orderBy="date_desc")
```
```
search(query="*", type="observations", obs_type="decision", project=PROJECT, limit=10, orderBy="date_desc")
```

Each returns a compact index (~50-100 tokens/row). Review titles and timestamps — discard anything clearly unrelated to the current task.

#### 2b. Timeline — get narrative arc around the most recent work

Use the `timeline` MCP tool to understand what happened leading up to now:

```
timeline(query="recent work", depth_before=10, depth_after=2, project=PROJECT)
```

This shows sessions, observations, and prompts interleaved — use it to understand the sequence of attempts, not just isolated observations.

#### 2c. Fetch — get full details only for relevant IDs

After reviewing the index from 2a and narrative from 2b, select the IDs that matter. Discard the rest.

```
get_observations(ids=[ID1, ID2, ID3, ...], project=PROJECT)
```

Use `get_observations` for all selected IDs in a single call. Returns full narrative, facts, files_modified, and concepts (~500-1000 tokens each) — only fetch what you'll actually use in the handoff.

Also get current git state:

```bash
git status --short && git log --oneline -5
```

### Step 3: Synthesize and Write HANDOFF.md

Write `HANDOFF.md` to the project root. Address it to an incoming Claude agent that has never seen this conversation. Every section must be specific enough to act on immediately.

```markdown
# Handoff

> Generated: [ISO timestamp]
> Project: [project name]
> Observations reviewed: [N bugfix] bugfixes · [N change] changes · [N decision] decisions

## Goal

[What the user is trying to accomplish — the end state, not the current sub-task.
One paragraph. Specific enough that a fresh agent orients in 10 seconds.]

## Current State

**Working:**
- [Confirmed working — cite files/functions if relevant]

**Broken / Blocked:**
- [Exact symptom or error verbatim from observations or conversation]

**Git status:**
- Branch: [branch name]
- Uncommitted: [list or "clean"]
- Last commits: [3 one-liners from git log]

## Files in Play

| File | Status | Why It Matters |
|------|--------|----------------|
| `path/to/file` | modified / created / read | [one line from change observations] |

*Sourced from claude-mem change observations + git status.*

## What Has Been Tried (Do Not Repeat)

*Sourced from claude-mem bugfix observations.*

### [Observation title]
- **What:** [what was attempted — from observation facts]
- **Why it failed:** [root cause — from observation narrative, not just "it didn't work"]

### [Next observation...]

## Current Best Theory

[Most promising path forward, drawn from recent decision observations and discoveries.
Include the reasoning — what evidence points here. If no clear theory exists, say so explicitly.]

## Next Steps

1. [Specific, actionable — include file path or command]
2. [Next step]
3. ...

## Key Constraints

*From claude-mem observations:*
- [Non-obvious constraint or preference surfaced in observations]
- [Explicitly ruled-out approaches — link back to the failed attempts above]
```

### Step 4: Tell the User

> `HANDOFF.md` written.
> 1. Run `/clear` or start a new Claude Code session
> 2. Say: **"Read HANDOFF.md and continue from where we left off"**
>
> "What Has Been Tried" is sourced from your recorded bugfix observations — the fresh agent won't repeat the same approaches.

## Notes

- If no bugfix/change/decision observations exist yet for this project, note it in the doc and fall back to synthesizing from conversation context. Add: `> ⚠️ No observations recorded yet — based on conversation context only.`
- Keep the handoff under ~150 lines. Dense and specific beats long and vague.
- The files listed in `get_observations` results include `files_modified` — use those directly for the Files in Play table rather than guessing.
