# AGENTS.md — Instructions for AI Coding Agents

This is a **fork** of [valhalla/valhalla](https://github.com/valhalla/valhalla)
maintained by Ramon Smits. See [MODIFICATIONS.md](./MODIFICATIONS.md) for
the rationale behind fork-specific changes.

## Branching Model

This repository uses three kinds of branches with strict rules:

### `master` — upstream mirror

- **Must equal `upstream/master` exactly.** No fork-specific commits.
- Updated by fast-forward only: `git fetch upstream && git merge --ff-only upstream/master`.
- Any commit added here by mistake breaks the rebase workflow.

### `downstream` — the fork's default branch

- Built from `master` by `--no-ff` merges of each `feature/*` branch.
- Also contains fork-specific docs: `MODIFICATIONS.md`, `AGENTS.md`,
  fork notice at the top of `README.md`.
- This is what consumers clone and pin.

### `feature/*` — one per independent change

- Each branches from `master`.
- Each holds a single coherent change (may be multiple commits).
- Each has an open **draft PR** into `downstream`, acting as tracking
  for work-in-progress review.
- **Branches are kept alive** after merging into `downstream` — they
  may still receive new iteration commits.

## Workflow Order (IMPORTANT)

**Always create the draft PR BEFORE merging** into `downstream`:

1. Branch from `master`
2. Commit your changes
3. Push feature branch
4. **Open draft PR: `feature/<name>` → `downstream`**
5. Merge into `downstream` with `--no-ff` (closes PR as merged)
6. For next iteration: new commits on feature branch → new draft PR → merge

If you merge first, the PR will fail with "No commits between" because
the feature content is already in the base. Don't make this mistake.

## Rules for AI Agents

### NEVER rewrite pushed commits

Once a commit is pushed to `origin`, it is immutable. This means:

- **No `git rebase`** on pushed branches.
- **No `git commit --amend`** on pushed commits.
- **No `git push --force`** or `--force-with-lease` on any branch.
- **No deleting remote branches** without explicit user approval.

If you need to change a pushed commit, add a new commit that reverts or
modifies the behavior.

### NEVER fast-forward merges into `downstream`

Always use `git merge --no-ff`. This preserves the branching structure
and makes feature origins visible in `git log --graph`.

### Sync from upstream — the right way

```bash
git fetch upstream
git checkout master
git merge --ff-only upstream/master   # fail loudly if not fast-forward
git push origin master

# Do NOT rebase feature branches (they're pushed).
# Instead, to bring a feature branch current with new master:
#   option 1: merge master INTO the feature branch (adds merge commit)
#   option 2: create a new feature/*-v2 branch from current master,
#             cherry-pick the original commits, open a new draft PR
```

### Creating a new feature branch

```bash
git checkout master
git pull --ff-only
git checkout -b feature/<short-name>
# ... make commits ...
git push -u origin feature/<short-name>
gh pr create --draft --base downstream --title "..." --body "..."
```

### Merging a feature branch into `downstream`

```bash
# PR must already exist (see workflow order above)
git checkout downstream
git pull --ff-only
git merge --no-ff feature/<short-name>
# resolve conflicts if any — new costing options all touch the same
# additive blocks, so conflicts are expected and trivial
git push origin downstream
# The PR auto-closes as merged. Feature branch stays alive.
```

### Adding a fix/iteration to an existing feature

```bash
git checkout feature/<short-name>
git pull --ff-only
# ... new commit(s) ...
git push origin feature/<short-name>

# Open a NEW draft PR for this iteration (previous PR is closed/merged)
gh pr create --draft --base downstream --title "..." --body "..."

# Then merge as before:
git checkout downstream
git pull --ff-only
git merge --no-ff feature/<short-name>
git push origin downstream
```

## Code Conventions

- **Additive only**: new costing options should default to no-op values
  so requests without them behave identically to upstream.
- **High proto field numbers** (97+) to avoid collisions with upstream's
  future additions.
- **Minimize touchpoints**: prefer one helper method called from
  existing hooks over editing many files.
- **No line renumbering**: add new lines, don't reformat existing ones.

## Consumer Projects

This fork is consumed by:

- **p2000** (Dutch fire service dispatch system): clones the `downstream`
  branch into a Docker image. See the project's `Containerfile.valhalla`.

When changing the public API (adding/renaming costing options,
changing defaults), check with the maintainer before merging into
`downstream` — consumer projects pin to the branch name, not a tag.
