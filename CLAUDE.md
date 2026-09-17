# Workspace: multi-tasking command center

This folder is my task multiplexer. When I'm working here, I don't want to iterate line-by-line on code — I want you to manage a pipeline of tasks: scope them, isolate them in their own folders + worktrees, run them to completion (spec → commits → PR), and keep me out of the clone directories.

All repos I work on live under `~/projects/`.

## Goals

- Higher-level asks, end-to-end. Given a GitHub issue / bug description / verbal ask, go from "here's the thing" to "there's a PR" without me having to hand-hold.
- Each task is isolated in its own folder with its own worktree(s), so many tasks can run concurrently.
- Prefer a **spec-driven** workflow: I iterate on `TASK.md` in my editor; you one-shot the implementation from it. If something goes wrong, I'd rather fix the spec than iterate with you in the prompt.

## Task lifecycle

When I say "let's work on <thing>" (GitHub issue URL, bug description, verbal ask):

1. **Slug** — invent a short kebab slug (e.g. `parser-timeout`). If there's a GitHub issue, suffix it with the number: `parser-timeout-42`.
2. **Folder** — `mkdir ~/workspace/<slug>/`.
3. **TASK.md** — delegate the initial scaffold to a sub-agent (keeps the top-level context clean for routing). The sub-agent writes `<slug>/TASK.md` with these sections:
   - Always: `# <slug>`, `## Status` (Repo: `<name>`, Issue: URL / none, PR: none yet / URL), `## Goal` (one paragraph from the issue/source), `## Notes` (links to related issues, discussions, prior PRs, docs).
   - Add `## Approach` only when it's load-bearing:
     - If I said "scaffold only" / "just set up the task" — include an empty `## Approach` so I have a place to write the spec.
     - If you're proceeding straight to implementation — include `## Approach` with a short plan before coding (useful for sub-agent handoff and later review).
     - Otherwise, omit it.
   - Enough context that a fresh agent can pick it up. Do **not** investigate root cause yet.
4. **Worktree(s)** — figure out which repo(s) the task touches and create a git worktree per repo under `<slug>/`. See "Repo discovery" below. Naming convention: `<slug>/<repo>.tmp/`.
5. **Decide mode:**
   - If the ask is clearly scoped (an issue with a tight fix, a specific bug description, a well-defined change), continue straight into implementation. The PR-creation gate will still pause for me.
   - If the ask is vague or exploratory, or if I said "just set up the task" / "scaffold only", stop here, summarize what you set up, and wait for me to flesh out `TASK.md` before saying "go" / "implement" / "run the spec".
   - When genuinely unsure which bucket the ask falls into, ask once rather than guessing.

## Repo discovery

All my clones live in `~/projects/`. To find the repo a task touches:

- `ls ~/projects/` and match by name; if the name doesn't make it obvious, grep across the likely candidates.
- Not every folder there is a git repo — check for `.git` before treating one as a target.
- If none match, ask me before cloning anything new.

Creating the worktree:

```sh
git -C ~/projects/<repo> fetch origin
git -C ~/projects/<repo> worktree add ~/workspace/<slug>/<repo>.tmp -b <slug> origin/HEAD
```

Don't assume the default branch is `main` — some repos use `master`. `origin/HEAD` resolves it; if it's unset, run `git remote set-head origin -a` first.

Some clones under `~/projects/` are forks or third-party checkouts whose `origin` is not mine (`github.com/zuqini/...`). Check `git remote -v` before pushing. If `origin` isn't mine and there's no `upstream`/fork remote of mine, ask me how I want to handle it rather than pushing.

## Code changes

- All edits happen in the `.tmp` worktree — **never** in the source clone directories under `~/projects/`. Those are my persistent checkouts; touching them causes conflicts across parallel tasks.
- Small, semantically meaningful commits. Squash typos / fixups into the relevant parent commit rather than leaving them as separate commits.
- Follow the repo's existing commit message style (`git log` in the worktree shows it).
- Don't commit generated / lockfile churn unless the change actually requires it.

## Pull requests

Before creating a PR:

1. `git fetch` + rebase on the latest default branch (`origin/HEAD`, or `upstream/HEAD` if the repo has an `upstream` remote).
2. Run the repo's build / tests / lint (whatever the repo defines — check its README, Makefile, `package.json`, CI config); they must pass locally.
3. **Always** show me the title and description and wait for confirmation before running `gh pr create`.

If the repo has both `upstream` and `origin` remotes: push the branch to `origin` but open the PR against `upstream`.

Checking PR state (use these exact commands — **do not** read bot comments to decide if checks passed):

- `gh pr view <n> --json state,mergedAt,reviewDecision,statusCheckRollup`
- `gh pr checks <n>` — authoritative CI status

After a PR is created:
- Show me the PR URL (and the issue URL, if any).
- If there's a linked issue, make sure the PR body references it (`Closes #<n>`) so GitHub links them. Don't post separate issue comments unless I ask.

## Task status board

When I say "refresh task status" / "what's going on" / "summarize my tasks":

1. Scan every `~/workspace/*/TASK.md`.
2. Cross-check against live GitHub state (`gh issue view`, `gh pr view`). **TASK.md can drift** — treat GitHub as source of truth and update TASK.md if stale.
3. Render a table:

   | Task | Repo | Issue | Issue state | PR | PR state | Next step |

## "What should I work on next?"

Look at: open issues across my repos (`gh issue list --repo zuqini/<repo>` / `gh search issues --owner zuqini --state open`), existing `~/workspace/*` tasks that are blocked or stale, and open PRs that need action (`gh search prs --author @me --state open`). Suggest 2–3 options ranked by priority / staleness / blocker status. Don't pick for me — just recommend.

## Delegation

For non-trivial investigations or implementations, prefer launching a sub-agent (investigator → implementer → reviewer) via the orchestrator skill rather than doing everything inline. This keeps the top-level context clean for task routing.

## Living docs

These instructions are not frozen. If a recurring friction or a missing rule becomes obvious while working, offer to restructure `CLAUDE.md` or a task's `TASK.md` to capture it — don't just work around it silently. Same for `TASK.md` when the spec is unclear or stale: suggest rewriting the spec before iterating on the code.

## Long-running tasks (context spans multiple sessions)

For any task likely to outlive a single Claude session, treat **`TASK.md` as the spec** (what should be true when done) and **`git log` in the worktree as the progress log** (what's actually done). The delta between them is the remaining work. Do not rely on in-conversation memory across sessions.

**On bootstrap into an in-progress task** (TASK.md exists, worktree exists, commits ahead of the default branch):

1. Read `TASK.md`.
2. In the `.tmp` worktree, run `git log --oneline $(git merge-base HEAD origin/HEAD)..HEAD` and `git diff $(git merge-base HEAD origin/HEAD)...HEAD --stat`.
3. Compute the spec-vs-committed delta and continue from there. If the spec is unclear, ask before guessing — don't redo work that's already committed.

**Before `/clear` (or when wrapping a session on an unfinished task):** update `TASK.md` *only* if something decided or learned in chat isn't already captured in the spec or the commits — e.g. "tried X, abandoned because Y", an `## Approach` pivot, a Status bullet for uncommitted partial work. **Do not write a per-step progress journal** — it drifts and starts lying. Commits + tests are the only honest signal of completeness.

**`## Log` / `## Decisions` sections in `TASK.md`:** if I've added one, treat it as **append-only** and never touch `## Goal` / `## Approach` without asking. The Log is yours to write to; the spec is mine.

## Cleanup

When a task's PR is merged (or the task is abandoned) and I say "clean up <slug>" / "close out <slug>":

1. `git -C ~/projects/<repo> worktree remove ~/workspace/<slug>/<repo>.tmp` (add `--force` only if I confirm uncommitted changes can go).
2. `git -C ~/projects/<repo> branch -d <slug>` (or `-D` if I confirm).
3. Remove `~/workspace/<slug>/` after showing me what's in it.

Never delete a task folder or worktree without asking first.

## Out of scope here

When I'm in `~/workspace/`, don't edit my dotfiles, shell config, or anything outside this folder and the task worktrees unless I explicitly ask.
