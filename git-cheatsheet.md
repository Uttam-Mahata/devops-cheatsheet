# Git Cheatsheet

> A comprehensive reference for everyday Git workflows — from basics to advanced scenarios.

---

## Table of Contents

- [Setup & Configuration](#setup--configuration)
- [Repository Initialization](#repository-initialization)
- [Staging & Committing](#staging--committing)
- [Branching](#branching)
- [Merging & Rebasing](#merging--rebasing)
- [Remote Repositories](#remote-repositories)
- [Undoing Changes](#undoing-changes)
- [Stashing](#stashing)
- [Tagging](#tagging)
- [Log & Inspection](#log--inspection)
- [Cherry Picking](#cherry-picking)
- [Submodules](#submodules)
- [Bisect (Bug Hunting)](#bisect-bug-hunting)
- [Worktrees](#worktrees)
- [Advanced Scenarios](#advanced-scenarios)

---

## Setup & Configuration

```bash
# Set global identity
git config --global user.name "Jane Doe"
git config --global user.email "jane@example.com"

# Set default editor
git config --global core.editor "vim"

# Set default branch name
git config --global init.defaultBranch main

# Enable color output
git config --global color.ui auto

# View all config
git config --list

# View single config value
git config user.name
```

> **Tip:** Use `--local` instead of `--global` to scope config to the current repo only.

---

## Repository Initialization

```bash
# Initialize a new repo
git init my-project
cd my-project

# Clone an existing repo
git clone https://github.com/user/repo.git

# Clone into a specific folder
git clone https://github.com/user/repo.git my-folder

# Clone a specific branch
git clone -b develop https://github.com/user/repo.git

# Clone with limited history (shallow)
git clone --depth 1 https://github.com/user/repo.git
```

---

## Staging & Committing

```bash
# Check status
git status

# Stage a file
git add README.md

# Stage all changes
git add .

# Stage parts of a file interactively
git add -p README.md

# Unstage a file (keep changes)
git restore --staged README.md

# Commit with message
git commit -m "feat: add homepage layout"

# Stage all tracked files and commit in one step
git commit -am "fix: correct typo in header"

# Amend the last commit (message or content)
git commit --amend -m "fix: correct typo in header (amended)"

# Amend without changing commit message
git commit --amend --no-edit

# Empty commit (useful for triggering CI)
git commit --allow-empty -m "chore: trigger CI pipeline"
```

---

## Branching

```bash
# List local branches
git branch

# List all branches (local + remote)
git branch -a

# Create a new branch
git branch feature/login

# Switch to a branch
git switch feature/login

# Create and switch in one command
git switch -c feature/signup

# Rename current branch
git branch -m feature/sign-up

# Delete a branch (safe — must be merged)
git branch -d feature/login

# Force delete a branch
git branch -D feature/experiment

# Push a branch to remote
git push origin feature/signup

# Track a remote branch
git switch --track origin/feature/signup
```

---

## Merging & Rebasing

### Merging

```bash
# Merge a branch into current
git switch main
git merge feature/login

# Merge with a commit (no fast-forward)
git merge --no-ff feature/login

# Abort an in-progress merge
git merge --abort

# Squash all commits from branch into one
git merge --squash feature/login
git commit -m "feat: add login feature"
```

### Rebasing

```bash
# Rebase current branch onto main
git switch feature/login
git rebase main

# Interactive rebase — last 3 commits
git rebase -i HEAD~3
# Options in editor:  pick | reword | edit | squash | fixup | drop

# Continue after resolving conflicts
git rebase --continue

# Abort rebase
git rebase --abort

# Rebase onto a specific commit
git rebase --onto main feature/base feature/login
```

> **When to use which:**
> - `merge` — preserves history, good for shared branches
> - `rebase` — linear history, good for local feature branches before PR

---

## Remote Repositories

```bash
# List remotes
git remote -v

# Add a remote
git remote add origin https://github.com/user/repo.git

# Rename a remote
git remote rename origin upstream

# Remove a remote
git remote remove upstream

# Fetch changes (don't merge)
git fetch origin

# Fetch all remotes
git fetch --all

# Pull (fetch + merge)
git pull origin main

# Pull with rebase instead of merge
git pull --rebase origin main

# Push to remote
git push origin main

# Push and set upstream
git push -u origin feature/signup

# Force push (use with caution)
git push --force-with-lease origin feature/signup

# Delete a remote branch
git push origin --delete feature/old-branch

# Prune stale remote-tracking branches
git fetch --prune
```

---

## Undoing Changes

```bash
# Discard unstaged changes in a file
git restore README.md

# Discard all unstaged changes
git restore .

# Unstage a file
git restore --staged README.md

# Revert a specific commit (creates new commit)
git revert abc1234

# Revert a merge commit
git revert -m 1 abc1234

# Reset to a previous commit (keep changes staged)
git reset --soft HEAD~1

# Reset to a previous commit (keep changes unstaged)
git reset --mixed HEAD~1

# Reset and discard all changes (DESTRUCTIVE)
git reset --hard HEAD~1

# Recover a deleted branch using reflog
git reflog
git checkout -b recovered-branch abc1234
```

> **Reset modes at a glance:**
>
> | Flag      | Working Dir | Staging Area | Commit |
> |-----------|-------------|--------------|--------|
> | `--soft`  | unchanged   | unchanged    | moved  |
> | `--mixed` | unchanged   | cleared      | moved  |
> | `--hard`  | cleared     | cleared      | moved  |

---

## Stashing

```bash
# Stash current changes
git stash

# Stash with a description
git stash push -m "WIP: login form validation"

# Stash including untracked files
git stash push -u

# List stashes
git stash list

# Apply most recent stash (keep it in list)
git stash apply

# Apply a specific stash
git stash apply stash@{2}

# Pop most recent stash (apply + remove)
git stash pop

# Drop a specific stash
git stash drop stash@{1}

# Clear all stashes
git stash clear

# Create a branch from a stash
git stash branch feature/stashed-work stash@{0}
```

---

## Tagging

```bash
# List tags
git tag

# Create a lightweight tag
git tag v1.0.0

# Create an annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag a specific commit
git tag -a v0.9.0 abc1234 -m "Beta release"

# Push a single tag
git push origin v1.0.0

# Push all tags
git push origin --tags

# Delete a local tag
git tag -d v1.0.0

# Delete a remote tag
git push origin --delete v1.0.0

# Checkout a tag (detached HEAD)
git checkout v1.0.0
```

---

## Log & Inspection

```bash
# View commit log
git log

# One-line log
git log --oneline

# Log with graph
git log --oneline --graph --all

# Log for a specific file
git log -- README.md

# Log between two commits
git log abc1234..def5678

# Show changes in a commit
git show abc1234

# Show diff of staged changes
git diff --staged

# Show diff between branches
git diff main..feature/login

# Show who changed each line (blame)
git blame README.md

# Show blame with line range
git blame -L 10,20 README.md

# Search commit messages
git log --grep="fix:"

# Find commits that added/removed a string
git log -S "getUserById"

# Count commits per author
git shortlog -sn
```

---

## Cherry Picking

```bash
# Apply a specific commit to current branch
git cherry-pick abc1234

# Cherry-pick without committing (stage only)
git cherry-pick --no-commit abc1234

# Cherry-pick a range of commits
git cherry-pick abc1234^..def5678

# Continue after resolving conflicts
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

**Scenario:** You fixed a critical bug on `feature/auth` and need it on `main` now:

```bash
git log feature/auth --oneline
# → abc1234 fix: patch null pointer in token refresh

git switch main
git cherry-pick abc1234
```

---

## Submodules

```bash
# Add a submodule
git submodule add https://github.com/user/lib.git libs/lib

# Initialize submodules after cloning
git submodule update --init --recursive

# Update all submodules to latest
git submodule update --remote

# Remove a submodule
git submodule deinit libs/lib
git rm libs/lib
rm -rf .git/modules/libs/lib

# Clone a repo with all submodules
git clone --recurse-submodules https://github.com/user/repo.git
```

---

## Bisect (Bug Hunting)

```bash
# Start bisect session
git bisect start

# Mark current commit as bad
git bisect bad

# Mark a known good commit
git bisect good v1.0.0

# Git will checkout a commit — test it, then:
git bisect good   # if the bug is not present
git bisect bad    # if the bug is present

# Git narrows down automatically — repeat until found
# → "abc1234 is the first bad commit"

# Reset bisect session
git bisect reset

# Automate with a test script
git bisect run ./run-tests.sh
```

---

## Worktrees

```bash
# Add a worktree for a branch
git worktree add ../hotfix-branch hotfix/critical

# List worktrees
git worktree list

# Remove a worktree
git worktree remove ../hotfix-branch

# Prune stale worktree references
git worktree prune
```

**Scenario:** Work on a hotfix while keeping your current feature branch intact:

```bash
git worktree add ../my-repo-hotfix hotfix/payment-crash
cd ../my-repo-hotfix
# fix the bug, commit, push
cd ../my-repo
# continue feature work, uninterrupted
```

---

## Advanced Scenarios

### Scenario 1 — Squash all commits on a feature branch before merging

```bash
git switch main
git merge --squash feature/user-dashboard
git commit -m "feat: add user dashboard"
```

### Scenario 2 — Undo a pushed commit without rewriting history

```bash
git revert abc1234
git push origin main
```

### Scenario 3 — Split the last commit into two

```bash
git reset HEAD~1                 # unstage the last commit
git add src/auth.js              # stage first logical change
git commit -m "feat: add JWT auth"
git add src/middleware.js        # stage second
git commit -m "feat: add auth middleware"
```

### Scenario 4 — Move commits to a new branch (already committed to wrong branch)

```bash
git branch feature/new-work      # create branch at current HEAD
git reset --hard HEAD~3          # roll back main (or wherever you are)
git switch feature/new-work      # now the 3 commits live here
```

### Scenario 5 — Recover a deleted branch

```bash
git reflog                       # find the last commit hash of the deleted branch
git checkout -b recovered abc1234
```

### Scenario 6 — Interactively clean up last N commits before a PR

```bash
git rebase -i HEAD~5
# In editor:
#   pick  → keep as-is
#   reword → change commit message
#   squash → merge into previous commit
#   fixup  → like squash but discard message
#   drop   → delete commit
```

### Scenario 7 — Apply a `.gitignore` to already-tracked files

```bash
echo "*.log" >> .gitignore
git rm -r --cached .              # untrack all
git add .
git commit -m "chore: apply gitignore to tracked files"
```

### Scenario 8 — Find which branch contains a commit

```bash
git branch --contains abc1234
git branch -r --contains abc1234  # remote branches
```

### Scenario 9 — Compare a file between two branches

```bash
git diff main..feature/login -- src/api/auth.js
```

### Scenario 10 — Archive a repo snapshot as tar/zip

```bash
git archive --format=zip HEAD > snapshot.zip
git archive --format=tar.gz HEAD > snapshot.tar.gz
```

---

## Quick Reference Card

| Task                        | Command                                  |
|-----------------------------|------------------------------------------|
| New branch + switch         | `git switch -c branch-name`              |
| Stage all + commit          | `git commit -am "message"`               |
| Push + set upstream         | `git push -u origin branch-name`         |
| Undo last commit (keep)     | `git reset --soft HEAD~1`                |
| Discard all local changes   | `git restore .`                          |
| Pull with rebase            | `git pull --rebase origin main`          |
| Stash + pop                 | `git stash` / `git stash pop`            |
| View pretty log             | `git log --oneline --graph --all`        |
| Search commit history       | `git log -S "keyword"`                   |
| Safe force push             | `git push --force-with-lease`            |

---

> Made with Git — the tool that never forgets (unless you `reset --hard`).
