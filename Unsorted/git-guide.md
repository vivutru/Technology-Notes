# Git: A Practical Guide

Git is a distributed version control system. It tracks changes to files over time, lets many people work on the same project, and lets you move back and forth through history safely.

---

## 1. Core Concepts

| Concept | Meaning |
|---|---|
| **Repository (repo)** | A project folder plus its full history, stored in a hidden `.git` directory |
| **Commit** | A snapshot of your project with a message, author, and timestamp |
| **Branch** | A movable pointer to a commit; lets you work on features in isolation |
| **HEAD** | Pointer to the commit/branch you currently have checked out |
| **Working directory** | The files you see and edit |
| **Staging area (index)** | Where you assemble the next commit |
| **Remote** | A copy of the repo hosted elsewhere (GitHub, GitLab, etc.), usually named `origin` |
| **Clone / Fork** | A local copy of a repo / a personal server-side copy of someone else's |

### The three areas

```
Working directory  --git add-->  Staging area  --git commit-->  Repository
        ^                                                            |
        +------------------------ git checkout / switch -------------+
```

Files move through states: **untracked → modified → staged → committed**.

---

## 2. Setup

```bash
git --version

git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"     # or vim, nano, etc.
git config --global pull.rebase false              # or true, see section 7

git config --list                                  # view settings
```

Config levels: `--system` (all users), `--global` (your user), `--local` (this repo only, default).

---

## 3. Starting a Repository

```bash
git init                          # turn current folder into a repo
git clone <url>                   # copy a remote repo
git clone <url> my-folder         # into a specific folder
git clone --depth 1 <url>         # shallow clone (latest commit only)
```

---

## 4. The Basic Workflow

```bash
git status                        # what's changed?
git add file.txt                  # stage one file
git add src/                      # stage a folder
git add -p                        # stage interactively, hunk by hunk
git add .                         # stage everything in current dir

git commit -m "Add login form"    # commit staged changes
git commit                        # opens editor for a longer message
git commit -am "Fix typo"         # stage tracked files + commit (skips new files)
```

### Inspect changes

```bash
git diff                          # unstaged changes
git diff --staged                 # staged changes
git diff main..feature            # between two branches
git log                           # full history
git log --oneline --graph --all   # compact visual history
git log -p file.txt               # history with patches for one file
git show <commit>                 # details of one commit
git blame file.txt                # who changed each line
```

### Writing good commit messages

```
Short summary in imperative mood (50 chars or less)

Optional body explaining WHY the change was made, not just what.
Wrap at roughly 72 characters.

Refs: #123
```

Good: `Fix null pointer when cart is empty`. Bad: `fixed stuff`.

**Commit small, focused changes.** Each commit should do one logical thing.

---

## 5. Branching and Merging

```bash
git branch                        # list local branches
git branch -a                     # include remote branches
git branch feature/login          # create
git switch feature/login          # move to it
git switch -c feature/login       # create and switch
git branch -d feature/login       # delete (safe, only if merged)
git branch -D feature/login       # force delete
git branch -m new-name            # rename current branch
```

(`git checkout` does the same job as `switch` but is older and does more things; both work.)

### Merge

```bash
git switch main
git merge feature/login
```

| Merge type | When | Result |
|---|---|---|
| **Fast-forward** | `main` has no new commits since branching | Pointer simply moves forward, no merge commit |
| **Three-way** | Both branches have new commits | Creates a merge commit |
| `--no-ff` | Force a merge commit | Keeps branch history visible |
| `--squash` | Want one commit from many | Stages combined changes; you commit manually |

### Resolving conflicts

When Git can't combine changes automatically:

```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> feature/login
```

1. Open the conflicted files and edit to the final content, removing the markers.
2. `git add <file>` to mark resolved.
3. `git commit` (or `git merge --continue`).
4. To bail out: `git merge --abort`.

Helpful: `git status` lists conflicted files; `git mergetool` launches a visual tool.

---

## 6. Working with Remotes

```bash
git remote -v                          # list remotes
git remote add origin <url>            # add one
git remote set-url origin <new-url>    # change URL

git fetch                              # download changes, don't merge
git pull                               # fetch + merge (or rebase)
git push                               # upload commits
git push -u origin feature/login       # push and set upstream tracking
git push origin --delete old-branch    # delete remote branch
```

`fetch` is safe: it only updates your view of the remote. `pull` changes your current branch.

### Typical contribution flow (fork model)

```bash
git clone <your-fork-url>
git remote add upstream <original-repo-url>
git switch -c fix/bug-123
# ...edit, commit...
git fetch upstream
git rebase upstream/main          # or merge
git push -u origin fix/bug-123
# open a Pull Request on the hosting site
```

---

## 7. Rebase vs Merge

| | Merge | Rebase |
|---|---|---|
| History | Preserves true history, adds merge commits | Rewrites commits to form a straight line |
| Safety | Non-destructive | Rewrites history (new commit hashes) |
| Use for | Integrating finished branches, shared branches | Cleaning up your own local/feature branch before merging |

```bash
git switch feature
git rebase main                   # replay feature commits on top of main
git rebase --continue             # after resolving a conflict
git rebase --abort
```

### Interactive rebase (tidy your history)

```bash
git rebase -i HEAD~4              # edit the last 4 commits
```
Actions: `pick`, `reword`, `edit`, `squash`, `fixup`, `drop`, and reorder lines to reorder commits.

> **Golden rule:** never rebase commits that others have already pulled. If you must update a rewritten branch you own, use `git push --force-with-lease` (safer than `--force`).

---

## 8. Undoing Things

| Situation | Command |
|---|---|
| Discard unstaged changes to a file | `git restore file.txt` |
| Unstage a file (keep changes) | `git restore --staged file.txt` |
| Fix the last commit message or add forgotten files | `git commit --amend` |
| Undo a commit but keep changes staged | `git reset --soft HEAD~1` |
| Undo a commit, keep changes unstaged | `git reset HEAD~1` (mixed, default) |
| Undo a commit and **discard** changes | `git reset --hard HEAD~1` |
| Safely undo a pushed commit | `git revert <commit>` (creates an opposing commit) |
| Grab a file from another commit | `git restore --source=<commit> file.txt` |
| Remove untracked files | `git clean -n` (preview), then `git clean -f` |

### Reset modes

| Mode | Moves HEAD | Resets staging | Resets working dir |
|---|---|---|---|
| `--soft` | Yes | No | No |
| `--mixed` (default) | Yes | Yes | No |
| `--hard` | Yes | Yes | **Yes (data loss)** |

### The safety net: reflog

Git records where HEAD has been. If you lose commits after a reset or bad rebase:

```bash
git reflog                        # find the lost commit hash
git reset --hard <hash>           # or: git switch -c rescue <hash>
```
Entries typically stay for about 90 days.

---

## 9. Stashing

Temporarily shelve uncommitted work.

```bash
git stash                         # stash tracked changes
git stash -u                      # include untracked files
git stash list
git stash pop                     # apply latest and remove it
git stash apply stash@{1}         # apply without removing
git stash drop stash@{0}
git stash push -m "wip: search"   # with a name
```

---

## 10. Tags

Mark releases or important points.

```bash
git tag v1.0.0                          # lightweight
git tag -a v1.0.0 -m "First release"    # annotated (recommended)
git tag                                 # list
git push origin v1.0.0                  # push one tag
git push origin --tags                  # push all tags
git tag -d v1.0.0                       # delete locally
```

---

## 11. Ignoring Files

Create a `.gitignore` at the repo root:

```gitignore
# dependencies
node_modules/
venv/

# build output
dist/
*.o

# secrets and local config
.env
*.pem

# OS / editor
.DS_Store
.vscode/
*.log

# but keep this one
!important.log
```

Already-tracked files aren't affected by `.gitignore`. To untrack: `git rm --cached file`.

Global ignore for all repos: `git config --global core.excludesfile ~/.gitignore_global`.

> **Never commit secrets** (API keys, passwords). If you do, rotating the secret matters more than deleting the commit; history rewriting tools like `git filter-repo` can help, but assume the secret is compromised.

---

## 12. Branching Strategies

| Strategy | Summary | Fits |
|---|---|---|
| **GitHub Flow** | `main` is always deployable; short-lived feature branches + pull requests | Most web teams, continuous delivery |
| **Trunk-based** | Very short branches (or none), frequent merges to `main`, feature flags | High-velocity teams with strong CI |
| **Git Flow** | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | Scheduled releases, versioned software |

Common branch naming: `feature/…`, `fix/…`, `hotfix/…`, `chore/…`.

---

## 13. Useful Extras

```bash
git cherry-pick <commit>          # copy a commit onto current branch
git bisect start                  # binary search for the commit that introduced a bug
git bisect bad                    # current commit is broken
git bisect good <commit>          # known good commit
git bisect reset                  # finish

git worktree add ../hotfix main   # second working dir on another branch
git submodule add <url> path      # nest another repo
git grep "TODO"                   # search tracked files
git shortlog -sn                  # commits per author
git log --author="Name" --since="2 weeks ago"
git log -S"functionName"          # commits that added/removed that string
```

### Handy aliases

```bash
git config --global alias.st "status -sb"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.co "checkout"
git config --global alias.unstage "restore --staged"
```

---

## 14. Troubleshooting

| Problem | Fix |
|---|---|
| "detached HEAD" | You checked out a commit, not a branch. Run `git switch -c new-branch` to keep work, or `git switch main` to leave |
| Push rejected (non-fast-forward) | Remote has new commits: `git pull --rebase`, then push |
| Committed to the wrong branch | `git switch correct-branch`, `git cherry-pick <hash>`, then on the wrong branch `git reset --hard HEAD~1` (if not pushed) |
| Accidentally committed a large/secret file | Remove it, amend or rewrite history, rotate secrets |
| Merge went badly | `git merge --abort`, or `git reset --hard ORIG_HEAD` after the fact |
| Lost commits | `git reflog` |
| Line-ending noise (Windows/Linux) | `git config --global core.autocrlf true` (Windows) or `input` (Mac/Linux); or use `.gitattributes` |
| Authentication failing over HTTPS | Use a personal access token or credential manager; or switch to SSH keys |

### SSH key setup (quick)

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub         # add this to GitHub/GitLab
ssh -T git@github.com             # test
```

---

## 15. Cheat Sheet

| Task | Command |
|---|---|
| New repo | `git init` |
| Clone | `git clone <url>` |
| Status | `git status` |
| Stage all | `git add .` |
| Commit | `git commit -m "msg"` |
| New branch + switch | `git switch -c name` |
| Merge | `git merge name` |
| Push (first time) | `git push -u origin name` |
| Pull | `git pull` |
| History | `git log --oneline --graph --all` |
| Discard file changes | `git restore file` |
| Unstage | `git restore --staged file` |
| Amend last commit | `git commit --amend` |
| Undo pushed commit | `git revert <hash>` |
| Stash | `git stash` / `git stash pop` |
| Recover lost work | `git reflog` |

---

## 16. Best Practices

1. **Commit early, commit often**, with focused, well-described commits.
2. **Pull/fetch before you push** to stay in sync.
3. **Use branches** for every feature or fix; keep `main` healthy.
4. **Review diffs** (`git diff --staged`) before committing.
5. **Keep history readable**: squash noisy WIP commits before merging.
6. **Don't rewrite shared history.**
7. **Never commit secrets or large binaries**; use `.gitignore`, environment variables, or Git LFS.
8. **Protect `main`** on the hosting platform: require pull requests, reviews, and passing CI.

---

*Further reading: the free [Pro Git book](https://git-scm.com/book) and `git help <command>` for any command's manual.*
