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

Flip to `--dangerously-skip-permissions` when you're comfortable. Right now you keep confirmations.

## A small nvim tip for the TASK.md loop

You'll be bouncing between `nvim ~/workspace/<slug>/TASK.md` and the Claude prompt. Keep both in the same terminal workspace / column — window management is what kills multi-tasking, and one folder per task kills the chaos.

## Long-running tasks: context handoff

For tasks that span multiple Claude sessions (context fills up, you `/clear`, or you come back the next day):

- **`TASK.md` = the spec** (what *should* be true when done)
- **`git log` in the worktree = the progress log** (what's *actually* done)
- The delta between them is the remaining work. Claude already knows to compute this on bootstrap (see `CLAUDE.md` → "Long-running tasks").

**Bootstrap prompt to paste into the new session** (if Claude doesn't pick up the in-progress task on its own):

> Read TASK.md in this folder. Then in `*.tmp/`, run `git log --oneline $(git merge-base HEAD origin/HEAD)..HEAD` and `git diff $(git merge-base HEAD origin/HEAD)...HEAD --stat`. Compute what's left between the spec and what's committed, and continue from there.

**When to add a `## Log` section yourself:** for genuinely long / exploratory tasks (multi-day, lots of dead ends). Once you've added it, agents will append to it but won't touch `## Goal` / `## Approach`.

**Why not just have every agent maintain a completion checklist in `TASK.md`?** Two failure modes: (1) an agent that *thinks* it finished step 3 will check it off even when the implementation is broken — only commits + tests are honest; (2) if agents keep editing the spec you iterate on in nvim, they'll fight your edits and the spec stops being a stable contract.

## What lives where

| File | Purpose |
|---|---|
| `~/workspace/CLAUDE.md` | Multi-tasking workflow (task folders, `TASK.md` spec, worktrees, PR rules, status board, cleanup). Loads whenever you `cd` into `~/workspace/`. |
| `~/workspace/usage.md` | This file. |
| `~/workspace/<slug>/` | One folder per task: `TASK.md` + `<repo>.tmp/` worktree(s). Not tracked in git. |
| `~/projects/<repo>` | Persistent clones. Claude never edits these directly; worktrees are spawned from them. |
| `~/dotfiles/claude/.claude/CLAUDE.md` | Global personal rules (symlinked to `~/.claude/CLAUDE.md`). Points at the workspace workflow when CWD is `~/workspace/`. |
