# Git 101 Cookbook (OCNG 489/689)

This cookbook uses the class repository as the running example:

`git@github.com:odsl/ocng-489-689-2026-spring.git`

## 0) What Git Is

- `git` tracks file history in a local repository.
- `GitHub` hosts remote repositories and pull requests.
- A **commit** is a snapshot with a message.
- A **branch** is a movable pointer to commits.
- A **remote** is a named link to another repo (`origin`, `upstream`).

## 1) One-Time Setup

Check your Git install:

```bash
git --version
```

Set your identity (used in commit history):

```bash
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

Optional quality-of-life defaults:

```bash
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.editor "code --wait"   # or nano, vim, etc.
```

## 2) Clone the Class Repo

```bash
git clone git@github.com:odsl/ocng-489-689-2026-spring.git
cd ocng-489-689-2026-spring
```

Inspect repository state:

```bash
git status
git branch -a
git remote -v
```

## 3) Daily Core Commands (Local Work)

See changed files:

```bash
git status
```

See exact line changes:

```bash
git diff
```

Stage files:

```bash
git add path/to/file
git add .   # stage all tracked/untracked changes in current dir
```

Commit:

```bash
git commit -m "Add intro notes for Week 3"
```

Read history:

```bash
git log --oneline --graph --decorate -n 20
```

## 4) Branching Basics

Create and switch to a branch:

```bash
git switch -c yourname/topic-short-name
```

Switch branches:

```bash
git switch main
git switch yourname/topic-short-name
```

Delete a fully merged branch:

```bash
git branch -d yourname/topic-short-name
```

## 5) Push and Pull

Push your branch to remote:

```bash
git push -u origin yourname/topic-short-name
```

Later pushes on same branch:

```bash
git push
```

Update local `main` from remote:

```bash
git switch main
git pull origin main
```

## 6) Fork Workflow (Typical Student Workflow)

Use this when you do **not** have direct write access to `odsl/ocng-489-689-2026-spring`.

### Step A: Fork on GitHub

1. Open `https://github.com/odsl/ocng-489-689-2026-spring`.
2. Click **Fork** (creates `yourusername/ocng-489-689-2026-spring`).

### Step B: Clone Your Fork and Add Upstream

Clone your fork:

```bash
git clone git@github.com:yourusername/ocng-489-689-2026-spring.git
cd ocng-489-689-2026-spring
```

Add the class repo as `upstream`:

```bash
git remote add upstream git@github.com:odsl/ocng-489-689-2026-spring.git
git remote -v
```

Expected remote pattern:

- `origin` -> your fork
- `upstream` -> class repo

### Step C: Create Feature Branch, Commit, Push

```bash
git switch -c yourname/lab1-fix
# edit files
git add .
git commit -m "Fix typo in lab instructions"
git push -u origin yourname/lab1-fix
```

### Step D: Open Pull Request (PR)

On GitHub:

1. Open your fork branch.
2. Click **Compare & pull request**.
3. Base repo: `odsl/ocng-489-689-2026-spring`
4. Base branch: usually `main`
5. Fill title/description, submit PR.

## 7) Keep Your Fork Synced

Do this regularly before new work:

```bash
git switch main
git fetch upstream
git merge upstream/main
git push origin main
```

Then branch from updated `main`:

```bash
git switch -c yourname/new-task
```

## 8) Team Project Workflow (Recommended)

For team repos, avoid committing directly to `main`.

### Team Rules

- Protect `main` (require PR + at least 1 review).
- Use short-lived feature branches.
- Keep PRs small (one topic per PR).
- Merge only when CI/tests pass (if configured).

### Daily Team Flow

1. Update local main:

```bash
git switch main
git pull origin main
```

2. Create branch:

```bash
git switch -c teamname/feature-brief-name
```

3. Work and commit in logical chunks:

```bash
git add .
git commit -m "Implement parser for CTD metadata"
```

4. Push and open PR:

```bash
git push -u origin teamname/feature-brief-name
```

5. Address review comments with new commits; push again.
6. Merge PR on GitHub.
7. Delete merged branch (remote and local) to keep repo clean.

## 9) Resolve Merge Conflicts (Basic)

If conflict occurs during merge/pull:

```bash
git status
```

Open conflicted file and find markers:

```text
<<<<<<< HEAD
your version
=======
incoming version
>>>>>>> branch-name
```

Edit to desired final content, then:

```bash
git add path/to/conflicted-file
git commit
```

If conflict happened in a PR branch, push the resolved commit:

```bash
git push
```

## 10) Undo and Recovery (Safe Essentials)

Unstage file (keep edits):

```bash
git restore --staged path/to/file
```

Discard local unstaged edits in one file:

```bash
git restore path/to/file
```

See where `HEAD` moved recently:

```bash
git reflog -n 20
```

## 11) Practical Commit Message Pattern

Use present-tense, specific subjects:

- `Add Week 5 plotting example`
- `Fix wrong units in salinity calculation`
- `Refactor notebook setup for reproducibility`

Good pattern:

`<Verb> <what changed> [optional context]`

## 12) Quick Command Cheat Sheet

```bash
git status
git diff
git add .
git commit -m "Message"
git switch -c my/branch
git switch main
git pull origin main
git push -u origin my/branch
git fetch upstream
git merge upstream/main
git log --oneline --graph --decorate -n 20
```

## 13) Suggested Classroom Policy

- Every assignment update should come through a PR.
- No force-push to shared `main`.
- PR title should include team name and task.
- At least one teammate review before merge.
- Resolve conflicts in your branch, not in `main`.

---

If you are new to Git, focus on this loop first:

1. `pull main`
2. `create branch`
3. `edit + commit`
4. `push`
5. `open PR`
6. `merge after review`

