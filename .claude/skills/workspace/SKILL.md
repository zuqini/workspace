---
name: workspace
description: "Procedures for my ~/workspace task folders: refresh the task status board, suggest what to work on next, resume an in-progress task (including taking over its lanes), and clean up a finished task. Use when I say 'refresh task status', 'what's going on', 'summarize my tasks', 'what should I work on next?', 'clean up <slug>' or 'close out <slug>', or when a session starts in a task that already has a TASK.md and commits."
---

## Status board

When I say "refresh task status", "what's going on" or "summarize my tasks":

1. Read every `~/workspace/*/TASK.md`.
2. Cross-check against live GitHub state (`gh issue view`, `gh pr view`). GitHub wins; update any stale TASK.md.
3. Render:

   | Task | Repo | Issue | Issue state | PR | PR state | Next step |

   For a task with `lanes/`, Next step gets each lane's live state (`ListAgents`, or `claude agents --json --all --cwd ~/workspace/<slug>`) and its latest `LOG.md` heading.

## What to work on next

Look at open issues across my repos (`gh search issues --owner zuqini --state open`), `~/workspace/*` tasks that are blocked or stale, and my open PRs that need action (`gh search prs --author @me --state open`). Recommend 2–3, ranked by priority, staleness and blockers. Don't pick for me.

## Resume a task

When a task has a `TASK.md`, a worktree, and commits ahead of the default branch:

1. Read `TASK.md`, then the worktree's commits and diff stat since its merge base with `origin/HEAD`.
2. If `<slug>/lanes/` exists, you're its coordinator now: check live lanes (`ListAgents`, or `claude agents --json --all --cwd ~/workspace/<slug>`), read each lane's latest `LOG.md` entry, match them to the lane ids in TASK.md Status, and put your session name and `[ref]` in each brief as coordinator.
3. Continue from the spec-vs-committed delta. If the spec is unclear, ask before guessing; don't redo committed work.

## Clean up

When a task's PR is merged or the task is abandoned and I say "clean up <slug>" or "close out <slug>". Show me what goes and get my go-ahead before deleting anything.

1. Lanes: `claude agents --json --all --cwd ~/workspace/<slug>`, then `claude rm <id>` for each. Their `<repo>.<lane>.tmp` worktrees and `<slug>-<lane>` branches go in steps 2 and 3.
2. `git -C ~/projects/<repo> worktree remove ~/workspace/<slug>/<repo>.tmp`; add `--force` only if I confirm uncommitted changes can go.
3. `git -C ~/projects/<repo> branch -d <slug>` (`-D` if I confirm).
4. Remove `~/workspace/<slug>/` after showing me what's in it.
