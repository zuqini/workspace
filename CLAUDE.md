# Workspace: multi-tasking command center

This folder is my task multiplexer. Given a GitHub issue, bug report or verbal ask, take it from "here's the thing" to "there's a PR" without hand-holding, with each task isolated in its own folder and worktree(s) so many can run at once, and keep me out of the clone directories. Work spec-driven: I iterate on `TASK.md` in my editor and you one-shot the implementation from it. When something goes wrong, I'd rather fix the spec than iterate in chat.

All repos I work on live under `~/projects/`. The `workspace` skill has the on-demand procedures: status board, what to work on next, resuming a task, cleanup.

**If your launch prompt says you are a lane**, you are a worker, not the router: follow `~/.claude/skills/lanes/worker.md` and your `BRIEF.md`, and skip the task lifecycle and pull request sections below.

## Task lifecycle

When I say "let's work on <thing>":

1. **Slug**: a short kebab slug (`parser-timeout`), suffixed with the issue number if there is one (`parser-timeout-42`).
2. **Folder**: `~/workspace/<slug>/`.
3. **TASK.md** in it, with `# <slug>`, `## Status` (Repo, Issue URL or none, PR none yet or URL), `## Goal` (one paragraph from the source) and `## Notes` (related issues, discussions, prior PRs, docs). Add `## Approach` only when it's load-bearing:
   - empty, if I said "scaffold only" or "just set up the task", so I have a place to write the spec;
   - a short plan, if you're going straight to implementation.

   Give a fresh agent enough context to pick it up, but don't investigate root cause yet.
4. **Worktree(s)**: one per repo the task touches, at `<slug>/<repo>.tmp/` (see Repo discovery).
5. **Mode**:
   - Clearly scoped (a tight issue, a specific bug, a well-defined change): continue into implementation. The PR gate still pauses for me.
   - Vague or exploratory, or I said "scaffold only": stop, summarize what you set up, and wait for me to flesh out `TASK.md` and say "go".
   - Unsure which: ask once.

## Repo discovery

Match the task to a repo under `~/projects/`; if none fits, ask before cloning anything. Create the worktree with:

```sh
git -C ~/projects/<repo> fetch origin
git -C ~/projects/<repo> worktree add ~/workspace/<slug>/<repo>.tmp -b <slug> origin/HEAD
```

`origin/HEAD` resolves the default branch (some repos use `master`); if it's unset, run `git remote set-head origin -a` first.

Some clones are forks or third-party checkouts whose `origin` isn't mine (`github.com/zuqini/...`). Check `git remote -v` before pushing; if `origin` isn't mine and there's no fork remote of mine, ask how to handle it.

## Code changes

- All edits happen in the `.tmp` worktree, never in the clones under `~/projects/`: those are my persistent checkouts, and touching them causes conflicts across parallel tasks.
- Small, semantically meaningful commits. Squash typos and fixups into the commit they fix.
- Don't commit generated or lockfile churn unless the change requires it.

## Pull requests

Before creating a PR, run the repo's build, tests and lint; they must pass locally. Then show me the title and description and wait for my confirmation before `gh pr create`.

Don't rebase reflexively. The worktree is cut from the latest default branch when the task is created; after that, rebase only for a reason: a conflict with the default branch, a CI failure it already fixed, or I ask. Rebasing a pushed branch means a force-push, so it costs more than it looks.

If the repo has both `upstream` and `origin`, push the branch to `origin` and open the PR against `upstream`.

Check PR state with `gh pr view <n> --json state,mergedAt,reviewDecision,statusCheckRollup` and `gh pr checks <n>` (authoritative CI status), never by reading bot comments.

If there's a linked issue, the PR body says `Closes #<n>`. Don't post separate issue comments unless I ask.

## Delegation

To review and fix a PR until it's clean, use `/review-loop`.

**Subagents or lanes.** One session with subagents is the default. When a task will run for hours or many review rounds, splits into independent workstreams that need their own worktrees, or I'd want to steer workstreams separately, run it as **lanes**: named, long-lived background sessions, one per workstream, each with a brief and a log under `<slug>/lanes/<lane>/`. The `lanes` skill has the protocol, and the tripwires that say when to switch mid-task. Lanes report to you, never to each other, and never push or open PRs; you verify, integrate and hold the PR gate. Put the split (lanes, the paths each owns, acceptance checks) in `## Approach` before starting any; the mode rules above decide whether you wait for my go.

Whatever the topology, agents talk through files (TASK.md, briefs, logs, commits, beads), not conversation, and any agent must be restartable from those files.

## Long-running tasks

`TASK.md` is the spec (what should be true when done); `git log` in the worktree is the progress (what's actually done). The delta is the remaining work; don't rely on conversation memory across sessions. `TASK.md` Status can drift: GitHub is the source of truth for issue and PR state. The `workspace` skill has how to resume an in-progress task.

Before `/clear`, or when wrapping up an unfinished task, update `TASK.md` only with what was decided or learned in chat and isn't in the spec or commits: tried X and abandoned it because Y, an Approach pivot, uncommitted partial work. Don't write a per-step progress journal; it drifts and starts lying. Commits and tests are the honest signal.

If I've added a `## Log` or `## Decisions` section to `TASK.md`, it's append-only and yours to write; `## Goal` and `## Approach` are mine, so ask before touching them.

## Living docs

These instructions aren't frozen. When a recurring friction or a missing rule becomes obvious, offer to restructure this file or a task's `TASK.md` instead of working around it. Same when a spec is unclear or stale: suggest rewriting it before iterating on the code.

## Never without asking

- Delete a task folder, worktree or branch.
- Edit my dotfiles, shell config, or anything outside this folder and the task worktrees.
