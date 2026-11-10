# Workspace usage guide

How to drive the multi-tasking workflow defined in `./CLAUDE.md`.

## The flow, regardless of input channel (GitHub issue, bug report, verbal)

1. **Open a new terminal workspace / column** so this task gets its own place.
2. **`cd ~/workspace/`** and start Claude.
3. Say one of these, based on what you have:

| You have... | Say this |
|---|---|
| A GitHub issue with a clear, scoped fix | `let's work on https://github.com/zuqini/<repo>/issues/42` — Claude will slug it, make a folder, write `TASK.md`, create the worktree, implement, commit, and prompt before pushing a PR. |
| A GitHub issue with a vague or exploratory ask | `just set up the task for <issue URL>` — Claude stops after `TASK.md` + worktree. You then open `~/workspace/<slug>/TASK.md` in nvim and **write the spec yourself** (Goal / Approach / Notes). Then tell Claude `go`. |
| A bug or idea with no issue | `set up a task: <describe it>` — same as above, just without the issue. You can file / link an issue later and Claude will backfill it. |
| Nothing specific | `what should I work on next?` — Claude scans open issues + existing workspace tasks + open PRs and suggests 2–3. |
| Status check | `refresh task status` — Claude rebuilds a table of all tasks with live GitHub issue + PR state. |
| A big task with independent workstreams (hours of work, many review rounds) | `run <slug> as lanes` — Claude puts the split in `## Approach`, then starts one named background session per workstream, each in its own worktree with a brief and a log. It verifies and integrates their work itself and still pauses before the PR. |
| A PR to drive to a clean review | `/review-loop <PR URL>` — Claude loops `/code-review xhigh` → a fixer grades each finding and fixes criticals and should-fixes (one commit each, with a test) → re-review, until a round comes back clean. It stops early on stalls (default 6 rounds) and asks before pushing. To keep it going unattended (auto mode), start with `/goal run the review-loop skill on <PR URL>; done when a reply contains a line starting "REVIEW-LOOP DONE" or "REVIEW-LOOP STOPPED", or after 60 turns` instead. |
| Done with a task | `clean up <slug>` — Claude removes the worktree, branch, and task folder (asks first). |

## The one mindset shift (this is the actual hack)

**Stop iterating with Claude in the prompt.** When Claude has scaffolded the task, **open `TASK.md` in nvim and write your spec there.** Claude will one-shot from the spec. If it goes wrong, go back and fix the spec — don't fight it in chat.

## Things Claude will always pause for

- Before creating a PR — you get to see title + description.
- Before pushing to a repo whose `origin` isn't yours (forks / third-party clones under `~/projects/`).
- Before deleting a task folder, worktree, or branch.

## Things Claude will never do

- Edit code in `~/projects/<repo>` directly — it always works in a `.tmp` worktree under the task folder.
- Trust PR bot comments for CI status — uses `gh pr checks` instead.
- Force-push (global rule).

## If you want more autonomy later

Flip to `--dangerously-skip-permissions` when you're comfortable. Right now you keep confirmations. With lanes running, a coordinator in bypass mode asks you to approve each lane's signal, since messages between bypass and prompting sessions are held by default; keep the coordinator in auto mode, or set `crossSessionInbound: "accept"` for it.

## A small nvim tip for the TASK.md loop

You'll be bouncing between `nvim ~/workspace/<slug>/TASK.md` and the Claude prompt. Keep both in the same terminal workspace / column — window management is what kills multi-tasking, and one folder per task kills the chaos.

## Long-running tasks: context handoff

For tasks that span multiple Claude sessions (context fills up, you `/clear`, or you come back the next day):

- **`TASK.md` = the spec** (what *should* be true when done)
- **`git log` in the worktree = the progress log** (what's *actually* done)
- The delta between them is the remaining work. Claude already knows to compute this on bootstrap (the `workspace` skill → "Resume a task").

**Bootstrap prompt to paste into the new session** (if Claude doesn't pick up the in-progress task on its own):

> Read TASK.md in this folder. Then in `*.tmp/`, run `git log --oneline $(git merge-base HEAD origin/HEAD)..HEAD` and `git diff $(git merge-base HEAD origin/HEAD)...HEAD --stat`. Compute what's left between the spec and what's committed, and continue from there.

**When to add a `## Log` section yourself:** for genuinely long / exploratory tasks (multi-day, lots of dead ends). Once you've added it, agents will append to it but won't touch `## Goal` / `## Approach`.

**Why not just have every agent maintain a completion checklist in `TASK.md`?** Two failure modes: (1) an agent that *thinks* it finished step 3 will check it off even when the implementation is broken — only commits + tests are honest; (2) if agents keep editing the spec you iterate on in nvim, they'll fight your edits and the spec stops being a stable contract.

## Lanes: long-lived sessions

For big tasks Claude can run each workstream as a lane: a named background session (`<slug>-<lane>`) in its own worktree, with `lanes/<lane>/BRIEF.md` (its contract) and `LOG.md` (what it decided, tried, or got stuck on). The coordinating session hands out the work, re-runs the checks itself, integrates, and opens the PR. Lanes never push.

- **See them:** `claude agents` (agent view). Lane ids are in TASK.md Status.
- **Watch or steer one:** `claude attach <id>` (← goes back to agent view). To keep one on screen, attach it in a herdr pane. If you change direction while attached, ask the lane to log it as a DECISION. Better: edit its `BRIEF.md` and tell the coordinator, so a restarted lane knows too.
- **When one is stuck:** a lane waiting on a permission prompt stays blocked until you attach and answer it. Moshi pages you as it does for any session. A lane whose first log entry is BLOCKED "gates inactive" started without its push/PR guard; don't let it continue until that's fixed.
- **Restarts are normal:** the coordinator restarts lanes from their files between milestones to keep their context fresh.
- **Cloud:** for work that must keep going while the laptop sleeps, Claude Projects (claude.ai/code → Projects) gives you a cloud coordinator with threads; you start it there. Claude Tag (per-channel Slack agents) needs a Team or Enterprise plan.

## What lives where

| File | Purpose |
|---|---|
| `~/workspace/CLAUDE.md` | Multi-tasking workflow (task folders, `TASK.md` spec, worktrees, PR rules). Loads whenever you `cd` into `~/workspace/`. |
| `~/workspace/.claude/skills/workspace/` | On-demand procedures: status board, what to work on next, resuming a task, cleanup. Loads only when one is needed. |
| `~/workspace/usage.md` | This file. |
| `~/workspace/<slug>/` | One folder per task: `TASK.md` + `<repo>.tmp/` worktree(s). Not tracked in git. |
| `~/workspace/<slug>/lanes/<lane>/` | A lane's `BRIEF.md` (coordinator writes) and `LOG.md` (lane writes). Its worktree is `<slug>/<repo>.<lane>.tmp/`. |
| `~/dotfiles/claude/.claude/skills/lanes/` | Lane protocol: `SKILL.md` (coordinator), `worker.md` (lane), `brief-template.md`, `scripts/start-lane.sh`, `scripts/watch-lanes.py`, `scripts/lane-guard.py` (hook that blocks push/PR/`claude` in any spelling) with `scripts/test_lane_guard.py`. |
| `~/projects/<repo>` | Persistent clones. Claude never edits these directly; worktrees are spawned from them. |
| `~/dotfiles/claude/.claude/CLAUDE.md` | Global personal rules (symlinked to `~/.claude/CLAUDE.md`). |
