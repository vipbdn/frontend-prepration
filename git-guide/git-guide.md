# 🌿 The Complete Git Guide — Daily Use, Revision & Interview Prep

> A practical guide you can **follow while working**, **revise quickly**, and **prepare for interviews** with. Beginners can read it top to bottom; experienced developers can jump to the cheat sheet, recipes, or interview questions.

---

## 📖 How to Use This Guide

| If you want to… | Go to |
| --- | --- |
| Learn Git from scratch | Sections 1 → 9 in order |
| Look up a command while working | [§21 Cheat Sheet](#21-cheat-sheet-) |
| Fix a mistake right now | [§10 Undoing Things](#10-undoing-things-) and [§19 Real-World Recipes](#19-real-world-recipes-) |
| Prepare for an interview | [§17 Internals](#17-how-git-works-internally-), [§22 Interview Questions](#22-interview-questions--answers-) |
| Learn team workflows | [§16 Workflows & Best Practices](#16-team-workflows--best-practices-) |

**Legend:** ⭐ used daily · ⚠️ dangerous / rewrites history · 💡 tip

---

## 📑 Table of Contents

1. [What is Git?](#1-what-is-git)
2. [Core Concepts (the mental model)](#2-core-concepts-the-mental-model-)
3. [Installation & Configuration](#3-installation--configuration)
4. [Starting a Repository](#4-starting-a-repository)
5. [The Daily Workflow](#5-the-daily-workflow-)
6. [.gitignore](#6-gitignore)
7. [Branching](#7-branching-)
8. [Merging & Conflicts](#8-merging--conflicts-)
9. [Remotes: Push, Pull, Fetch](#9-remotes-push-pull-fetch-)
10. [Undoing Things](#10-undoing-things-)
11. [Rebasing](#11-rebasing-)
12. [Stashing](#12-stashing)
13. [Cherry-Pick](#13-cherry-pick)
14. [Inspecting History & Debugging](#14-inspecting-history--debugging)
15. [Tags & Releases](#15-tags--releases)
16. [Team Workflows & Best Practices](#16-team-workflows--best-practices-)
17. [How Git Works Internally](#17-how-git-works-internally-)
18. [Advanced Tools](#18-advanced-tools)
19. [Real-World Recipes](#19-real-world-recipes-)
20. [GitHub / GitLab Essentials](#20-github--gitlab-essentials)
21. [Cheat Sheet](#21-cheat-sheet-)
22. [Interview Questions & Answers](#22-interview-questions--answers-)
23. [Practice Exercises](#23-practice-exercises)

---

# 1. What is Git?

**Git** is a **distributed version control system (VCS)**. It records snapshots of your project over time so you can:

- See **who changed what, when, and why**
- Go back to any previous version
- Work on features in parallel using **branches**
- **Collaborate** with others and merge everyone's work safely

### Centralized vs Distributed

| Centralized (SVN, Perforce) | Distributed (Git, Mercurial) |
| --- | --- |
| One central server holds history | **Every clone has the full history** |
| Need network for most operations | Commit, branch, log, diff all work **offline** |
| Server down = can't commit | Any clone can restore the server |
| Branching is heavy | Branching is cheap and fast |

### Git vs GitHub

| Git | GitHub / GitLab / Bitbucket |
| --- | --- |
| The version control **tool** (runs on your machine) | **Hosting services** for Git repositories |
| Created by Linus Torvalds (2005) for the Linux kernel | Add pull requests, code review, issues, CI/CD, permissions |
| Works without the internet | Web platforms |

---

# 2. Core Concepts (the mental model) ⭐

Understand these and every command makes sense.

## 2.1 The Three Areas (+ Remote)

```text
┌──────────────────┐   git add    ┌──────────────────┐  git commit  ┌──────────────────┐  git push   ┌──────────────────┐
│ Working Directory│ ───────────▶ │  Staging Area    │ ───────────▶ │ Local Repository │ ──────────▶ │ Remote Repository│
│ (your files)     │              │  (index)         │              │ (.git folder)    │             │ (GitHub, etc.)   │
└──────────────────┘ ◀─────────── └──────────────────┘              └──────────────────┘ ◀────────── └──────────────────┘
                      git restore                                     git fetch / git pull
```

| Area | What it is |
| --- | --- |
| **Working directory** | The actual files you edit |
| **Staging area (index)** | A "draft" of your next commit — you choose exactly which changes go in |
| **Local repository** | The `.git` folder: all commits, branches, history |
| **Remote repository** | A copy on a server (GitHub) shared with the team |

💡 The staging area lets you commit **part** of your changes — e.g. fix two unrelated bugs, then make two clean commits.

## 2.2 File States

```text
Untracked ──git add──▶ Staged ──git commit──▶ Unmodified (committed)
                         ▲                          │
                         │ git add                  │ edit file
                         └──────── Modified ◀───────┘
```

- **Untracked** — new file Git doesn't know about yet
- **Modified** — tracked file changed since last commit
- **Staged** — changes marked to go into the next commit
- **Committed / unmodified** — safely stored in the repository

## 2.3 Commits

A **commit** is a **snapshot** of the whole project at a point in time (not just a diff), plus metadata:

```text
commit 3f2a9c1e…            ← SHA-1 hash (unique ID, 40 hex chars; short form 3f2a9c1)
Author: Asha <asha@x.com>
Date:   Mon Nov 17 10:00
Parent: a81b2d0…            ← previous commit(s) → history is a chain (a graph)

    Add login form validation
```

- Commits are **immutable** — "changing" a commit actually creates a **new** commit with a new hash.
- Each commit points to its **parent(s)**: normal commits have 1 parent, merge commits have 2+, the first commit has 0.

## 2.4 Branches and HEAD

```text
                    main
                     │
A ─── B ─── C ─── D ◀┘
             \
              E ─── F ◀── feature ◀── HEAD
```

- A **branch** is just a **movable pointer (label) to a commit**. Creating one is instant (a 41-byte file).
- When you commit, the current branch pointer moves forward to the new commit.
- **HEAD** is a pointer to **where you are now** — usually to a branch name (`HEAD → feature → F`).
- **Detached HEAD**: HEAD points directly to a commit instead of a branch (e.g. after `git checkout <sha>`). Commits made here belong to no branch — create one if you want to keep them.

## 2.5 Remote-Tracking Branches

`origin/main` is your local **read-only bookmark** of where `main` was on the remote **the last time you fetched**. `git fetch` updates it; your local `main` is separate.

```text
Local:   main          → D
Remote-tracking: origin/main → F   (after git fetch — remote has new commits)
```

---

# 3. Installation & Configuration

## 3.1 Install & verify

```bash
git --version
# macOS: xcode-select --install  or  brew install git
# Ubuntu: sudo apt install git
# Windows: https://git-scm.com (Git Bash)
```

## 3.2 First-time setup ⭐

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main        # new repos start on "main"
git config --global core.editor "code --wait"      # VS Code as editor (or nano, vim)
git config --global pull.rebase true                # pull = fetch + rebase (cleaner history)
git config --global fetch.prune true                # auto-remove deleted remote branches
git config --global rebase.autoStash true           # stash/unstash automatically during rebase
git config --global push.autoSetupRemote true       # first push sets upstream automatically
git config --global rerere.enabled true             # remember conflict resolutions
```

### Config levels

| Level | Flag | File | Applies to |
| --- | --- | --- | --- |
| System | `--system` | `/etc/gitconfig` | All users |
| Global | `--global` | `~/.gitconfig` | Your user |
| Local | `--local` (default) | `.git/config` | One repo (overrides global) |

```bash
git config --list --show-origin     # see all settings and where they come from
git config user.email               # read one value
git config --local user.email "work@company.com"   # different email for a work repo
```

## 3.3 Line endings (cross-platform teams)

```bash
git config --global core.autocrlf input   # macOS/Linux: convert CRLF→LF on commit
git config --global core.autocrlf true    # Windows: LF in repo, CRLF in working tree
```

Better: commit a `.gitattributes` file so everyone behaves the same:

```text
* text=auto eol=lf
*.png binary
```

## 3.4 SSH key setup (GitHub/GitLab)

```bash
ssh-keygen -t ed25519 -C "you@example.com"     # creates ~/.ssh/id_ed25519(.pub)
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub                       # paste into GitHub → Settings → SSH keys
ssh -T git@github.com                           # test
```

HTTPS alternative: use a **personal access token** or the `gh auth login` credential helper (passwords are no longer accepted).

## 3.5 Useful aliases 💡

```bash
git config --global alias.st "status -sb"
git config --global alias.co checkout
git config --global alias.sw switch
git config --global alias.br branch
git config --global alias.cm "commit -m"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage "restore --staged"
git config --global alias.undo "reset --soft HEAD~1"
```

Now `git lg` shows a pretty graph of all branches.

---

# 4. Starting a Repository

```bash
# Option 1: turn an existing folder into a repo
cd my-project
git init                      # creates .git/
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:user/my-project.git
git push -u origin main

# Option 2: copy an existing repo
git clone https://github.com/user/repo.git            # into ./repo
git clone git@github.com:user/repo.git my-folder      # into ./my-folder
git clone --branch develop <url>                       # check out a specific branch
git clone --depth 1 <url>                              # shallow clone: latest snapshot only (fast CI)
```

---

# 5. The Daily Workflow ⭐

```text
git pull  →  edit  →  git status  →  git diff  →  git add  →  git commit  →  git push
```

## 5.1 `git status` ⭐

```bash
git status          # full status
git status -s       # short: "M " staged, " M" modified, "??" untracked, "A " added
git status -sb      # short + branch & ahead/behind info
```

## 5.2 `git add` — stage changes ⭐

```bash
git add file.js              # one file
git add src/                 # a folder
git add .                    # everything in current directory (new, modified, deleted)
git add -A                   # everything in the whole repo
git add -u                   # only tracked files (modified/deleted), not new files
git add -p                   # 💡 interactive: choose hunks one by one (y/n/s/e)
git add *.css                # by pattern
```

💡 `git add -p` is the secret to clean commits: review every change before staging it.

## 5.3 `git commit` ⭐

```bash
git commit -m "Add login form validation"
git commit -m "Title" -m "Longer description paragraph"   # subject + body
git commit                       # opens editor for a detailed message
git commit -am "Fix typo"        # stage all TRACKED modified files + commit (not new files)
git commit --amend               # ⚠️ modify the last commit (message and/or content)
git commit --amend --no-edit     # add staged changes to last commit, keep message
git commit --allow-empty -m "Trigger CI"
```

### Writing good commit messages

```text
feat(auth): add password strength meter          ← subject: imperative, ≤ 50–72 chars, no period

Users often chose weak passwords. Show a meter     ← body: WHAT and WHY (not how), wrapped at 72
based on zxcvbn score and block scores below 2.

Closes #142                                         ← footer: issue refs, BREAKING CHANGE
```

**Rules:** imperative mood ("Add", not "Added"), one logical change per commit, explain *why*.

**Conventional Commits** (widely used; enables automatic changelogs & versioning):

| Type | Use |
| --- | --- |
| `feat:` | New feature |
| `fix:` | Bug fix |
| `docs:` | Documentation |
| `style:` | Formatting, no code change |
| `refactor:` | Code change that neither fixes a bug nor adds a feature |
| `perf:` | Performance improvement |
| `test:` | Tests |
| `chore:` / `build:` / `ci:` | Tooling, deps, pipelines |
| `feat!:` or `BREAKING CHANGE:` footer | Breaking change |

## 5.4 `git diff` — see changes ⭐

```bash
git diff                     # working directory vs staging (unstaged changes)
git diff --staged            # staging vs last commit (what will be committed) — alias --cached
git diff HEAD                # all changes (staged + unstaged) vs last commit
git diff main..feature       # difference between two branches
git diff main...feature      # changes on feature since it branched from main (what a PR shows)
git diff abc123 def456       # between two commits
git diff HEAD~1 -- file.js   # one file vs previous commit
git diff --stat              # summary: files and line counts
git diff --name-only         # just file names
git diff --word-diff         # word-level (great for prose/docs)
```

## 5.5 `git log` — view history ⭐

```bash
git log                                  # full log
git log --oneline                        # one line per commit
git log --oneline --graph --all          # ASCII graph of all branches
git log -5                               # last 5
git log -p file.js                       # history of a file with diffs
git log --stat                           # files changed per commit
git log --author="Asha"
git log --since="2 weeks ago" --until="yesterday"
git log --grep="login"                   # search commit messages
git log -S "functionName"                # 💡 "pickaxe": commits that added/removed this text
git log main..feature                    # commits on feature not on main
git log --follow -- old/path.js          # follow a file across renames
git log --pretty=format:"%h %an %ar %s"  # custom format
```

## 5.6 `git show` — inspect one commit

```bash
git show                 # last commit with diff
git show abc123          # a specific commit
git show abc123 --stat
git show HEAD~2:src/app.js   # file content as it was 2 commits ago
```

## 5.7 Remove, rename, restore files

```bash
git rm file.js              # delete file and stage the deletion
git rm --cached secret.env  # stop tracking but keep the file locally (then add to .gitignore)
git rm -r --cached node_modules
git mv old.js new.js        # rename (Git also detects renames automatically)

git restore file.js         # ⚠️ discard unstaged changes in working dir (back to staged/committed)
git restore --staged file.js   # unstage (keep changes in working dir)
git restore --source=HEAD~2 file.js   # restore file content from an older commit
```

💡 `git restore` and `git switch` (Git 2.23+) split the overloaded `git checkout` into clearer commands.

---

# 6. .gitignore

Tells Git which **untracked** files to ignore.

```gitignore
# Dependencies
node_modules/
vendor/

# Build output
dist/
build/
*.min.js

# Environment & secrets
.env
.env.*
!.env.example        # ! = exception: DO track this one

# Logs & OS / editor files
*.log
.DS_Store
Thumbs.db
.idea/
.vscode/*
!.vscode/settings.json

# Patterns
/config.local.js     # leading / = only in repo root
docs/**/*.pdf        # ** = any depth
temp?.txt            # ? = single character
```

### Rules

- Patterns apply to paths relative to the `.gitignore` location; you can have one per folder.
- **Already-tracked files are NOT ignored** by adding them to `.gitignore`. Untrack them first:

```bash
git rm --cached .env          # or -r for folders
echo ".env" >> .gitignore
git commit -m "Stop tracking .env"
```

- Global ignore for your OS/editor junk (not project-specific):

```bash
git config --global core.excludesFile ~/.gitignore_global
```

- Debug: `git check-ignore -v path/to/file` shows which rule ignores it.
- Empty folders aren't tracked — add a placeholder like `.gitkeep`.
- Templates: github.com/github/gitignore

---

# 7. Branching ⭐

## 7.1 Why branches?

Isolate work (features, bug fixes, experiments) without affecting `main`. Branches are cheap — create them freely.

## 7.2 Commands

```bash
git branch                     # list local branches (* = current)
git branch -a                  # local + remote-tracking branches
git branch -vv                 # with last commit and upstream (ahead/behind)
git branch --merged            # branches already merged into current (safe to delete)
git branch --no-merged

git switch feature/login       # ⭐ move to an existing branch
git switch -c feature/login    # ⭐ create and switch
git switch -c fix/bug origin/fix/bug   # create local branch from a remote branch
git switch -                   # switch to previous branch (like cd -)

# Older equivalents (still everywhere in tutorials)
git checkout feature/login
git checkout -b feature/login

git branch feature/x           # create without switching
git branch feature/x abc123    # create at a specific commit

git branch -m new-name         # rename current branch
git branch -m old new          # rename another branch

git branch -d feature/login    # delete (only if merged — safe)
git branch -D feature/login    # ⚠️ force delete (even if unmerged)
git push origin --delete feature/login   # delete remote branch
```

## 7.3 Branch naming conventions

```text
feature/user-profile      feat/JIRA-123-user-profile
fix/login-redirect        bugfix/payment-timeout
hotfix/critical-crash
chore/upgrade-react
release/2.4.0
```

Lowercase, hyphens, include ticket ID if your team uses one.

## 7.4 Switching with uncommitted changes

- If changes don't conflict with the target branch, Git carries them over.
- If they would be overwritten, Git refuses → **commit**, **stash** (§12), or discard them first.

---

# 8. Merging & Conflicts ⭐

## 8.1 Merge

```bash
git switch main
git merge feature/login      # bring feature's commits into main
```

### Fast-forward merge

If `main` hasn't moved since the branch was created, Git just moves the pointer — no merge commit.

```text
Before:  A ─ B (main)            After:  A ─ B ─ C ─ D (main, feature)
              \
               C ─ D (feature)
```

### Three-way merge (merge commit)

If both branches have new commits, Git creates a **merge commit** with two parents, using the common ancestor as a base.

```text
Before:  A ─ B ─ E (main)        After:  A ─ B ─ E ─── M (main)
              \                               \       /
               C ─ D (feature)                 C ─ D (feature)
```

### Options

```bash
git merge --no-ff feature     # always create a merge commit (preserves "this was a feature" grouping)
git merge --ff-only feature   # only merge if fast-forward possible, else fail
git merge --squash feature    # combine all feature changes into staged changes → you make ONE commit
git merge --abort             # cancel a merge in progress (during conflicts)
```

## 8.2 Merge Conflicts ⭐

A conflict happens when **both branches changed the same lines** (or one deleted a file the other modified). Git pauses and asks you to decide.

```text
<<<<<<< HEAD
const timeout = 5000;          ← your current branch's version
=======
const timeout = 10000;         ← incoming branch's version
>>>>>>> feature/login
```

### Step-by-step resolution

```bash
git merge feature/login          # CONFLICT (content): Merge conflict in config.js
git status                       # lists "both modified" files

# 1. Open each file, decide the final content, delete the <<<<<<< ======= >>>>>>> markers
#    (VS Code: "Accept Current / Incoming / Both")
# 2. Mark resolved:
git add config.js
# 3. Complete the merge:
git commit                       # (or `git merge --continue`)

# Changed your mind?
git merge --abort
```

Take one side for a whole file:

```bash
git checkout --ours config.js    # keep your branch's version
git checkout --theirs config.js  # keep the incoming version
git add config.js
```

⚠️ During a **rebase**, "ours" and "theirs" are **swapped** (ours = the branch you're rebasing onto).

### Reducing conflicts

- Pull/rebase from `main` often; keep branches short-lived
- Small, focused PRs
- Agree on formatting (Prettier) so whitespace doesn't conflict
- `git config rerere.enabled true` — reuse recorded resolutions
- `git diff --name-only --diff-filter=U` lists unresolved files

---

# 9. Remotes: Push, Pull, Fetch ⭐

## 9.1 Managing remotes

```bash
git remote -v                                   # list remotes with URLs
git remote add origin git@github.com:me/app.git
git remote add upstream git@github.com:org/app.git   # original repo (forks)
git remote set-url origin git@github.com:me/new.git  # change URL (e.g. HTTPS → SSH)
git remote rename origin old-origin
git remote remove upstream
git remote show origin                          # details: branches, tracking
```

`origin` is just the conventional name for the remote you cloned from.

## 9.2 `git fetch` — download, don't change your work

```bash
git fetch                 # update all remote-tracking branches (origin/*)
git fetch origin main
git fetch --all --prune   # all remotes; remove refs to deleted remote branches
```

Safe anytime: it never modifies your working directory or local branches. Then inspect:

```bash
git log main..origin/main     # what's new on the remote
git diff main origin/main
```

## 9.3 `git pull` — fetch + integrate ⭐

```bash
git pull                  # = git fetch + git merge origin/<branch>  (or rebase if configured)
git pull --rebase         # = fetch + rebase your local commits on top → linear history
git pull origin main      # pull a specific branch into the current one
git pull --ff-only        # refuse if it can't fast-forward (safest)
```

### `fetch` vs `pull` ⭐ (interview favourite)

| `git fetch` | `git pull` |
| --- | --- |
| Downloads new commits into `origin/*` | Fetch **+** merge/rebase into your current branch |
| Never touches your files | Changes your branch & files; may cause conflicts |
| Lets you review first | Convenient |

## 9.4 `git push` — upload your commits ⭐

```bash
git push                              # push current branch to its upstream
git push -u origin feature/login      # first push: set upstream (tracking) — then just `git push`
git push origin main
git push origin --delete old-branch   # delete remote branch
git push --tags                       # push all tags
git push origin v1.2.0                # push one tag

git push --force-with-lease           # ⚠️ overwrite remote branch ONLY if no one else pushed meanwhile
git push --force                      # ⚠️⚠️ overwrite remote blindly — avoid on shared branches
```

### "Rejected — non-fast-forward"

```text
! [rejected] main -> main (fetch first)
```

Someone pushed commits you don't have. Fix:

```bash
git pull --rebase      # put your commits on top of theirs
# resolve conflicts if any
git push
```

**Never** "fix" this with `--force` on a shared branch — you'd delete their work.

### `--force-with-lease` vs `--force`

After rebasing **your own** feature branch, you must force-push. `--force-with-lease` refuses if the remote branch changed since you last fetched (someone else pushed), protecting their work. Always prefer it.

## 9.5 Upstream (tracking) branches

```bash
git branch -u origin/feature/login     # set upstream for current branch
git branch -vv                          # show upstream and ahead/behind counts
git status -sb                          # "## feature...origin/feature [ahead 2, behind 1]"
```

## 9.6 Working with forks

```bash
git clone git@github.com:ME/project.git
git remote add upstream git@github.com:ORIGINAL/project.git

# Keep your fork updated
git fetch upstream
git switch main
git rebase upstream/main      # or: git merge upstream/main
git push origin main

# Contribute: branch → push to YOUR fork (origin) → open PR to upstream
git switch -c fix/typo
git push -u origin fix/typo
```

---

# 10. Undoing Things ⭐

The most valuable section in practice. First ask: **Has it been pushed/shared?**

- **Not pushed** → rewriting history (amend, reset, rebase) is fine.
- **Pushed to a shared branch** → use `git revert` (adds a new commit; doesn't rewrite history).

## 10.1 Decision table ⭐

| Situation | Command |
| --- | --- |
| Discard changes in a file (not staged) | `git restore file` |
| Discard all unstaged changes | `git restore .` |
| Unstage a file (keep changes) | `git restore --staged file` |
| Remove untracked files/folders | `git clean -n` (preview) → `git clean -fd` ⚠️ |
| Fix last commit message (not pushed) | `git commit --amend -m "New message"` |
| Add forgotten file to last commit (not pushed) | `git add file && git commit --amend --no-edit` |
| Undo last commit, keep changes **staged** | `git reset --soft HEAD~1` |
| Undo last commit, keep changes **unstaged** | `git reset HEAD~1` (= `--mixed`) |
| Undo last commit and **delete** the changes | `git reset --hard HEAD~1` ⚠️ |
| Undo a commit that's already **pushed** | `git revert <sha>` |
| Undo a pushed **merge** commit | `git revert -m 1 <merge-sha>` |
| Reset branch to match remote exactly | `git fetch && git reset --hard origin/main` ⚠️ |
| Recover a "lost" commit / undo a bad reset | `git reflog` → `git reset --hard <sha>` |
| Restore a deleted branch | `git reflog` → `git switch -c branch <sha>` |
| Restore a file from an old commit | `git restore --source=<sha> file` |
| Abort a merge / rebase / cherry-pick | `git merge --abort` / `git rebase --abort` / `git cherry-pick --abort` |

## 10.2 `git reset` explained ⭐⭐⭐

`git reset <commit>` moves the **current branch pointer** to `<commit>`. The mode decides what happens to the staging area and working directory.

```text
History: A ─ B ─ C (HEAD, main)        git reset --<mode> B
```

| Mode | Branch pointer | Staging area | Working directory | Use |
| --- | --- | --- | --- | --- |
| `--soft` | → B | keeps C's changes **staged** | unchanged | Recommit differently / squash |
| `--mixed` (default) | → B | reset to B (changes **unstaged**) | unchanged | Re-stage selectively |
| `--hard` ⚠️ | → B | reset to B | **reset to B — changes lost** | Throw away work |

```bash
git reset --soft HEAD~3     # squash last 3 commits: changes staged, then commit once
git reset HEAD~1            # undo last commit, keep edits
git reset --hard HEAD       # discard ALL uncommitted changes to tracked files
git reset file.js           # (path form) unstage a file — same as git restore --staged
```

`HEAD~1` = parent of HEAD, `HEAD~3` = 3 commits back, `HEAD^2` = second parent of a merge.

## 10.3 `git revert` — safe undo for shared history ⭐

Creates a **new commit** that applies the inverse of the given commit.

```bash
git revert abc123              # opens editor for message; creates "Revert ..." commit
git revert abc123 --no-edit
git revert HEAD~3..HEAD        # revert a range (newest first)
git revert -n abc123           # apply inverse but don't commit yet (--no-commit)
git revert -m 1 <merge-sha>    # revert a merge, keeping parent #1 (usually main) as mainline
```

### reset vs revert ⭐

| `reset` | `revert` |
| --- | --- |
| Moves branch pointer back — **rewrites history** | Adds a new commit — **preserves history** |
| Commits "disappear" from the branch | Original commit stays; its effect is undone |
| Only for local / unpushed commits | Safe for pushed / shared branches |

## 10.4 `git reflog` — your safety net ⭐⭐⭐

Git records every position HEAD has been at (commits, resets, checkouts, rebases) for ~90 days. Even "deleted" commits can be found.

```bash
git reflog
# 9c1d2e3 HEAD@{0}: reset: moving to HEAD~2
# 4b5a6f7 HEAD@{1}: commit: Add payment form      ← the commit you "lost"
# 1a2b3c4 HEAD@{2}: commit: Add cart

git reset --hard 4b5a6f7        # go back to it
# or keep current state and just recover:
git switch -c recovered 4b5a6f7
```

💡 Almost nothing committed is ever truly lost in Git — unless it was never committed (`reset --hard` on uncommitted changes and `git clean` are the real dangers).

## 10.5 `git clean` — remove untracked files

```bash
git clean -n        # dry run: show what would be deleted
git clean -f        # delete untracked files
git clean -fd       # + untracked directories
git clean -fdx      # ⚠️ + ignored files too (node_modules, .env!) — full reset of build artifacts
git clean -i        # interactive
```

---

# 11. Rebasing ⭐

## 11.1 What rebase does

`git rebase main` (while on `feature`) **replays** your feature commits **on top of** the latest `main`, creating **new** commits with new hashes.

```text
Before:                         After git rebase main (on feature):
A ─ B ─ E ─ F (main)            A ─ B ─ E ─ F (main)
     \                                         \
      C ─ D (feature)                           C' ─ D' (feature)
```

Result: a **linear** history, as if you started the feature from the latest `main`.

```bash
git switch feature
git rebase main
# conflicts? resolve each commit:
git add <file>
git rebase --continue
git rebase --skip      # drop the current commit
git rebase --abort     # go back to how it was before

git push --force-with-lease   # if the branch was pushed before (history changed)
```

Common daily pattern:

```bash
git fetch origin
git rebase origin/main        # update my feature branch with latest main
```

## 11.2 Merge vs Rebase ⭐⭐⭐

| | Merge | Rebase |
| --- | --- | --- |
| History | Preserves exact history; adds merge commits | Linear, clean history |
| Rewrites commits? | No | **Yes** (new hashes) |
| Safe on shared branches? | ✅ Yes | ⚠️ Only on your own unshared branches |
| Conflicts | Resolved once in the merge commit | Possibly resolved per replayed commit |
| Traceability | Shows when branches joined | Hides branch structure |
| Typical use | Integrating a feature into `main` (PR merge) | Updating your feature branch; cleaning up commits before a PR |

### 🛑 The Golden Rule of Rebasing

> **Never rebase commits that others have based work on** (i.e., public/shared branches like `main`, `develop`). Rebasing rewrites history; collaborators' copies will diverge and get duplicated commits.

## 11.3 Interactive Rebase — clean up your commits ⭐

```bash
git rebase -i HEAD~4          # edit the last 4 commits
git rebase -i main            # all commits on this branch since main
```

The editor opens (oldest at the top):

```text
pick a1b2c3 Add login form
pick d4e5f6 WIP
pick 789abc Fix typo in login form
pick def012 Add validation

# Commands:
# p, pick   = use commit
# r, reword = use commit, edit message
# e, edit   = stop to amend the commit (change content / split it)
# s, squash = meld into previous commit, combine messages
# f, fixup  = meld into previous commit, discard this message
# d, drop   = remove commit
# (reorder lines to reorder commits)
```

Example — squash WIP and typo fix into the first commit:

```text
pick a1b2c3 Add login form
fixup 789abc Fix typo in login form
fixup d4e5f6 WIP
pick def012 Add validation
```

### Autosquash workflow 💡

```bash
git commit --fixup=a1b2c3        # creates "fixup! Add login form"
git rebase -i --autosquash main  # automatically placed & marked as fixup
```

### Split a commit

```bash
git rebase -i HEAD~3      # mark the commit as "edit"
git reset HEAD~1          # un-commit it, keep changes
git add -p && git commit -m "Part 1"
git add -p && git commit -m "Part 2"
git rebase --continue
```

## 11.4 `rebase --onto` (advanced)

Move a branch to a different base:

```bash
# feature-b was branched from feature-a, but should be based on main
git rebase --onto main feature-a feature-b
```

---

# 12. Stashing

Temporarily shelve uncommitted changes to get a clean working directory (e.g. urgent bug fix on another branch).

```bash
git stash                       # stash tracked modified + staged changes
git stash push -m "WIP: header" # with a message ⭐
git stash -u                    # include untracked files
git stash push -- src/app.js    # stash specific files

git stash list                  # stash@{0}: On main: WIP: header
git stash show -p stash@{1}     # view a stash's diff

git stash pop                   # apply latest stash and remove it
git stash apply stash@{1}       # apply but keep it in the list
git stash drop stash@{1}        # delete one
git stash clear                 # ⚠️ delete all

git stash branch new-branch stash@{0}   # create branch from stash (if pop would conflict)
```

Typical use:

```bash
git stash push -m "half-done feature"
git switch main && git switch -c hotfix/crash
# …fix, commit, push…
git switch feature && git stash pop
```

💡 Stashes are easy to forget. For longer interruptions, a WIP commit on your branch (amended later) or a **worktree** (§18) is often better.

---

# 13. Cherry-Pick

Apply the changes of **specific commits** onto your current branch (creates new commits).

```bash
git switch release/2.4
git cherry-pick abc123               # copy one commit
git cherry-pick abc123 def456        # several
git cherry-pick abc123^..def456      # a range (inclusive)
git cherry-pick -x abc123            # append "(cherry picked from commit …)" to message
git cherry-pick -n abc123            # apply without committing
git cherry-pick --continue | --abort | --skip
```

**Use cases:** backport a bug fix to a release branch, rescue a commit made on the wrong branch, take one commit from an abandoned branch.

⚠️ Overuse duplicates commits across branches (different hashes, same change) and complicates history — prefer merging when possible.

---

# 14. Inspecting History & Debugging

## 14.1 `git blame` — who changed this line?

```bash
git blame src/auth.js
git blame -L 40,60 src/auth.js      # only lines 40–60
git blame -w -C src/auth.js         # ignore whitespace; detect moved/copied lines
```

💡 Ignore formatting commits: list them in `.git-blame-ignore-revs` and `git config blame.ignoreRevsFile .git-blame-ignore-revs`.

## 14.2 Searching

```bash
git grep "TODO"                    # search tracked files (fast)
git grep -n "useAuth" -- "*.ts"
git log -S "API_KEY"               # commits that added/removed the string
git log -G "regex"                 # commits whose diff matches regex
git log --grep="payment" -i        # search commit messages
```

## 14.3 `git bisect` — find the commit that introduced a bug ⭐

Binary search through history: O(log n) steps.

```bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v2.3.0         # this older version worked
# Git checks out a commit in the middle → test it
git bisect good                # or: git bisect bad
# …repeat until Git prints "abc123 is the first bad commit"
git bisect reset               # return to where you started
```

Automate with a test script (exit 0 = good, non-zero = bad):

```bash
git bisect start HEAD v2.3.0
git bisect run npm test
```

## 14.4 Summaries

```bash
git shortlog -sn              # commit counts per author
git log --oneline | wc -l     # total commits
git diff --stat main..feature
git count-objects -vH         # repo size
```

---

# 15. Tags & Releases

Tags mark specific commits — usually releases (`v1.0.0`).

```bash
git tag                                  # list
git tag -l "v2.*"
git tag v1.0.0                           # lightweight tag (just a pointer)
git tag -a v1.0.0 -m "Release 1.0.0"     # ⭐ annotated tag (author, date, message — recommended)
git tag -a v0.9.0 abc123 -m "Late tag"   # tag an older commit
git show v1.0.0

git push origin v1.0.0                   # tags are NOT pushed by default
git push --tags                          # push all tags
git push --follow-tags                   # push commits + annotated tags reachable from them

git tag -d v1.0.0                        # delete local
git push origin --delete v1.0.0          # delete remote

git switch --detach v1.0.0               # inspect code at a tag (detached HEAD)
git switch -c hotfix/1.0.1 v1.0.0        # branch from a tag
```

| Lightweight | Annotated |
| --- | --- |
| Just a name → commit | Full object: tagger, date, message, can be signed |
| Private/temporary markers | Releases |

### Semantic Versioning (SemVer)

`MAJOR.MINOR.PATCH` → `2.4.1`
- **MAJOR**: breaking changes
- **MINOR**: new features, backwards compatible
- **PATCH**: bug fixes

Tools like `semantic-release`, `release-please` and Changesets create tags and changelogs from Conventional Commits.

---

# 16. Team Workflows & Best Practices ⭐

## 16.1 Branching strategies

### GitHub Flow (simple, very common)

```text
main (always deployable)
  └── feature/x → Pull Request → review + CI → merge → deploy
```

1. Branch from `main`
2. Commit, push, open a PR early
3. Review + automated checks
4. Merge to `main` → deploy

Best for: web apps with continuous deployment.

### Git Flow (structured releases)

```text
main ─────────●──────────────●─────── (production releases, tagged)
               \            / \
release/1.2     ●──●──●────●   \
               /                \
develop ──●──●──●──●──●──●──●────●── (integration)
           \     /    \     /
feature/a   ●──●       ●──●  feature/b
hotfix/1.1.1 branches from main → merged into main AND develop
```

Branches: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`.
Best for: versioned software with scheduled releases (mobile apps, libraries). Heavier; many teams find it overkill for web apps.

### Trunk-Based Development

- Everyone integrates into `main` (trunk) at least daily via very short-lived branches or direct commits.
- Incomplete features hidden behind **feature flags**.
- Requires strong CI and testing.
- Used by high-velocity teams (Google, Meta); enables continuous delivery.

| | GitHub Flow | Git Flow | Trunk-Based |
| --- | --- | --- | --- |
| Complexity | Low | High | Low (needs discipline + CI) |
| Long-lived branches | `main` | `main`, `develop` | `main` |
| Release style | Continuous | Scheduled versions | Continuous |
| Merge conflicts | Moderate | More (long branches) | Fewest |

## 16.2 Pull Request (PR) workflow ⭐

```bash
git switch main && git pull
git switch -c feat/JIRA-42-profile-page
# work… small logical commits
git fetch origin && git rebase origin/main    # stay up to date
git push -u origin feat/JIRA-42-profile-page
# open PR → description, screenshots, linked ticket, reviewers
# address review: new commits (or fixup + autosquash), push
# merge when approved + CI green → delete branch
```

### PR merge strategies (GitHub "Merge" button options)

| Option | Result | Pros | Cons |
| --- | --- | --- | --- |
| **Create a merge commit** | All commits + a merge commit | Full history, clear PR boundary | Noisy history |
| **Squash and merge** | All PR commits → **one** commit on main | Clean main history, one commit per PR | Loses individual commits; branch looks unmerged locally |
| **Rebase and merge** | PR commits replayed linearly, no merge commit | Linear history keeps commits | Rewritten hashes; each commit must be clean |

### Good PR habits

- Small PRs (< ~400 lines changed) → faster, better reviews
- One purpose per PR; descriptive title (often Conventional Commit style)
- Self-review the diff before requesting review
- Include testing steps, screenshots for UI
- Keep CI green; respond to all comments
- Delete merged branches

## 16.3 Commit best practices

- Commit **often**, push regularly (backup + visibility)
- **Atomic commits**: one logical change, builds & tests pass
- Meaningful messages (why, not just what)
- Don't commit: secrets, `.env`, `node_modules`, build output, large binaries, IDE files
- Review with `git diff --staged` before committing
- Clean up local history before sharing (interactive rebase); **never rewrite shared history**

## 16.4 Protecting `main`

- **Branch protection rules**: require PR reviews, passing status checks, up-to-date branches, linear history, signed commits; disallow force-push and deletion.
- **CODEOWNERS** file: auto-request reviews from owners of changed paths.
- **Git hooks** (§18.3) + CI for linting/tests.

---

# 17. How Git Works Internally ⭐

Interviewers love this; it also makes Git feel far less magical.

## 17.1 Git is a content-addressable key-value store

Everything is stored as an **object** identified by the **SHA-1 hash of its content** (newer repos can use SHA-256). Same content → same hash → stored once.

| Object | Stores |
| --- | --- |
| **blob** | File **content** (no name, no permissions) |
| **tree** | A directory: list of names → blob/tree hashes + modes |
| **commit** | Pointer to a root **tree** (snapshot), parent commit(s), author, committer, message |
| **tag** (annotated) | Pointer to an object + tagger, message, optional signature |

```text
commit 7f3a…
 ├─ tree 9c1b…            (project root)
 │   ├─ blob 1aa2…  README.md
 │   └─ tree 4d5e…  src/
 │       └─ blob 8f9a…  app.js
 ├─ parent 2b3c…
 ├─ author / committer
 └─ message
```

💡 Unchanged files between commits reuse the **same blob** — that's why snapshots are cheap. Packfiles compress objects further using deltas.

```bash
git cat-file -t 7f3a     # type: commit
git cat-file -p 7f3a     # pretty-print content
git ls-tree HEAD         # tree of the current commit
git hash-object file.js  # compute a blob hash
```

## 17.2 The `.git` directory

```text
.git/
├── HEAD              → "ref: refs/heads/main"  (current branch)
├── config            → repo-level settings, remotes
├── index             → the staging area (binary file)
├── objects/          → all blobs, trees, commits, tags (+ packs/)
├── refs/
│   ├── heads/        → local branches: files containing a commit hash
│   ├── remotes/      → remote-tracking branches (origin/main)
│   └── tags/
├── logs/             → reflogs
└── hooks/            → sample hook scripts
```

```bash
cat .git/HEAD                 # ref: refs/heads/main
cat .git/refs/heads/main      # 7f3a… (a branch is literally a file with a hash)
```

## 17.3 What commands really do

| Command | Under the hood |
| --- | --- |
| `git add` | Writes blobs for file content, updates the **index** |
| `git commit` | Writes tree objects from the index, a commit object pointing to it, moves the branch ref |
| `git branch x` | Creates `refs/heads/x` containing the current commit hash |
| `git switch x` | Updates `HEAD` to `ref: refs/heads/x`, updates index & working dir |
| `git merge` | Finds the merge base, three-way merges trees, writes a commit with 2 parents (or moves the ref for fast-forward) |
| `git rebase` | Computes patches of your commits, re-applies them on the new base (new commits), moves the ref |
| `git reset` | Moves the branch ref (and optionally index/working dir) |
| `git fetch` | Downloads missing objects, updates `refs/remotes/*` |
| `git push` | Uploads missing objects, asks remote to move its ref (fails if not fast-forward unless forced) |
| `git gc` | Packs loose objects, removes unreachable ones (after reflog expiry) |

---

# 18. Advanced Tools

## 18.1 `git worktree` — multiple branches checked out at once 💡

```bash
git worktree add ../app-hotfix hotfix/crash     # new folder with that branch checked out
git worktree add -b feat/new ../app-new main    # create branch + worktree
git worktree list
git worktree remove ../app-hotfix
```

Work on a hotfix without stashing or disturbing your current branch (and without re-cloning).

## 18.2 Submodules (and alternatives)

Embed another repository at a fixed commit.

```bash
git submodule add https://github.com/org/lib.git libs/lib
git clone --recurse-submodules <url>
git submodule update --init --recursive    # after a normal clone
git submodule update --remote              # pull latest from submodule's branch
```

Submodules are notoriously awkward (detached heads, forgotten updates). Alternatives: package managers (npm), monorepos, `git subtree`.

## 18.3 Git hooks

Scripts that run on events (`.git/hooks/`, not committed by default).

| Hook | Runs | Typical use |
| --- | --- | --- |
| `pre-commit` | Before commit is created | Lint/format staged files |
| `commit-msg` | After message is written | Enforce Conventional Commits |
| `pre-push` | Before push | Run tests |
| `post-merge` / `post-checkout` | After merge/checkout | Reinstall deps if lockfile changed |

Share hooks with the team using **Husky** + **lint-staged** (JS projects), `pre-commit` framework (Python), or `git config core.hooksPath .githooks`.

```json
// package.json
"lint-staged": { "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"] }
```

Bypass in emergencies: `git commit --no-verify` (CI should still catch problems).

## 18.4 Large files — Git LFS

Git stores every version of every file; large binaries (videos, PSDs, datasets) bloat the repo.

```bash
git lfs install
git lfs track "*.psd"
git add .gitattributes
```

LFS stores pointers in Git and the actual files on an LFS server.

## 18.5 Signing commits

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git log --show-signature
```

Shows "Verified" on GitHub; proves commits came from you.

## 18.6 Rewriting large history (secrets, big files) ⚠️

```bash
# Use git-filter-repo (recommended over filter-branch)
git filter-repo --path secrets.env --invert-paths      # remove a file from all history
git filter-repo --replace-text replacements.txt         # replace leaked strings
```

Then force-push and have everyone re-clone. **Always rotate leaked secrets** — rewriting history doesn't un-leak them (forks, clones, caches may have copies).

## 18.7 Sparse checkout & partial clone (huge monorepos)

```bash
git clone --filter=blob:none <url>        # partial clone: fetch file contents lazily
git sparse-checkout set apps/web packages/ui   # only check out these folders
```

## 18.8 Other handy commands

```bash
git rerere                       # reuse recorded conflict resolutions (enable via config)
git notes add -m "Deployed to prod" abc123
git archive --format=zip HEAD > release.zip   # export without .git
git describe --tags              # v1.2.0-5-gabc123 (5 commits after v1.2.0)
git range-diff main old-feature new-feature   # compare two versions of a rebased branch
git maintenance start            # background optimization for big repos
```

---

# 19. Real-World Recipes ⭐

Copy-paste solutions to the problems you'll actually hit.

### 1. I committed to `main` instead of a feature branch (not pushed)

```bash
git switch -c feature/my-work     # new branch keeps the commits
git switch main
git reset --hard origin/main      # move main back to the remote state
```

### 2. I committed to the wrong branch (move last commit to another existing branch)

```bash
git switch correct-branch
git cherry-pick wrong-branch      # copies the tip commit of wrong-branch
git switch wrong-branch
git reset --hard HEAD~1           # remove it from the wrong branch (if not pushed)
```

### 3. Wrong commit message on the last commit

```bash
git commit --amend -m "Correct message"     # not pushed
# already pushed to YOUR branch: git push --force-with-lease
```

### 4. Forgot to include a file in the last commit

```bash
git add forgotten.js
git commit --amend --no-edit
```

### 5. Undo the last commit but keep my changes

```bash
git reset --soft HEAD~1     # staged
git reset HEAD~1            # unstaged
```

### 6. Throw away ALL local changes and match the remote

```bash
git fetch origin
git reset --hard origin/main
git clean -fd               # also remove untracked files (preview with -n first!)
```

### 7. Undo a commit that's already on `main` (shared)

```bash
git revert <sha>
git push
```

### 8. Recover after an accidental `reset --hard` or deleted branch

```bash
git reflog                          # find the commit before the mistake
git reset --hard HEAD@{2}           # or: git switch -c rescued <sha>
```

### 9. Update my feature branch with the latest `main`

```bash
git fetch origin
git rebase origin/main              # linear (then push --force-with-lease if already pushed)
# or, if the branch is shared with others:
git merge origin/main
```

### 10. Squash all my branch commits into one before a PR

```bash
git rebase -i origin/main           # mark all but the first as "squash"/"fixup"
# or:
git reset --soft $(git merge-base HEAD origin/main)
git commit -m "feat: profile page"
git push --force-with-lease
```

### 11. Push was rejected (remote has new commits)

```bash
git pull --rebase
git push
```

### 12. I need to switch branches but have unfinished work

```bash
git stash push -m "WIP"     # later: git stash pop
# or keep both checked out:
git worktree add ../hotfix main
```

### 13. Remove a file from Git but keep it locally (e.g. `.env` committed by mistake)

```bash
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "chore: stop tracking .env"
# ⚠️ If it contained secrets and was pushed: ROTATE the secrets, then purge history (§18.6)
```

### 14. See what changed in a PR / between my branch and main

```bash
git fetch origin
git diff origin/main...HEAD           # three dots: changes since branching
git log --oneline origin/main..HEAD   # my commits not in main
```

### 15. Find which commit broke something

```bash
git bisect start && git bisect bad && git bisect good <last-good-sha>
git bisect run npm test
git bisect reset
```

### 16. Who wrote this line and why?

```bash
git blame -L 120,130 src/payment.ts
git show <sha>                         # read the commit message & diff
```

### 17. Undo a merge that hasn't been pushed

```bash
git reset --hard ORIG_HEAD     # ORIG_HEAD = where HEAD was before the merge
```

### 18. Undo a merge that HAS been pushed

```bash
git revert -m 1 <merge-commit-sha>
```

⚠️ To re-merge that branch later, revert the revert first (`git revert <revert-sha>`), otherwise Git thinks those changes are already merged.

### 19. Resolve "both modified" conflicts by keeping one side for every file

```bash
git checkout --theirs .     # or --ours .
git add .
git commit
```

### 20. Delete local branches that were deleted on the remote

```bash
git fetch --prune
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D   # review first!
```

### 21. Change the author of the last commit

```bash
git commit --amend --author="Asha <asha@company.com>" --no-edit
```

### 22. Get a single file from another branch

```bash
git restore --source=feature/login -- src/Login.tsx
# older: git checkout feature/login -- src/Login.tsx
```

### 23. Create a branch from a specific old commit

```bash
git switch -c investigate abc123
```

### 24. Temporarily inspect an old version, then come back

```bash
git switch --detach abc123   # detached HEAD — look around, run the app
git switch -                 # back to your branch
```

### 25. Rename `master` to `main`

```bash
git branch -m master main
git push -u origin main
# change the default branch on GitHub, then:
git push origin --delete master
```

### 26. Make Git forget a file's permission (chmod) changes

```bash
git config core.fileMode false
```

### 27. Copy a commit's changes as a patch

```bash
git format-patch -1 abc123        # creates 0001-....patch
git am 0001-....patch              # apply with author info
git diff > my.patch && git apply my.patch   # plain diff patch
```

---

# 20. GitHub / GitLab Essentials

| Concept | Meaning |
| --- | --- |
| **Fork** | Your own server-side copy of someone else's repo (for contributing without write access) |
| **Pull Request (GitHub) / Merge Request (GitLab)** | Request to merge a branch, with review, discussion & CI |
| **Draft PR** | Work-in-progress PR, not ready for review |
| **Code review** | Comments, suggestions, approve / request changes |
| **Branch protection** | Rules for `main`: required reviews, checks, no force-push |
| **CODEOWNERS** | Auto-assigns reviewers by path |
| **Issues / Projects** | Task tracking; link with "Closes #42" in commits/PRs |
| **Actions / GitLab CI** | CI/CD pipelines triggered by pushes and PRs |
| **Releases** | Tag + changelog + assets |
| **Dependabot / Renovate** | Automated dependency update PRs |
| **Secret scanning / push protection** | Blocks pushes containing known secret formats |

### GitHub CLI (`gh`) quick hits

```bash
gh auth login
gh repo clone org/app
gh pr create --fill                 # PR from current branch
gh pr list
gh pr checkout 123                  # check out a PR locally to test it
gh pr view --web
gh pr merge 123 --squash --delete-branch
gh issue create
gh run watch                        # follow CI
```

### Minimal CI example (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request, push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npm test
```

---

# 21. Cheat Sheet ⭐

### Setup

| Command | Description |
| --- | --- |
| `git config --global user.name "Name"` | Set your name |
| `git config --global user.email "email"` | Set your email |
| `git config --list` | Show settings |
| `git init` | Create a repo |
| `git clone <url>` | Copy a remote repo |

### Everyday

| Command | Description |
| --- | --- |
| `git status` / `git status -sb` | What changed |
| `git add <file>` / `git add .` | Stage changes |
| `git add -p` | Stage interactively |
| `git commit -m "msg"` | Commit staged changes |
| `git commit --amend` | Edit last commit ⚠️ |
| `git diff` | Unstaged changes |
| `git diff --staged` | Staged changes |
| `git log --oneline --graph --all` | Visual history |
| `git show <sha>` | Inspect a commit |
| `git restore <file>` | Discard unstaged changes ⚠️ |
| `git restore --staged <file>` | Unstage |
| `git rm --cached <file>` | Untrack, keep locally |

### Branches

| Command | Description |
| --- | --- |
| `git branch` / `-a` / `-vv` | List branches |
| `git switch <branch>` | Change branch |
| `git switch -c <branch>` | Create + change |
| `git switch -` | Previous branch |
| `git branch -m <new>` | Rename current branch |
| `git branch -d <branch>` | Delete merged branch |
| `git branch -D <branch>` | Force delete ⚠️ |
| `git merge <branch>` | Merge into current |
| `git merge --abort` | Cancel merge |
| `git rebase <base>` | Replay commits on base ⚠️ |
| `git rebase -i HEAD~n` | Edit last n commits ⚠️ |
| `git rebase --continue / --abort` | During rebase |
| `git cherry-pick <sha>` | Copy a commit here |

### Remote

| Command | Description |
| --- | --- |
| `git remote -v` | List remotes |
| `git remote add origin <url>` | Add remote |
| `git fetch --prune` | Download changes, prune deleted |
| `git pull` / `git pull --rebase` | Fetch + integrate |
| `git push -u origin <branch>` | First push + set upstream |
| `git push` | Push |
| `git push --force-with-lease` | Safe force push ⚠️ |
| `git push origin --delete <branch>` | Delete remote branch |

### Undo

| Command | Description |
| --- | --- |
| `git reset --soft HEAD~1` | Undo commit, keep staged |
| `git reset HEAD~1` | Undo commit, keep unstaged |
| `git reset --hard HEAD~1` | Undo commit + discard ⚠️ |
| `git revert <sha>` | New commit undoing <sha> (safe for shared) |
| `git reflog` | History of HEAD — recover anything |
| `git clean -fd` | Delete untracked files ⚠️ (use `-n` first) |

### Stash & tags

| Command | Description |
| --- | --- |
| `git stash push -m "msg"` | Shelve changes |
| `git stash list` | List stashes |
| `git stash pop` | Re-apply + remove |
| `git stash apply stash@{n}` | Re-apply, keep |
| `git tag -a v1.0.0 -m "msg"` | Annotated tag |
| `git push --tags` | Push tags |

### Investigate

| Command | Description |
| --- | --- |
| `git blame <file>` | Who changed each line |
| `git log -S "text"` | Commits adding/removing text |
| `git grep "text"` | Search tracked files |
| `git bisect start / good / bad / reset` | Find bad commit |

### Reference syntax

| Syntax | Meaning |
| --- | --- |
| `HEAD` | Current commit |
| `HEAD~1`, `HEAD~3` | 1 / 3 commits back (first parent) |
| `HEAD^2` | Second parent of a merge commit |
| `main..feature` | Commits in feature not in main |
| `main...feature` | Commits in either but not both (for `diff`: changes since merge base) |
| `@{u}` | Upstream of current branch (`git log @{u}..`) |
| `HEAD@{2}` | Where HEAD was 2 moves ago (reflog) |
| `ORIG_HEAD` | HEAD before the last dangerous operation (merge, reset, rebase) |

---

# 22. Interview Questions & Answers ⭐

### Basics

**1. What is Git and why use it?**
A distributed version control system. Every developer has the full history locally, so most operations are fast and offline. It enables branching, collaboration, history tracking and safe rollback.

**2. Git vs GitHub?**
Git is the version control tool. GitHub (like GitLab and Bitbucket) is a hosting platform for Git repos that adds pull requests, code review, issues, CI/CD and access control.

**3. What are the three areas in Git?**
Working directory (your files), staging area/index (the next commit being prepared), and the repository (committed history in `.git`). Plus remotes for sharing.

**4. What is the staging area for?**
To build commits deliberately: choose exactly which changes (even which lines with `add -p`) go into the next commit, producing clean, focused commits.

**5. What is a commit?**
An immutable snapshot of the project, identified by a SHA hash, with a pointer to a tree (the files), its parent commit(s), the author, the committer and a message.

**6. What is HEAD? What is a detached HEAD?**
HEAD points to your current position — normally a branch name. In detached HEAD state it points directly at a commit (after checking out a SHA or tag). New commits there aren't on any branch, so create a branch to keep them.

**7. What is a branch, internally?**
A movable pointer: a file under `.git/refs/heads/` containing a commit hash. That's why creating and switching branches is nearly instant.

**8. What does `git clone` do?**
Copies the entire repository: all objects, all branches as remote-tracking refs, and sets up `origin`. Then it checks out the default branch.

### Everyday commands

**9. `git fetch` vs `git pull`?**
`fetch` downloads remote commits and updates `origin/*` without touching your work. `pull` is fetch plus merge (or rebase) into your current branch.

**10. `git merge` vs `git rebase`?**
Merge combines histories with a merge commit and preserves exactly what happened. Rebase replays your commits on top of another base, creating new commits and a linear history. Use rebase for local or feature-branch cleanup; never rebase shared history; use merge (or the PR merge button) to integrate into `main`.

**11. What is a fast-forward merge?**
When the target branch has no new commits since the feature branched, Git just moves the pointer forward. No merge commit is created. `--no-ff` forces one.

**12. How do you resolve a merge conflict?**
Run `git status` to find the conflicted files. Edit each one to the correct final content and remove the markers, then `git add` the files and `git commit` (or `merge --continue`). Use `--abort` to cancel. Afterwards, run the tests to verify.

**13. `git reset --soft` vs `--mixed` vs `--hard`?**
All three move the branch pointer. Soft keeps the changes staged. Mixed (the default) keeps them in the working directory but unstaged. Hard discards them entirely.

**14. `git reset` vs `git revert`?**
`reset` moves the branch back, rewriting history, so it's for unpushed commits. `revert` creates a new commit that undoes a previous one, preserving history, so it's safe for shared branches.

**15. `git restore` vs `git reset` vs `git checkout`?**
`restore` works on files: it discards working-tree changes or unstages them. `reset` moves branch pointers (and can unstage). `checkout` is the older multi-purpose command for switching branches and restoring files, now split into `switch` and `restore`.

**16. What does `git stash` do?**
It saves uncommitted changes onto a stack and cleans the working directory, so you can switch context. `pop` or `apply` brings them back. Add `-u` to include untracked files.

**17. What is `git cherry-pick`?**
Applies the changes from specific commits onto the current branch as new commits. It's used for backporting fixes to release branches or moving a commit made on the wrong branch.

**18. How do you undo the last commit?**
If it isn't pushed: `git reset --soft HEAD~1` (keeps the changes) or `--hard` (discards them). If it is pushed: `git revert HEAD`.

**19. How do you change the last commit message?**
`git commit --amend -m "new message"`. If it was already pushed to your own branch, follow with `git push --force-with-lease`.

**20. How do you stop tracking a file that's already committed?**
`git rm --cached file`, add it to `.gitignore`, then commit. If it contained secrets, rotate them and purge the history with `git filter-repo`.

### Intermediate / advanced

**21. What is `git reflog` and when do you use it?**
A local log of every position HEAD has been at. Use it to recover commits after a bad reset, rebase, amend or branch deletion.

**22. What is interactive rebase used for?**
Cleaning up commits before sharing: squashing, fixing up, rewording, reordering, dropping, splitting.

**23. What is the golden rule of rebasing?**
Never rebase commits that exist outside your local repository and that others may have based work on. It rewrites history and causes divergence for everyone else.

**24. `--force` vs `--force-with-lease`?**
`--force` overwrites the remote branch unconditionally. `--force-with-lease` only overwrites it if the remote is still at the commit you last fetched, so it refuses rather than deleting someone else's new commits.

**25. What is `git bisect`?**
A binary search over history to find the commit that introduced a bug. You mark commits good or bad, or automate it with `bisect run <test-script>`.

**26. What is `git blame`?**
Shows the last commit and author for each line of a file. It's used to understand why code changed (by reading that commit), not to blame people.

**27. What is a merge commit with two parents, and how do you revert one?**
A merge commit records both histories joining. Revert it with `git revert -m 1 <sha>`, keeping parent 1 (usually `main`) as the mainline.

**28. Double dot vs triple dot?**
`git log A..B` shows commits reachable from B but not A. `git diff A...B` shows changes on B since the merge base of A and B, which is what a PR diff shows.

**29. What are Git objects?**
Blob (file content), tree (directory listing), commit (snapshot plus metadata plus parents) and annotated tag. All are content-addressed by hash.

**30. Does Git store diffs or snapshots?**
Conceptually snapshots: each commit references a full tree. Unchanged files reuse the same blobs, and packfiles compress storage with deltas.

**31. What is `.gitignore`, and why doesn't it work for some files?**
It lists patterns for untracked files to ignore. It doesn't affect files that are already tracked, so untrack them with `git rm --cached` first.

**32. Lightweight vs annotated tags?**
A lightweight tag is just a pointer. An annotated tag is a full object with tagger, date and message, and it can be signed. Use annotated tags for releases.

**33. What is `HEAD~` vs `HEAD^`?**
`HEAD~n` goes n generations back along first parents. `HEAD^n` selects the nth parent of a merge commit. `HEAD~1` equals `HEAD^`.

**34. What is `git worktree`?**
Lets you check out multiple branches of the same repo in different folders at the same time, sharing one `.git`.

**35. What are Git hooks?**
Scripts triggered by Git events (`pre-commit`, `commit-msg`, `pre-push`), used to lint, format, enforce commit messages or run tests. They're shared via Husky or `core.hooksPath`.

**36. What is a submodule?**
A pointer to a specific commit of another repository, embedded in your repo. Useful but cumbersome. Alternatives are package managers, monorepos or subtree.

### Workflow & scenario questions

**37. Describe your Git workflow at work.**
Example answer: "We use GitHub Flow with protected `main`. I branch from `main` with a ticket-based name, make small Conventional Commits, rebase on `origin/main` regularly, push and open a PR early. CI runs lint and tests, at least one reviewer approves, and we squash-merge so `main` has one commit per PR, then delete the branch. Releases are tagged automatically."

**38. Git Flow vs GitHub Flow vs trunk-based?**
Git Flow uses `develop`, release and hotfix branches, which suits scheduled versioned releases. GitHub Flow is `main` plus short-lived feature branches and PRs, which suits continuous deployment. Trunk-based means everyone integrates into `main` at least daily with feature flags, which suits high-velocity CI/CD.

**39. Squash merge vs rebase merge vs merge commit?**
A merge commit keeps all commits plus a merge commit. Squash merge collapses the PR into one commit (clean `main`, but individual commits are lost). Rebase merge gives a linear history that keeps every commit.

**40. You accidentally committed an API key and pushed. What do you do?**
1. **Revoke/rotate the key immediately**, since the key is compromised regardless.
2. Remove it from the code, use env variables or a secret manager, and add the file to `.gitignore`.
3. Purge it from history with `git filter-repo` (or BFG), force-push, and ask the team to re-clone.
4. Ask the host to clear cached views if needed.
5. Prevent recurrence with secret scanning, push protection and pre-commit hooks.

**41. Your push was rejected. Why, and how do you fix it?**
The remote has commits you don't have (non-fast-forward). Run `git pull --rebase` (or a merge), resolve any conflicts and push again. Never force-push over a shared branch.

**42. You did `git reset --hard` and lost commits. Can you recover?**
Yes, if they were committed. Find them in `git reflog`, then `git reset --hard <sha>` or create a branch from it. Uncommitted changes discarded by `--hard` are generally not recoverable.

**43. How do you keep a long-running feature branch up to date?**
Regularly `git fetch` and rebase onto `origin/main` (if the branch is only yours) or merge `main` into it (if it's shared). Keep branches short-lived, or use feature flags to merge partial work early.

**44. How do you review someone's PR locally?**
`gh pr checkout 123`, or `git fetch origin pull/123/head:pr-123 && git switch pr-123`. Then run it and test.

**45. How would you find when a bug was introduced?**
Use `git bisect` between a known good tag and the current bad commit, ideally with an automated test (`bisect run`). Then read the culprit commit with `git show`.

---

# 23. Practice Exercises

Do these in a throwaway repo (`mkdir git-practice && cd git-practice && git init`).

1. **Basics:** create 3 files, commit them in 2 separate commits using `git add -p` to split changes in one file.
2. **Branch & merge:** create `feature/a`, commit, then add a commit on `main`; merge with a merge commit. Repeat with a fast-forward merge.
3. **Conflict:** change the same line on two branches; merge; resolve; then repeat with rebase and notice ours/theirs swap.
4. **Interactive rebase:** make 5 messy commits ("wip", "fix typo"…); clean them into 2 meaningful commits.
5. **Undo:** practise `reset --soft`, `reset --mixed`, `reset --hard`, then recover the "lost" commit with `reflog`.
6. **Revert:** revert a commit, then revert the revert.
7. **Stash:** stash work, switch branches, make a hotfix commit, return and pop.
8. **Cherry-pick:** copy one commit from `feature` to `release`.
9. **Remote:** create an empty GitHub repo, push with `-u`, clone it into a second folder, make conflicting changes in both, and resolve the rejected push with `pull --rebase`.
10. **Bisect:** create 20 commits where commit 13 writes the word `BUG` into `file.txt`. Then find it automatically:

    ```bash
    git bisect start HEAD <first-commit-sha>
    git bisect run sh -c '! grep -q BUG file.txt'   # exit 0 (good) when BUG is absent, 1 (bad) when present
    git bisect reset
    ```
11. **Tags:** create an annotated tag `v1.0.0`, push it, branch a hotfix from it.
12. **Ignore:** commit `.env` by mistake, then untrack it properly.

---

## ✅ Final Revision Checklist

- [ ] Explain working directory → staging → repository → remote
- [ ] Explain commits, branches and HEAD as pointers
- [ ] Use status / add -p / commit / diff / log fluently
- [ ] Create, switch, rename, delete branches (local & remote)
- [ ] Merge (ff vs no-ff vs squash) and resolve conflicts confidently
- [ ] Rebase and interactive rebase; state the golden rule
- [ ] fetch vs pull; push -u; force-with-lease
- [ ] reset (3 modes) vs revert vs restore; recover with reflog
- [ ] stash, cherry-pick, tags, blame, bisect
- [ ] Describe GitHub Flow / Git Flow / trunk-based and PR merge strategies
- [ ] Explain Git objects and the `.git` folder
- [ ] Handle the "leaked secret" and "rejected push" scenarios

> **Golden habits:** commit small and often · write meaningful messages · pull/rebase frequently · never rewrite shared history · use `--force-with-lease`, never plain `--force` · when in doubt, `git status` — and remember `git reflog` has your back.
