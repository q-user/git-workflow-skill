---
name: git-workflow
description: >-
  Use when establishing branching strategies, implementing Conventional Commits,
  creating or reviewing PRs, resolving PR review comments, merging PRs (including
  CI verification, auto-merge queues, and post-merge cleanup), managing PR review
  threads, merging PRs with signed commits, handling merge conflicts, creating
  releases, integrating Git with CI/CD, setting up git hooks (lefthook,
  captainhook, husky, pre-commit), coordinating multiple agents in parallel with
  isolated git worktrees, or driving implementation from GitHub Issues in
  https://github.com/q-user/hh_apply/issues.

  Also use for post-merge handoff: closing issues, updating GitHub Project
  statuses (Todo/In Progress/In Review/Done), preparing local repo for the next
  task, cleaning up merged branches and git worktrees.

  Trigger on "parallel agents", "worktree", "GitHub issue", "issue",
  "hh_apply issues", "finish the task", "wrap up", "complete the issue",
  "prepare for next task", "close the PR and issue", "update project board",
  or any post-merge cleanup request.
disable-model-invocation: false
---

# Git Workflow Skill

Expert patterns for Git version control: branching, commits, collaboration, and CI/CD.

## Critical Rules (Non-Negotiable)

1. **No direct push to main** — always open a PR.
2. **No merge before all review threads are resolved** — run the merge gate described in `PR Merge Requirements` below.
3. **No squash unless user asked** — atomic commits preserved; keeps GPG signatures and bisection.
4. **No "tested/verified/working" without pasted command output** — if you cannot run the check, say so.
5. **No edits to installed skill/plugin cache paths** (`~/.claude/skills/`, `~/.claude/plugins/cache/`, `**/.bare/**`) — always the repo worktree. Verify `pwd` first.
6. **Force-push only with `--force-with-lease`** — never plain `--force`.
7. **Parallel agents never share a worktree** — create one git worktree and one branch per agent/task, then merge results through an integration branch/worktree.
8. **Work is issue-driven for this repo** — use https://github.com/q-user/hh_apply/issues as the task source of truth. Link branches, worktrees, commits, PRs, and completion notes back to the relevant issue.

## Reference Files

Load references on demand based on the task at hand. Do not cite missing reference files; if a reference is absent, use the self-contained guidance in this file.

| Reference | Content Triggers |
|-----------|------------------|
| `references/github-issues-workflow.md` | Selecting issues from https://github.com/q-user/hh_apply/issues, creating issue-linked branches/worktrees, PR linking, closing issues |
| `references/parallel-agent-worktrees.md` | Coordinating multiple agents in parallel, creating isolated worktrees, integrating branches, cleaning up worktrees |
| `references/task-completion-protocol.md` | Post-merge handoff for `q-user/hh_apply`: close issue, update GitHub Project, prepare repo for next task, cleanup worktrees |

## Conventional Commits

```
<type>[scope]: <description>
```

**Types**: `feat` (MINOR), `fix` (PATCH), `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`

**Breaking change**: Add `!` after type or `BREAKING CHANGE:` in footer.

## Branch Naming

Prefer issue-linked branch names for this repository:

```
issue/123-short-description
agent/123-short-description
feature/123-short-description
fix/123-short-description
release/1.2.0
hotfix/1.2.1-security-patch
```

Use `agent/<issue-number>-<task-slug>` for parallel worktree branches and `feature/<issue-number>-<summary>` or `fix/<issue-number>-<summary>` for the final integration/PR branch.

## GitHub Issues Workflow

For `q-user/hh_apply`, use GitHub Issues as the source of truth: https://github.com/q-user/hh_apply/issues.

### Start from an Issue

Before implementing, identify the issue number and scope:

```bash
gh issue list --repo q-user/hh_apply --state open
gh issue view <issue-number> --repo q-user/hh_apply --comments
```

If `gh` is unavailable, use the issue URL provided by the user or ask for the issue details instead of inventing scope.

### Link Work to the Issue

- Branches should include the issue number: `feature/<issue-number>-<slug>`, `fix/<issue-number>-<slug>`, or `agent/<issue-number>-<slug>`.
- Worktree names should include the issue number: `../worktrees/hh_apply-<issue-number>-<slug>`.
- Commits should mention the issue where useful: `Refs #<issue-number>`.
- PR descriptions should include `Closes #<issue-number>` only when the PR fully resolves the issue; otherwise use `Refs #<issue-number>`.
- Completion notes should summarize validation and remaining risks on the issue or PR.

### Parallel Issue Execution

When multiple agents work in parallel, split by issue or by clearly independent subtask under one issue. Prefer one worktree per issue/subtask:

```bash
git worktree add ../worktrees/hh_apply-<issue-number>-<slug> -b agent/<issue-number>-<slug> <base-branch-or-sha>
```

Each agent must receive the issue URL, issue acceptance criteria, its branch/worktree, and its allowed write scope.

## Parallel Agent Worktrees

Use worktrees whenever multiple agents need to edit the same repository concurrently. The coordinator owns the original worktree; every implementation/research agent gets a dedicated worktree and branch.

### When to Use

- Use for parallel subtasks with disjoint write scopes.
- Use for experiments that should not dirty the main worktree.
- Do not use if all agents only perform read-only exploration.
- Do not let two agents write to the same worktree, branch, or file set.

### Coordinator Protocol

1. Inspect the repo before creating worktrees:

```bash
git status --short
git branch --show-current
git worktree list
```

2. Choose a clean base branch/SHA. Prefer the current task branch if it already exists; otherwise use the updated default branch.
3. Create one branch and one worktree per issue/subtask:

```bash
mkdir -p ../worktrees
git fetch origin
git worktree add ../worktrees/hh_apply-<issue-number>-<task-slug> -b agent/<issue-number>-<task-slug> <base-branch-or-sha>
```

4. Give each agent explicit instructions containing:
   - the issue URL (`https://github.com/q-user/hh_apply/issues/<issue-number>`);
   - issue acceptance criteria;
   - its worktree path;
   - its branch name;
   - allowed files/directories to edit;
   - validation command(s) to run from that worktree;
   - a requirement to report changed files, command output, issue status, and risks.
5. If the editor/agent tooling can only access registered workspace roots, add each worktree directory as a workspace root before delegating code edits. Do not delegate edits into an inaccessible path.
6. After agents finish, integrate only through a coordinator-owned integration branch/worktree:

```bash
git switch -c feature/<issue-number>-<summary> <base-branch-or-sha>
git merge --no-ff agent/<issue-number>-<task-a-slug>
git merge --no-ff agent/<issue-number>-<task-b-slug>
```

Resolve conflicts in the integration branch, run validation, then create the PR from the integration branch.

### Cleanup

Only remove worktrees after their changes are merged, intentionally discarded, or otherwise preserved:

```bash
git worktree remove ../worktrees/hh_apply-<issue-number>-<task-slug>
git branch -d agent/<issue-number>-<task-slug>
git worktree prune
```

Use `git branch -D` only when explicitly discarding unmerged work.

## Hook Detection

Before first commit, detect and install hooks:

```bash
ls lefthook.yml .lefthook.yml captainhook.json .pre-commit-config.yaml .husky/pre-commit 2>/dev/null || echo "No hooks"
```

Install: lefthook.yml -> `lefthook install` | captainhook.json -> `composer install` | .husky/ -> `npm install` | .pre-commit-config.yaml -> `pre-commit install`

In a worktree, verify hook paths with `git rev-parse --git-path hooks` before installing or debugging hooks. Some hook managers install into the shared common git directory; avoid changing hooks unexpectedly for other active worktrees.

## Critical Release Rules

1. **Immutable releases**: Deleted releases permanently block tag reuse; bump version instead.
2. **Multi-branch releases**: Use `--latest=false` from non-default branches.
3. **Pre-release**: Version bumped, CI green, CHANGELOG updated, `git pull` BEFORE `gh release create`.

## PR Merge Requirements

Before merging: all threads resolved, CI checks green (including annotations), branch rebased, commits signed (if required). For signed commits + rebase-only repos, use local `git merge --ff-only`.

Minimum local merge gate before pressing merge:

```bash
git status --short
git fetch origin
git log --oneline --decorate --max-count=12
gh pr checks --watch
gh pr view --json reviewDecision,mergeStateStatus,statusCheckRollup
```

If `gh` is unavailable, state that clearly and use the repository's available CI/status commands instead.

## Verification

```bash
./scripts/verify-git-workflow.sh /path/to/repository
```

---

> **Contributing:** https://github.com/netresearch/git-workflow-skill
