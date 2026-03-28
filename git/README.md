# Git Cheatsheet

Git is a distributed version control system that tracks changes to files, enables collaboration across teams, and supports non-linear development through branching and merging. Every Git repository contains the full project history, allowing developers to work offline, branch freely, and merge changes with precision. Whether you're managing solo projects or coordinating across hundreds of contributors, Git provides the primitives to do it reliably.

---

## Table of Contents

1. [Setup & Configuration](#setup--configuration)
2. [Repository Init](#repository-init)
3. [Staging & Committing](#staging--committing)
4. [Branching](#branching)
5. [Merging & Rebasing](#merging--rebasing)
6. [Remote Repositories](#remote-repositories)
7. [Undoing Changes](#undoing-changes)
8. [Stashing](#stashing)
9. [Tagging](#tagging)
10. [Log & Inspection](#log--inspection)
11. [Cherry-Picking](#cherry-picking)
12. [Submodules](#submodules)
13. [Bisect](#bisect)
14. [Worktrees](#worktrees)
15. [Common Patterns & Tips](#common-patterns--tips)

---

## Setup & Configuration

Git configuration is stored at three scopes: `--system` (all users on the machine), `--global` (current user), and `--local` (current repository). Values at narrower scopes override broader ones.

### `git config`

Sets or reads configuration values. Config is stored in `.git/config` (local), `~/.gitconfig` (global), or `/etc/gitconfig` (system).

| Flag / Option | Description |
|---|---|
| `--global` | Apply setting to the current user's global config |
| `--local` | Apply setting to the current repository only |
| `--system` | Apply setting system-wide |
| `--list` | List all configuration values currently in effect |
| `--unset` | Remove a configuration key |
| `--edit` | Open the config file in the default editor |

```bash
# Set identity (required before committing)
git config --global user.name "Ada Lovelace"
git config --global user.email "ada@example.com"

# Set default branch name to 'main'
git config --global init.defaultBranch main

# Use VS Code as the default editor
git config --global core.editor "code --wait"

# Enable coloured output
git config --global color.ui auto

# Set default pull strategy to rebase
git config --global pull.rebase true

# List all effective config values
git config --list --show-origin
```

### `git config` — Aliases

Aliases let you create short forms for frequently used commands.

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset HEAD~1 --mixed"
```

---

## Repository Init

### `git init`

Initialises a new, empty Git repository in the current directory (or in a specified path). Creates the hidden `.git/` directory that stores all version history and configuration.

| Flag / Option | Description |
|---|---|
| `--bare` | Create a bare repository (no working tree; used for remotes/servers) |
| `-b <branch>` | Set the name of the initial branch (Git ≥ 2.28) |

```bash
# Initialise in current directory
git init

# Initialise with 'main' as the initial branch
git init -b main my-project

# Create a bare repo for use as a server remote
git init --bare /srv/repos/project.git
```

### `git clone`

Creates a local copy of an existing repository (remote or local). Downloads all history, branches, and tags.

| Flag / Option | Description |
|---|---|
| `--depth <n>` | Shallow clone: only fetch the last `n` commits |
| `--branch <name>` | Check out a specific branch or tag after cloning |
| `--single-branch` | Only fetch history for the checked-out branch |
| `--recurse-submodules` | Automatically initialise and update submodules |
| `--filter=blob:none` | Partial clone — omit file blobs until needed (blobless clone) |

```bash
# Basic clone
git clone https://github.com/org/repo.git

# Clone into a specific directory name
git clone https://github.com/org/repo.git my-local-name

# Shallow clone for CI (faster)
git clone --depth 1 https://github.com/org/repo.git

# Clone a specific branch
git clone --branch release/2.0 --single-branch https://github.com/org/repo.git
```

---

## Staging & Committing

Git has a three-tree architecture: the working directory (what you see), the staging area / index (what will go into the next commit), and the HEAD (the last commit).

### `git add`

Stages changes (new files, modifications, deletions) to the index, readying them for a commit.

| Flag / Option | Description |
|---|---|
| `-A` / `--all` | Stage all changes including deletions |
| `-p` / `--patch` | Interactively select hunks to stage |
| `-u` | Stage modifications and deletions, but not untracked files |
| `-n` / `--dry-run` | Show what would be staged without doing it |

```bash
git add file.txt                # Stage a single file
git add src/                    # Stage everything inside src/
git add -A                      # Stage all changes
git add -p                      # Interactively choose hunks
```

### `git commit`

Records staged changes as a new snapshot in the repository history.

| Flag / Option | Description |
|---|---|
| `-m "<msg>"` | Provide the commit message inline |
| `-a` | Automatically stage tracked modified/deleted files before committing |
| `--amend` | Modify the most recent commit (message and/or content) |
| `--no-edit` | Amend without changing the commit message |
| `-S` | Sign the commit with a GPG key |
| `--allow-empty` | Create a commit with no changes (useful for triggering CI) |

```bash
git commit -m "feat: add user authentication"
git commit -am "fix: correct off-by-one in loop"

# Amend last commit message
git commit --amend -m "fix: correct off-by-one error in pagination loop"

# Add forgotten file to last commit without changing message
git add forgotten.txt
git commit --amend --no-edit
```

### `git status`

Shows the state of the working directory and staging area.

```bash
git status          # Verbose output
git status -s       # Short/porcelain format
git status -sb      # Short format with branch info
```

### `git diff`

Shows differences between working tree, index, and commits.

```bash
git diff                    # Unstaged changes vs index
git diff --staged           # Staged changes vs last commit
git diff HEAD               # All changes vs last commit
git diff main..feature      # Difference between two branches
git diff --stat             # Summary of changed files
```

---

## Branching

Branches are lightweight, movable pointers to commits. Creating and switching branches is nearly instantaneous in Git.

### `git branch`

Lists, creates, renames, or deletes branches.

| Flag / Option | Description |
|---|---|
| `-a` | List both local and remote-tracking branches |
| `-r` | List remote-tracking branches only |
| `-d <branch>` | Delete a branch (safe: refuses if unmerged) |
| `-D <branch>` | Force-delete a branch regardless of merge status |
| `-m <old> <new>` | Rename a branch |
| `--merged` | List branches merged into the current branch |
| `--no-merged` | List branches not yet merged |
| `-v` | Show last commit on each branch |

```bash
git branch                          # List local branches
git branch -a                       # List all branches
git branch feature/login            # Create new branch
git branch -d feature/login         # Delete merged branch
git branch -D hotfix/temp           # Force delete
git branch -m old-name new-name     # Rename branch
git branch --merged main            # Branches already in main
```

### `git switch` / `git checkout`

`git switch` (Git ≥ 2.23) is the preferred command for changing branches; `git checkout` is the older, multipurpose alternative.

| Flag / Option | Description |
|---|---|
| `-c <branch>` | Create and switch to a new branch (`switch`) |
| `-b <branch>` | Create and switch to a new branch (`checkout`) |
| `--detach` | Switch to a commit in detached HEAD state |

```bash
git switch main                     # Switch to existing branch
git switch -c feature/dashboard     # Create and switch
git switch -                        # Switch to previous branch

git checkout feature/login          # Older equivalent
git checkout -b hotfix/bug-42       # Create and switch (old style)
```

---

## Merging & Rebasing

### `git merge`

Integrates changes from one branch into the current branch. By default creates a merge commit; if the merge is fast-forwardable and `--ff` is in effect, it simply moves the pointer.

| Flag / Option | Description |
|---|---|
| `--no-ff` | Always create a merge commit even if fast-forward is possible |
| `--squash` | Squash all source commits into staged changes; commit separately |
| `--abort` | Abort a merge in progress and restore pre-merge state |
| `--continue` | Continue after resolving conflicts |
| `-X ours` / `-X theirs` | Automatically resolve conflicts using one side |

```bash
git switch main
git merge feature/login             # Merge with fast-forward if possible
git merge --no-ff feature/login     # Always create a merge commit
git merge --squash feature/login && git commit -m "feat: login feature"
git merge --abort                   # Cancel an in-progress merge
```

### `git rebase`

Re-applies commits from the current branch on top of another base, rewriting history to produce a linear sequence.

| Flag / Option | Description |
|---|---|
| `-i` / `--interactive` | Open interactive editor to reorder, squash, edit, or drop commits |
| `--onto <newbase>` | Rebase onto a different base than the upstream |
| `--abort` | Abort and return to pre-rebase state |
| `--continue` | Continue after resolving a conflict |
| `--autosquash` | Automatically squash commits prefixed with `fixup!` or `squash!` |

```bash
git switch feature/login
git rebase main                     # Rebase feature onto main

# Interactive rebase: edit last 3 commits
git rebase -i HEAD~3

# Rebase a range of commits onto a different base
git rebase --onto main server feature
```

> **Warning:** Never rebase commits that have been pushed to a shared remote branch, as it rewrites history and forces collaborators to re-synchronise.

---

## Remote Repositories

### `git remote`

Manages connections to remote repositories.

| Flag / Option | Description |
|---|---|
| `-v` | Show URLs for all remotes |
| `add <name> <url>` | Add a new remote |
| `remove <name>` | Remove a remote |
| `rename <old> <new>` | Rename a remote |
| `set-url <name> <url>` | Change the URL of an existing remote |

```bash
git remote -v                                       # List remotes
git remote add origin https://github.com/org/repo.git
git remote add upstream https://github.com/upstream/repo.git
git remote set-url origin git@github.com:org/repo.git
git remote remove upstream
```

### `git fetch`

Downloads objects and refs from a remote without integrating them into the working tree.

```bash
git fetch origin                    # Fetch all branches from origin
git fetch --all                     # Fetch from all remotes
git fetch --prune                   # Remove stale remote-tracking refs
git fetch origin main               # Fetch only the main branch
```

### `git pull`

Fetches from a remote and immediately integrates (merge or rebase) into the current branch.

| Flag / Option | Description |
|---|---|
| `--rebase` | Use rebase instead of merge when integrating |
| `--ff-only` | Only fast-forward; fail if a merge commit would be needed |
| `--no-commit` | Fetch and merge but don't auto-commit |

```bash
git pull origin main
git pull --rebase origin main       # Preferred for clean linear history
git pull --ff-only                  # Safe pull that refuses to merge
```

### `git push`

Uploads local commits to a remote repository.

| Flag / Option | Description |
|---|---|
| `-u` / `--set-upstream` | Set tracking relationship for the branch |
| `--force-with-lease` | Safe force-push: fails if remote has new commits you haven't fetched |
| `--force` / `-f` | Force push (overwrites remote history — use carefully) |
| `--tags` | Push all local tags |
| `--delete <branch>` | Delete a remote branch |
| `--dry-run` | Show what would be pushed |

```bash
git push -u origin feature/login        # First push; set upstream
git push                                # Subsequent pushes
git push --force-with-lease             # Safe force-push after rebase
git push origin --delete old-branch     # Delete remote branch
git push origin --tags                  # Push all tags
```

---

## Undoing Changes

### `git restore`

Discards changes in the working tree or unstages files from the index (Git ≥ 2.23).

| Flag / Option | Description |
|---|---|
| `--staged` | Unstage a file (move from index back to working tree) |
| `--source=<tree>` | Restore file content from a specific commit or tree |

```bash
git restore file.txt                    # Discard working-tree changes
git restore --staged file.txt           # Unstage a file
git restore --source=HEAD~2 file.txt    # Restore file from 2 commits ago
git restore .                           # Discard all unstaged changes
```

### `git reset`

Moves the HEAD (and branch pointer) to a different commit, optionally modifying the index and working tree.

| Mode | Effect on Index | Effect on Working Tree |
|---|---|---|
| `--soft` | Unchanged | Unchanged |
| `--mixed` (default) | Reset to commit | Unchanged |
| `--hard` | Reset to commit | Reset to commit |

```bash
git reset HEAD~1                        # Undo last commit; keep changes staged (--soft is even softer)
git reset --mixed HEAD~1               # Undo last commit; unstage changes
git reset --hard HEAD~1                # Undo last commit; discard all changes
git reset --hard origin/main           # Reset to match remote
```

### `git revert`

Creates a new commit that is the inverse of a given commit, without altering existing history. Safe for shared branches.

```bash
git revert HEAD                         # Revert the last commit
git revert abc1234                      # Revert a specific commit
git revert HEAD~3..HEAD                 # Revert a range of commits
git revert --no-commit HEAD             # Stage the revert without committing
```

---

## Stashing

Stashing saves your uncommitted changes (staged and unstaged) to a temporary storage stack so you can switch context without committing unfinished work.

### `git stash`

| Subcommand | Description |
|---|---|
| `push -m "<msg>"` | Save stash with a descriptive message |
| `list` | List all stashes |
| `pop` | Apply the most recent stash and remove it from the stack |
| `apply stash@{n}` | Apply a stash without removing it |
| `drop stash@{n}` | Delete a specific stash |
| `clear` | Delete all stashes |
| `show -p stash@{n}` | Show the diff of a stash |
| `branch <branch>` | Create a new branch from a stash |
| `-u` / `--include-untracked` | Also stash untracked files |

```bash
git stash                                   # Quick stash
git stash push -m "WIP: half-done refactor" -u
git stash list
git stash pop                               # Apply and drop latest
git stash apply stash@{2}                   # Apply without dropping
git stash drop stash@{0}
git stash branch fix/experiment stash@{1}  # Branch from stash
```

---

## Tagging

Tags mark specific commits as significant, typically for release versions.

### `git tag`

| Flag / Option | Description |
|---|---|
| `-a <name>` | Create an annotated tag (stores tagger, date, message) |
| `-m "<msg>"` | Provide the tag message |
| `-s` | Sign the tag with GPG |
| `-d <name>` | Delete a tag locally |
| `-l "<pattern>"` | List tags matching a pattern |
| `--sort=-version:refname` | Sort tags by version |

```bash
git tag v1.0.0                              # Lightweight tag
git tag -a v1.2.0 -m "Release 1.2.0"       # Annotated tag
git tag -a v1.1.0 abc1234 -m "Backfill"    # Tag an older commit
git tag -l "v1.*"                           # List v1.x tags
git tag -d v1.0.0-rc1                       # Delete local tag
git push origin v1.2.0                      # Push single tag
git push origin --tags                      # Push all tags
git push origin --delete v1.0.0-rc1         # Delete remote tag
```

---

## Log & Inspection

### `git log`

Displays the commit history with many formatting and filtering options.

| Flag / Option | Description |
|---|---|
| `--oneline` | Compact one-line format |
| `--graph` | ASCII art branch graph |
| `--decorate` | Show branch/tag pointers |
| `--all` | Show history from all branches |
| `--author="<name>"` | Filter by author |
| `--since` / `--until` | Filter by date |
| `-p` | Show patches (diffs) with each commit |
| `--stat` | Show file change statistics |
| `-n <num>` | Limit to last `n` commits |
| `--follow <file>` | Follow renames of a file |
| `--no-merges` | Exclude merge commits |
| `--grep="<pattern>"` | Filter commits by message |

```bash
git log --oneline --graph --decorate --all
git log --author="Ada" --since="2 weeks ago"
git log -p --follow -- src/auth.ts          # History of a file
git log --stat -n 10                         # Last 10 with stats
git log --grep="fix:" --oneline             # Find fix commits
git log main..feature                        # Commits in feature not in main
```

### `git show`

Shows the details and diff of a commit, tag, or other object.

```bash
git show HEAD                   # Show latest commit
git show v1.2.0                 # Show tagged commit
git show abc1234:src/main.go    # Show file content at a commit
```

### `git blame`

Annotates each line of a file with the commit and author that last modified it.

```bash
git blame src/auth.ts
git blame -L 20,40 src/auth.ts  # Only lines 20–40
git blame --follow src/auth.ts  # Follow renames
```

### `git shortlog`

Summarises commit history grouped by author.

```bash
git shortlog -sn            # Count commits per author, sorted
git shortlog -sn --no-merges
```

---

## Cherry-Picking

Cherry-picking applies the changes introduced by specific commits onto the current branch, creating new commits with the same changes but different SHAs.

### `git cherry-pick`

| Flag / Option | Description |
|---|---|
| `-e` / `--edit` | Edit the commit message before applying |
| `-n` / `--no-commit` | Apply changes without committing |
| `-x` | Append a line noting the source commit SHA |
| `--abort` | Abort the cherry-pick in progress |
| `--continue` | Continue after resolving conflicts |
| `<sha1>..<sha2>` | Pick a range of commits |

```bash
git cherry-pick abc1234                     # Apply a single commit
git cherry-pick abc1234 def5678             # Apply multiple commits
git cherry-pick abc1234..def5678            # Apply a range
git cherry-pick -n abc1234                  # Stage changes only
git cherry-pick -x abc1234                  # Note original SHA in message
```

---

## Submodules

Submodules let you embed one Git repository inside another, pinning it to a specific commit. Useful for tracking external dependencies as source.

### `git submodule`

| Subcommand | Description |
|---|---|
| `add <url> <path>` | Add a repository as a submodule |
| `init` | Register submodules listed in `.gitmodules` |
| `update --init --recursive` | Initialise and check out all submodules |
| `foreach <cmd>` | Run a shell command in each submodule |
| `status` | Show current SHA of each submodule |
| `sync` | Update remote URLs from `.gitmodules` |

```bash
# Add a submodule
git submodule add https://github.com/org/lib.git vendor/lib

# Clone a repo that has submodules
git clone --recurse-submodules https://github.com/org/repo.git

# Initialise submodules after a plain clone
git submodule update --init --recursive

# Update all submodules to the latest remote commit
git submodule update --remote --merge

# Run a command in every submodule
git submodule foreach 'git pull origin main'

# Remove a submodule cleanly
git submodule deinit vendor/lib
git rm vendor/lib
rm -rf .git/modules/vendor/lib
```

---

## Bisect

`git bisect` uses a binary search algorithm to find the commit that introduced a bug. You mark known good and bad commits; Git checks out the midpoint for you to test.

### `git bisect`

| Subcommand | Description |
|---|---|
| `start` | Begin a bisect session |
| `bad [<ref>]` | Mark a commit as bad (contains the bug) |
| `good <ref>` | Mark a commit as good (before the bug) |
| `skip` | Skip a commit that can't be tested |
| `reset` | End the session and return to the original HEAD |
| `run <script>` | Automate with a script that exits 0 (good) or 1 (bad) |

```bash
git bisect start
git bisect bad                      # Current HEAD is broken
git bisect good v1.3.0              # v1.3.0 was working

# Git checks out midpoint; test and mark:
git bisect good                     # or:
git bisect bad

# When finished, Git prints the first bad commit
git bisect reset                    # Return to original branch

# Automated bisect with a test script
git bisect start HEAD v1.3.0
git bisect run ./scripts/test.sh    # 0 = good, non-zero = bad
```

---

## Worktrees

Worktrees allow you to check out multiple branches simultaneously into separate working directories, all sharing the same `.git` repository. No need to stash or commit when switching tasks.

### `git worktree`

| Subcommand | Description |
|---|---|
| `add <path> <branch>` | Create a new worktree at `path` checked out to `branch` |
| `list` | List all linked worktrees |
| `remove <path>` | Remove a worktree |
| `prune` | Remove stale worktree references |

```bash
# Create a worktree for a hotfix while on feature branch
git worktree add ../hotfix-42 hotfix/issue-42

# Create a worktree with a new branch
git worktree add -b feature/new-ui ../new-ui main

# List all worktrees
git worktree list

# Remove a worktree when done
git worktree remove ../hotfix-42
git worktree prune                  # Clean up stale refs
```

---

## Common Patterns & Tips

1. **Write atomic commits.** Each commit should represent one logical change. This makes `git bisect`, `git revert`, and code review far more tractable. Prefer multiple small commits over one large commit.

2. **Use `--force-with-lease` instead of `--force`.** When force-pushing after a rebase, `--force-with-lease` checks that no one else has pushed to the remote branch since your last fetch. This prevents accidentally overwriting a colleague's work.

3. **Prefer `git pull --rebase` for syncing.** Running `git pull --rebase` keeps your local commits on top of the remote commits without creating noisy merge commits, resulting in a cleaner, linear history.

4. **Keep `.gitignore` comprehensive and committed early.** A well-crafted `.gitignore` prevents build artefacts, editor configs, credentials, and OS files from polluting the repository. Use `git check-ignore -v <file>` to debug unexpected ignoring behaviour.

5. **Leverage interactive rebase (`git rebase -i`) before pushing.** Before sharing a feature branch, clean up your commits: squash fixup commits, rewrite unclear messages, reorder related changes. This produces a readable history for reviewers.

6. **Use `git reflog` as a safety net.** The reflog records every movement of HEAD (including resets and rebases) for at least 90 days. If you accidentally lose commits, `git reflog` lets you find and restore them via `git reset --hard <sha>`.

7. **Sign commits and tags for sensitive projects.** Use `git commit -S` and `git tag -s` to GPG-sign your work. Configure `git config --global commit.gpgsign true` to sign automatically.

8. **Adopt a consistent branch naming convention.** Prefixes like `feature/`, `fix/`, `chore/`, `docs/`, and `release/` make branch purpose immediately clear and integrate cleanly with CI/CD automation and project management tooling.

9. **Use `git stash push -m` with descriptive messages.** Anonymous stashes are hard to identify after a few days. Always provide a message with `git stash push -m "context: what you were doing"` to make `git stash list` useful.

10. **Regularly prune stale remote-tracking branches.** Run `git fetch --prune` or configure `git config --global fetch.prune true` so that deleted remote branches are automatically removed from your local remote-tracking refs, keeping `git branch -r` output clean.

11. **Use worktrees for parallel workstreams.** Instead of stashing changes or juggling multiple clones, use `git worktree add` to check out a different branch in a sibling directory. Each worktree is independent and shares the same object store.

12. **Tag every release — annotated, not lightweight.** Annotated tags (`git tag -a`) store authorship, timestamp, and a message, making them proper release artefacts. Lightweight tags are just pointers and lack this metadata.
