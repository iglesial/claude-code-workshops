---
name: 'git-workflow'
description: 'Manage Git branches, pull, push, sync, and GitHub PRs using git and gh CLI. Use when the user asks about branches, pushing, pulling, syncing, creating PRs, or general git workflow tasks.'
---

# Git Workflow Manager

Manage Git operations and GitHub workflows using `git` and `gh` CLI.

## Prerequisites

- Current directory is a Git repository
- `gh` CLI installed and authenticated (for GitHub operations)
- Remote origin configured

## Operating Principles

- Check current state before making changes (`git status`, `git branch`)
- Use clear, descriptive branch names
- Never force push to main/master without explicit user request
- Prefer rebase for clean history when appropriate
- Always confirm destructive operations

---

## Branch Management

### View Branches

```bash
# List local branches
git branch

# List all branches (local + remote)
git branch -a

# List remote branches only
git branch -r

# Show current branch
git branch --show-current
```

### Create Branches

```bash
# Create and switch to new branch (also pushes to remote)
git checkout -b feature/my-feature
git push -u origin feature/my-feature

# Create branch from specific commit/branch
git checkout -b feature/my-feature origin/main
git push -u origin feature/my-feature

# Create branch without switching
git branch feature/my-feature
```

**Important:** Always push new branches to remote immediately with `-u` flag to set up tracking. This ensures the branch is visible on GitHub right away.

### Branch Naming Convention

| Prefix | Use For | Example |
|--------|---------|---------|
| `feature/` | New features | `feature/model-deployment` |
| `fix/` | Bug fixes | `fix/auth-token-expiry` |
| `chore/` | Maintenance | `chore/update-dependencies` |
| `spike/` | Research/experiments | `spike/test-sagemaker-api` |
| `docs/` | Documentation | `docs/api-reference` |

### Switch Branches

```bash
# Switch to existing branch
git checkout feature/my-feature

# Switch to previous branch
git checkout -

# Switch and discard local changes (careful!)
git checkout -f feature/my-feature
```

### Delete Branches

```bash
# Delete local branch (safe - warns if unmerged)
git branch -d feature/my-feature

# Force delete local branch
git branch -D feature/my-feature

# Delete remote branch
git push origin --delete feature/my-feature
```

---

## Syncing with Remote

### Fetch Updates

```bash
# Fetch all remotes
git fetch --all

# Fetch and prune deleted remote branches
git fetch --all --prune
```

### Pull Changes

```bash
# Pull current branch
git pull

# Pull with rebase (cleaner history)
git pull --rebase

# Pull specific branch
git pull origin main
```

### Push Changes

```bash
# Push current branch
git push

# Push and set upstream (first push of new branch)
git push -u origin feature/my-feature

# Push all branches
git push --all
```

### Sync Branch with Main

```bash
# Option 1: Merge main into feature branch
git checkout feature/my-feature
git merge main

# Option 2: Rebase onto main (cleaner history)
git checkout feature/my-feature
git rebase main

# If conflicts, resolve then:
git rebase --continue
# Or abort:
git rebase --abort
```

---

## Working with Changes

### Stage Changes

```bash
# Stage specific files
git add src/file.ts

# Stage all changes
git add .

# Stage interactively (not recommended in CLI - use VS Code)
# git add -p
```

### Commit

```bash
# Commit with message
git commit -m "Add model deployment feature"

# Commit with multi-line message
git commit -m "$(cat <<'EOF'
Add model deployment feature

- Implement file upload handler
- Add validation for model contracts
- Create deployment progress UI

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"

# Amend last commit (only if not pushed!)
git commit --amend -m "Updated message"
```

### Stash Changes

```bash
# Stash current changes
git stash

# Stash with message
git stash push -m "WIP: model validation"

# List stashes
git stash list

# Apply latest stash
git stash pop

# Apply specific stash
git stash pop stash@{1}

# Drop stash
git stash drop
```

---

## Pull Requests (gh CLI)

### Create PR

```bash
# Create PR interactively
gh pr create

# Create PR with title and body
gh pr create \
  --title "Add model deployment feature" \
  --body "$(cat <<'EOF'
## Summary
Implements model deployment via file upload.

## Changes
- File upload handler
- Model validation
- Deployment progress UI

## Test Plan
- [ ] Upload test model
- [ ] Verify deployment completes
- [ ] Test inference endpoint
EOF
)"

# Create draft PR
gh pr create --draft --title "WIP: Model deployment"

# Create PR and assign reviewers
gh pr create --title "Fix auth bug" --reviewer username1,username2
```

### View PRs

```bash
# List open PRs
gh pr list

# View specific PR
gh pr view 123

# View PR in browser
gh pr view 123 --web

# View current branch's PR
gh pr view
```

### PR Actions

```bash
# Checkout a PR locally
gh pr checkout 123

# Merge PR
gh pr merge 123

# Merge with squash
gh pr merge 123 --squash

# Merge with rebase
gh pr merge 123 --rebase

# Close PR without merging
gh pr close 123

# Mark as ready for review
gh pr ready 123
```

### Review PRs

```bash
# Approve PR
gh pr review 123 --approve

# Request changes
gh pr review 123 --request-changes --body "Please fix X"

# Comment
gh pr review 123 --comment --body "Looks good overall"
```

---

## Common Workflows

### Start New Feature

```bash
# 1. Ensure main is up to date
git checkout main
git pull

# 2. Create feature branch and push to remote immediately
git checkout -b feature/my-feature
git push -u origin feature/my-feature

# 3. Make changes, commit
git add .
git commit -m "Implement feature"

# 4. Push and create PR
git push
gh pr create --title "Add my feature"
```

**Note:** Always push the branch to remote right after creation (step 2). This ensures the branch exists on GitHub before any work begins.

### Update Feature Branch with Main

```bash
# 1. Fetch latest
git fetch origin

# 2. Rebase onto main
git rebase origin/main

# 3. Force push (safe for feature branches)
git push --force-with-lease
```

### Fix Conflicts During Rebase

```bash
# 1. Git will pause at conflicts
# 2. Edit conflicted files, resolve markers
# 3. Stage resolved files
git add <resolved-file>

# 4. Continue rebase
git rebase --continue

# 5. Or abort if needed
git rebase --abort
```

### Quick Fix on Main (hotfix)

```bash
# 1. Create hotfix branch from main
git checkout main
git pull
git checkout -b fix/critical-bug

# 2. Make fix, commit
git add .
git commit -m "Fix critical auth bug"

# 3. Push and create PR
git push -u origin fix/critical-bug
gh pr create --title "Fix critical auth bug"
```

---

## Useful Aliases

Add to `~/.gitconfig`:

```ini
[alias]
  co = checkout
  br = branch
  ci = commit
  st = status
  unstage = reset HEAD --
  last = log -1 HEAD
  lg = log --oneline --graph --decorate -10
```

---

## Troubleshooting

**"Your branch is behind"**
```bash
git pull --rebase
```

**"Cannot push - rejected"**
```bash
# If your branch, force push (careful!)
git push --force-with-lease

# If shared branch, pull first
git pull --rebase
git push
```

**"Detached HEAD"**
```bash
# Create branch from current state
git checkout -b my-branch

# Or return to a branch
git checkout main
```

**"Merge conflict"**
```bash
# See conflicted files
git status

# After resolving
git add <resolved-files>
git commit
```

**Undo last commit (keep changes)**
```bash
git reset --soft HEAD~1
```

**Undo last commit (discard changes)**
```bash
git reset --hard HEAD~1
```