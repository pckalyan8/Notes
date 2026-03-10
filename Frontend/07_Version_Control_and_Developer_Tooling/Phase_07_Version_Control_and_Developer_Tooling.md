# Phase 7 — Version Control & Developer Tooling
## Frontend Angular Mastery Roadmap
> **Duration:** 1–2 weeks | **Goal:** Professional development workflow

---

## Table of Contents

1. [7.1 — Git Fundamentals](#71--git-fundamentals)
2. [7.2 — Git Advanced](#72--git-advanced)
3. [7.3 — Branching Strategies](#73--branching-strategies)
4. [7.4 — Code Review & Collaboration](#74--code-review--collaboration)
5. [7.5 — Developer Tools (Browser DevTools)](#75--developer-tools-browser-devtools)
6. [7.6 — IDE & Extensions (VS Code Focus)](#76--ide--extensions-vs-code-focus)

---

## 7.1 — Git Fundamentals

### What Is Git and Why Does It Matter?

Git is a **distributed version control system** created by Linus Torvalds in 2005 to manage the Linux kernel source code. The word "distributed" is key — unlike older centralised systems (like SVN), every developer who clones a Git repository gets a full copy of the entire project history on their own machine. There is no single server whose failure would destroy the project's history.

As a frontend developer, Git is not optional. It is the universal standard for tracking changes, collaborating with teammates, rolling back mistakes, and shipping code safely. Every professional engineering workflow — pull requests, CI/CD pipelines, code reviews, automated deployments — is built on top of Git.

Before diving into commands, it helps to understand how Git stores data. Unlike most other version control systems that store files and the changes made to them (a delta approach), **Git stores snapshots**. Each commit is a complete snapshot of every tracked file in the project at that moment in time — not a diff from the previous state. This architecture is what makes Git so fast and what enables branching to be nearly instantaneous.

---

### The Three Areas of Git

Every Git repository has three fundamental areas that you need to understand deeply, because every Git command is essentially moving content between them:

**The Working Directory** is your actual file system — the files you can open in your editor, edit, and run. Changes you make here are "unstaged" and Git is aware of them but has not recorded them yet.

**The Staging Area (Index)** is a preparation zone. When you run `git add`, you are moving changes from the working directory into the staging area. You are essentially saying "I want these specific changes to be part of my next commit." This granular control is powerful — you can stage only part of your changes and leave other modifications out of a commit.

**The Repository (.git directory)** is the actual database where Git permanently stores your history. When you run `git commit`, everything in the staging area is packaged into a new snapshot and stored here permanently.

```
Working Directory  →  git add  →  Staging Area  →  git commit  →  Repository
       ↑                                                                ↓
       └─────────────────── git checkout ──────────────────────────────┘
```

---

### Initialising and Cloning

You start a Git repository in one of two ways. Either you initialise a fresh one in an existing directory, or you clone a remote repository.

```bash
# Start tracking an existing project with Git
git init

# This creates a hidden .git/ directory — this IS the repository.
# Delete .git/ and you delete all version history.

# Clone an existing repository from a remote (e.g., GitHub)
git clone https://github.com/angular/angular.git

# Clone into a specific folder name instead of the default
git clone https://github.com/angular/angular.git my-angular-fork

# Clone only the latest snapshot, not the full history (faster for large repos)
git clone --depth 1 https://github.com/angular/angular.git
```

---

### Tracking Changes: add, status, diff, commit

After making changes to files, this is the typical workflow:

```bash
# See what Git knows about the current state of your working directory
git status

# Typical output:
# Changes not staged for commit:
#   modified:   src/app/app.component.ts
# Untracked files:
#   src/app/new-feature.component.ts

# Stage a specific file
git add src/app/app.component.ts

# Stage all changes in the current directory (use carefully)
git add .

# Stage only part of a file interactively (very useful)
# Git will show you each changed "hunk" and ask: stage this? (y/n/s to split)
git add -p src/app/app.component.ts

# See what is staged (what will go into the next commit)
git diff --staged

# See what is changed but not yet staged
git diff

# Create a commit with everything in the staging area
git commit -m "feat(auth): add JWT token refresh logic"

# Stage all tracked modified files AND commit in one step
# (does NOT add untracked/new files)
git commit -am "fix(login): handle empty password field"
```

> **Best Practice:** Commit early and commit often. Small, focused commits that represent a single logical change are far easier to review, revert, and understand six months later than massive commits that change dozens of files. Think of each commit as a sentence in the story of your codebase.

---

### Pushing and Pulling: Syncing with Remotes

A **remote** is a version of your repository hosted somewhere else — typically on GitHub, GitLab, or Bitbucket. The default remote is named `origin` by convention.

```bash
# See configured remotes
git remote -v

# Add a remote
git remote add origin https://github.com/your-username/your-repo.git

# Push your local branch to the remote
git push origin main

# Push and set the upstream tracking relationship (only needed once per branch)
# After this, a bare "git push" knows where to push
git push -u origin feature/my-new-feature

# Fetch changes from the remote WITHOUT merging them into your local branch
# This updates your "remote tracking branches" (origin/main, etc.)
git fetch origin

# Pull = fetch + merge (or fetch + rebase if configured)
git pull origin main

# Pull using rebase instead of merge (keeps history linear — preferred in many teams)
git pull --rebase origin main
```

The difference between `fetch` and `pull` is important and often misunderstood. `git fetch` downloads the changes from the remote and updates your remote-tracking branches (like `origin/main`) but does NOT touch your local working branch. It is completely safe and non-destructive — you can always run it without fear. `git pull` does a fetch and then immediately tries to integrate those remote changes into your current branch. Many experienced developers prefer to always `fetch` first, inspect what changed, and then decide how to integrate.

---

### Branching: branch, checkout, switch

Branching is one of Git's greatest strengths. Creating a branch in Git is nearly instantaneous because a branch is simply a lightweight pointer to a commit — Git doesn't copy files. The special pointer called `HEAD` always points to the commit or branch you currently have checked out.

```bash
# List all local branches (* marks the current branch)
git branch

# List all branches including remote tracking branches
git branch -a

# Create a new branch
git branch feature/user-authentication

# Switch to an existing branch (modern command)
git switch feature/user-authentication

# Create AND switch to a new branch in one step (most common usage)
git switch -c feature/user-authentication

# Old equivalent (still works, you'll see this in many tutorials)
git checkout -b feature/user-authentication

# Delete a branch (safe — prevents deletion if branch is unmerged)
git branch -d feature/old-experiment

# Force delete (use with caution — discards unmerged work)
git branch -D feature/abandoned-experiment

# Rename current branch
git branch -m new-branch-name
```

---

### Merging: Fast-Forward vs 3-Way Merge

Merging integrates the work from one branch into another. The way Git merges depends on the history of the two branches.

**Fast-Forward Merge:** This is the simplest case. If the branch you are merging FROM is a direct descendant of the branch you are merging INTO (i.e., the target branch has not moved since the feature branch was created), Git simply moves the pointer forward. No new "merge commit" is created. The history stays perfectly linear.

```
Before:                   After fast-forward merge:
main:    A → B            main:    A → B → C → D
feature:     └→ C → D             (feature merged cleanly)
```

```bash
git switch main
git merge feature/quick-fix  # fast-forward happens automatically
```

**3-Way Merge:** If both branches have diverged — `main` has had new commits since the feature branch was created — Git cannot simply move a pointer. It needs to combine work from three points: the common ancestor commit, the tip of `main`, and the tip of the feature branch. Git creates a new **merge commit** that has two parents, preserving the full history of both lines of development.

```
Before:               After 3-way merge:
main:    A → B → E    main:    A → B → E → M (merge commit)
feature:     └→ C → D               ↗
                       feature: C → D
```

```bash
git switch main
git merge feature/long-running-feature
# Git opens an editor for you to write a merge commit message
```

> **Important:** When there are conflicting changes in both branches (both branches modified the same lines of the same file), Git cannot automatically merge them. It will mark the conflict in the file with `<<<<<<<`, `=======`, and `>>>>>>>` markers. You must manually edit the file to resolve the conflict, stage the resolved file, and then complete the merge with `git commit`.

```bash
# After resolving conflicts manually in your editor:
git add src/app/conflicted-file.ts
git commit  # Git will have a pre-filled merge commit message
```

---

### Rebasing: rebase and Interactive Rebase

Rebasing is an alternative to merging that rewrites history to produce a cleaner, linear result. Instead of creating a merge commit, `git rebase` takes your commits and "replays" them on top of the target branch as if they were written after the latest commit there.

```
Before rebase:            After: git rebase main (on feature branch)
main:    A → B → E        main:    A → B → E
feature:     └→ C → D     feature:         └→ C' → D'
                           (C and D are new commits with new hashes)
```

```bash
# Update your feature branch with the latest from main (keeping history linear)
git switch feature/my-feature
git rebase main

# If conflicts arise during rebase, resolve them, then:
git add .
git rebase --continue

# Or abort the rebase and go back to where you started
git rebase --abort
```

**Interactive Rebase** is one of Git's most powerful features. It lets you rewrite, reorder, squash, and edit commits in your branch's history before sharing it. This is how you clean up a messy work-in-progress branch into a polished sequence of meaningful commits before opening a pull request.

```bash
# Interactively rewrite the last 4 commits
git rebase -i HEAD~4

# Git opens an editor showing something like:
# pick a1b2c3d feat: add login form
# pick e4f5g6h fix typo
# pick i7j8k9l wip: half-done validation
# pick m1n2o3p feat: complete validation logic

# You can change the verb before each commit:
# pick   = keep as-is
# reword = keep but edit the commit message
# squash = meld into the previous commit (combines messages)
# fixup  = meld into previous commit (discards this message)
# drop   = remove this commit entirely
# edit   = pause rebase here to amend the commit

# A cleaned-up version might look like:
# pick   a1b2c3d feat: add login form
# squash e4f5g6h fix typo
# fixup  i7j8k9l wip: half-done validation
# reword m1n2o3p feat: complete validation logic
```

> **Critical Rule:** Never rebase commits that have already been pushed to a shared remote branch (like `main` or a shared `develop` branch). Rebasing rewrites commit hashes, which would create a diverged history for everyone else working from those commits. Rebase is safe only on commits that exist solely in your local or private remote branch.

---

### Stashing: Saving Work in Progress

Stashing lets you save your uncommitted changes to a temporary stack and restore a clean working directory — useful when you need to quickly switch contexts (like fixing an urgent bug) without committing half-done work.

```bash
# Save current changes to the stash stack
git stash

# Save with a descriptive message (highly recommended)
git stash push -m "WIP: user profile form validation"

# Also stash untracked (new) files
git stash push --include-untracked -m "WIP: new component draft"

# List all stashes
git stash list
# Output:
# stash@{0}: On feature/profile: WIP: user profile form validation
# stash@{1}: On main: WIP: experimental layout change

# Apply the most recent stash and remove it from the stack
git stash pop

# Apply a specific stash without removing it (safe to inspect first)
git stash apply stash@{1}

# Drop a specific stash (delete without applying)
git stash drop stash@{1}

# Clear all stashes
git stash clear
```

> **Best Practice:** Always provide a descriptive message when stashing with `git stash push -m "..."`. Without messages, `stash@{0}`, `stash@{1}` become meaningless after a few days. Also, stashes are not a substitute for commits — if you are stashing work that represents meaningful progress, commit it to a WIP branch instead.

---

### Tags: Marking Releases

Tags are pointers to specific commits, typically used to mark release versions. Unlike branches, tags do not move — they permanently point to the commit they were created on.

```bash
# Create a lightweight tag (just a pointer, no metadata)
git tag v1.0.0

# Create an annotated tag (recommended — stores tagger, date, message)
# This is what you use for real releases
git tag -a v1.0.0 -m "Release version 1.0.0 — initial production release"

# Tag a specific past commit
git tag -a v0.9.0 -m "Beta release" a1b2c3d

# List all tags
git tag

# Show tag details
git show v1.0.0

# Push a specific tag to remote (tags are NOT pushed with git push by default)
git push origin v1.0.0

# Push ALL tags at once
git push origin --tags

# Delete a local tag
git tag -d v0.9.0

# Delete a remote tag
git push origin --delete v0.9.0
```

---

### Remotes: origin, upstream, and fetch

In a typical open-source or forking workflow, you will work with two remotes:

**origin** is your personal fork on GitHub/GitLab — the copy you have write access to.

**upstream** is the original repository that you forked from — you typically only have read access and use it to pull in updates.

```bash
# See all configured remotes
git remote -v
# origin    https://github.com/your-username/angular.git (fetch)
# origin    https://github.com/your-username/angular.git (push)

# Add the upstream remote (after forking)
git remote add upstream https://github.com/angular/angular.git

# Fetch all branches from ALL remotes
git fetch --all

# Sync your fork's main with the original project's main
git switch main
git fetch upstream
git rebase upstream/main  # or: git merge upstream/main

# Push the updated main to your fork
git push origin main
```

> **Best Practice:** In a company project (not a fork), you typically only have `origin`. But in open-source contributions, always configure `upstream` so you can keep your fork current with `git fetch upstream && git rebase upstream/main`.

---

## 7.2 — Git Advanced

### Reflog: The Safety Net

The reflog is Git's internal log of every time `HEAD` moved — every checkout, commit, rebase, reset, and merge. It is **local only** (not pushed to remotes) and entries expire after 90 days by default. Understanding the reflog means you can almost always recover from mistakes that seem catastrophic — like accidentally running `git reset --hard` on the wrong branch.

```bash
# Show the reflog for HEAD
git reflog

# Typical output:
# a1b2c3d (HEAD -> main) HEAD@{0}: commit: feat: add dark mode
# e4f5g6h HEAD@{1}: commit: feat: user settings page
# i7j8k9l HEAD@{2}: checkout: moving from feature to main
# m1n2o3p HEAD@{3}: reset: moving to HEAD~1

# Oops! You accidentally ran "git reset --hard HEAD~3" and lost 3 commits.
# Reflog shows you the commit hash before the reset:
git reflog
# Find the hash of where you were BEFORE the bad reset (e.g., a1b2c3d)

# Recover by creating a new branch pointing to that commit
git switch -c recovery-branch a1b2c3d

# Or, if you want to restore main itself
git switch main
git reset --hard a1b2c3d
```

Think of the reflog as a 90-day undo history for your entire local Git state. If something seems lost in Git, the reflog is the first place to look.

---

### Cherry-Pick: Selecting Individual Commits

Cherry-pick lets you apply the changes from a specific commit onto your current branch, without merging the entire branch. This is useful for porting a hotfix from a release branch to main, or for pulling a single useful commit out of a long-running feature branch.

```bash
# Apply a single commit by its hash to the current branch
git cherry-pick a1b2c3d

# Cherry-pick a range of commits (from e4f5 up to and including a1b2)
git cherry-pick e4f5g6h^..a1b2c3d

# Cherry-pick without automatically creating a commit
# (stages the changes but lets you edit before committing)
git cherry-pick --no-commit a1b2c3d

# If there are conflicts during cherry-pick:
# 1. Resolve conflicts in your editor
git add .
git cherry-pick --continue

# Abort and restore the branch to its state before cherry-pick
git cherry-pick --abort
```

> **Important:** Cherry-pick creates a **new commit** with a different hash even though it contains the same changes as the original commit. This means if you cherry-pick from one branch and later merge that branch, Git may show the changes as already applied (since the content matches), or it may create a conflict if the context around the change has diverged. Use cherry-pick intentionally and sparingly.

---

### Bisect: Binary Search for Bugs

`git bisect` is a powerful debugging tool that uses binary search to find the exact commit that introduced a bug. You tell Git which commit is "good" (the bug doesn't exist) and which is "bad" (the bug exists), and Git checks out the midpoint commit for you to test. You mark it good or bad, and Git halves the search space. This process is logarithmic — finding the culprit among 1,024 commits takes at most 10 steps.

```bash
# Start the bisect session
git bisect start

# Mark the current commit as bad (the bug exists here)
git bisect bad

# Mark a known-good commit (where the bug definitely didn't exist)
git bisect good v2.0.0

# Git checks out a commit halfway between. Test your app.
# If the bug EXISTS in this commit:
git bisect bad

# If the bug DOES NOT EXIST in this commit:
git bisect good

# Repeat until Git identifies the first bad commit:
# "a1b2c3d is the first bad commit"

# End the session and return to where you started
git bisect reset

# Automate bisect with a test script (script exits 0 for good, non-zero for bad)
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
git bisect run npm test  # Git runs "npm test" at each step automatically
```

If you are debugging a regression in a large Angular app and you know it worked in a previous version, `git bisect` can save hours of manual investigation. It is one of the most underused Git commands by junior developers.

---

### Blame and Log: Investigating History

Understanding why code looks the way it does is crucial for maintenance. `git blame` and `git log` are your primary tools for this.

```bash
# Show who last modified each line of a file, and in which commit
git blame src/app/auth/auth.service.ts

# Blame output format:
# a1b2c3d4 (Alice Johnson  2024-11-15 14:32:01 +0000 42) async refreshToken() {
# ^^^^^^^^  ^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^
# hash      author          date/time                    line number

# Ignore whitespace changes in blame (very useful after reformatting)
git blame -w src/app/auth/auth.service.ts

# Show the log with a visual branch graph
git log --oneline --graph --all --decorate

# Show log for a specific file
git log --oneline src/app/auth/auth.service.ts

# Show detailed diff for each commit in the log
git log -p src/app/auth/auth.service.ts

# Search commit messages for a keyword
git log --oneline --grep="authentication"

# Find commits that added or removed a specific string (the "pickaxe" search)
# Extremely useful for finding where a function was introduced or removed
git log -S "refreshToken" --oneline

# Show commits between two dates
git log --after="2025-01-01" --before="2025-06-01" --oneline

# Show commits by a specific author
git log --author="Alice" --oneline
```

> **Best Practice:** `git log --oneline --graph --all --decorate` is so useful that most developers alias it. Add this to your global git config: `git config --global alias.lg "log --oneline --graph --all --decorate"`. Then `git lg` gives you a beautiful branch graph in seconds.

---

### Git Hooks: Automated Quality Gates

Git hooks are shell scripts that run automatically at specific points in the Git workflow. They live in the `.git/hooks/` directory of your repository. Because `.git/` is not committed, hooks are local by default — but tools like **Husky** allow you to commit hook configurations to your repository and share them with your team.

The most important hooks for frontend developers are:

**pre-commit** — runs before a commit is created. The commit is aborted if the script exits with a non-zero status. This is where you run linting and formatting checks.

**commit-msg** — runs after the user types their commit message but before the commit is finalised. Used to enforce commit message conventions (like Conventional Commits format).

**pre-push** — runs before `git push` sends data to the remote. Used to run the test suite to prevent pushing broken code.

```bash
# Setting up Husky (v9) in an Angular project
npm install --save-dev husky
npx husky init

# This creates a .husky/ directory with a sample pre-commit hook
# The "prepare" script in package.json is also added: "prepare": "husky"
```

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged  # run ESLint + Prettier only on staged files
```

```bash
# .husky/commit-msg
#!/bin/sh
npx --no -- commitlint --edit "$1"  # validate commit message format
```

```bash
# .husky/pre-push
#!/bin/sh
npm run test -- --watch=false  # run all tests before pushing
```

```json
// package.json — lint-staged configuration
{
  "lint-staged": {
    "*.{ts,html}": ["eslint --fix", "git add"],
    "*.{ts,html,scss,json,md}": ["prettier --write", "git add"]
  }
}
```

> **Best Practice:** Always run `lint-staged` rather than linting the entire project on every commit. It only processes files that are actually staged, making the pre-commit hook fast enough that developers don't bypass it with `git commit --no-verify`.

---

### Submodules

Git submodules allow you to embed one Git repository inside another as a subdirectory. This is used when you want to include a shared library or dependency as its own versioned repository rather than copying files.

```bash
# Add a submodule (links an external repo into a subdirectory)
git submodule add https://github.com/your-org/shared-utils.git libs/shared-utils

# This creates a .gitmodules file:
# [submodule "libs/shared-utils"]
#   path = libs/shared-utils
#   url = https://github.com/your-org/shared-utils.git

# Clone a repo that has submodules (initialise and populate them)
git clone --recurse-submodules https://github.com/your-org/main-app.git

# If you cloned without --recurse-submodules, initialise afterwards
git submodule update --init --recursive

# Update all submodules to their latest commit
git submodule update --remote
```

> **Important:** Submodules add complexity. Each developer must remember to initialise them, and updates to submodules are not automatic. In most modern frontend monorepo setups (using Nx or Turborepo), submodules have been largely replaced by workspace packages. Understand submodules for maintenance purposes, but prefer npm workspaces or monorepo tooling for new projects.

---

### Git Worktrees

Worktrees allow you to check out multiple branches simultaneously in separate directories from the same repository. This means you can work on a hotfix in one directory while keeping your in-progress feature branch untouched in another — no stashing required.

```bash
# Create a worktree for a hotfix branch in a sibling directory
git worktree add ../my-app-hotfix hotfix/critical-security-fix

# Now you have:
# ~/projects/my-app/           (your main working directory, on feature branch)
# ~/projects/my-app-hotfix/    (a new directory, on hotfix/critical-security-fix)

# List all worktrees
git worktree list

# Remove a worktree when done (the directory itself must be deleted manually)
git worktree remove ../my-app-hotfix
```

This is particularly useful for frontend developers who need to run both a production release branch and a development branch simultaneously to compare behaviour.

---

## 7.3 — Branching Strategies

### Why Branching Strategy Matters

Without an agreed branching strategy, a team's repository quickly becomes chaotic — long-lived branches that diverge massively from main, mysterious broken builds, and deployment confusion. A branching strategy is a contract that defines what branches exist, what they represent, when code moves between them, and who has the right to merge where.

---

### GitFlow: The Classic Enterprise Strategy

GitFlow, introduced by Vincent Driessen in 2010, is a robust strategy designed for projects with scheduled release cycles and multiple parallel versions in production. It defines a strict set of branch types and rules for how they interact.

**The branch types in GitFlow are:**

The `main` (or `master`) branch contains only production-ready code. Every commit on `main` is a release. It should be protected — nobody directly commits here.

The `develop` branch is the integration branch where all completed features are combined. It represents the latest state of development, which may not yet be stable enough to release.

`feature/*` branches are created from `develop` for each new feature. When the feature is complete, it is merged back into `develop`. Feature branches never interact with `main` directly.

`release/*` branches are cut from `develop` when a release is being prepared. Only bug fixes, documentation updates, and release metadata changes go here — no new features. When ready, the release branch is merged into both `main` (tagged with the version number) AND back into `develop` (so `develop` has the fixes too).

`hotfix/*` branches are created from `main` when a critical production bug is discovered. The fix is merged into both `main` (with a new patch version tag) AND `develop`.

```
main:     ──────●────────────────────────●─────────────────────●──
                │v1.0                    │v1.1                  │v1.1.1
                │                        │                      │
develop:  ──────┼────●───●───●───────────┼──●────●─────────────┼──
                │    │   │   │           │  │    │             │
feature:        │    A───┘   B───┘       │  │    │             │
release:        │            └───────────┘  │    │             │
hotfix:         │                           │    │             └──fix──┘
```

**When to use GitFlow:** It is well-suited for libraries, mobile apps, SaaS products with quarterly release cycles, and any project where multiple versions must be maintained simultaneously (e.g., v1.x and v2.x).

**When NOT to use GitFlow:** It is unnecessarily complex for projects that deploy to production continuously (multiple times per day). The overhead of managing release branches and maintaining parallel merges slows teams down.

---

### GitHub Flow: Simple, Continuous Delivery

GitHub Flow is a much simpler strategy designed for teams that deploy continuously. There are only two concepts: `main` (which is always deployable) and short-lived feature branches.

**The entire workflow:**

1. Create a branch from `main` with a descriptive name (`feature/user-avatars`, `fix/checkout-redirect-bug`).
2. Make commits on this branch.
3. Open a Pull Request when ready for review (or even earlier, as a Draft PR to get early feedback).
4. Discuss, review, and refine the code. Automated tests run on the PR.
5. Merge into `main` when approved and all checks pass.
6. Deploy `main` to production immediately (typically automated via CI/CD).
7. Delete the feature branch.

```bash
# GitHub Flow in practice
git switch main
git pull origin main
git switch -c feature/user-avatars

# ... make commits ...

git push -u origin feature/user-avatars
# Open PR on GitHub
# Get approval + green CI
# Merge PR
# Delete branch
git switch main
git pull origin main
git branch -d feature/user-avatars
```

**When to use GitHub Flow:** Teams that deploy to production daily, startups with rapid iteration, SaaS web applications, and most modern frontend teams. It is the most widely used strategy in the industry today.

---

### Trunk-Based Development

Trunk-Based Development (TBD) is the most extreme form of continuous integration — developers commit directly to `main` (the "trunk") multiple times per day, or use very short-lived branches (1–2 days maximum) that are merged without review.

This forces the team to keep code in a continuously releasable state at all times. It requires a strong culture of feature flags, comprehensive automated testing, and pair programming or continuous code review.

```bash
# Short-lived branch (max 1-2 days)
git switch -c task/add-loading-spinner
# ... a few small commits ...
git push origin task/add-loading-spinner
# Immediate review (or direct commit to trunk for very small changes)
# Merge same day
```

**Feature flags** are essential in TBD — you merge incomplete features behind a flag that is `false` in production, allowing you to keep the branch short-lived without exposing unfinished UI to users:

```typescript
// Angular feature flag service
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  isEnabled(flag: string): boolean {
    // Could come from a remote config service, environment, or local override
    const flags = environment.featureFlags ?? {};
    return flags[flag] ?? false;
  }
}

// In a component template
@if (featureFlags.isEnabled('new-checkout-flow')) {
  <app-new-checkout />
} @else {
  <app-old-checkout />
}
```

**When to use TBD:** Highly mature teams with excellent test coverage and CI/CD culture, large-scale platforms like Google and Facebook (which pioneered this approach), and any team prioritising maximum CI speed.

---

### Monorepo Branching Strategies

When your repository contains multiple applications and libraries (a monorepo, common with Nx in Angular), the branching strategy needs to account for the fact that different parts of the codebase have different release cadences.

The typical approach is still GitHub Flow or TBD, but CI is configured to only build and test the **affected projects** when a PR is merged. Tools like `nx affected` determine which apps and libraries are impacted by the changes in a given commit, running only the necessary pipelines.

```bash
# With Nx in a monorepo, only run tests for affected projects
npx nx affected --target=test --base=main --head=HEAD
```

---

### Release Management

Even on teams using GitHub Flow or TBD, some form of release management is needed. Modern practices include:

**Semantic Versioning (SemVer):** Version numbers follow `MAJOR.MINOR.PATCH`. A major version increment indicates breaking changes, minor indicates new backwards-compatible features, and patch indicates backwards-compatible bug fixes. Angular follows this strictly (`v17.0.0`, `v17.1.0`, `v17.0.1`).

**Automated releases with Conventional Commits:** When commit messages follow a structured format, tools like `semantic-release` or `release-please` can automatically determine the next version number, generate a CHANGELOG, create a GitHub Release, and publish to npm — all triggered by a merge to `main`.

---

## 7.4 — Code Review & Collaboration

### Why Code Review Matters

Code review is not gatekeeping — it is knowledge sharing, quality assurance, and collective ownership of the codebase. When done well, it catches bugs before production, spreads architectural knowledge across the team, onboards new developers, and produces higher-quality, more maintainable code. When done poorly, it becomes a bottleneck, a source of friction, and demoralises contributors.

---

### Writing Good Commit Messages: Conventional Commits

**Conventional Commits** is a specification for writing structured commit messages. Adopted by Angular, Vue, and hundreds of other projects, it makes the commit history machine-readable (for automated CHANGELOG generation and semantic versioning) and human-readable.

**The format:**

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

The `type` must be one of:

- `feat` — a new feature (triggers a MINOR version bump in semantic versioning)
- `fix` — a bug fix (triggers a PATCH version bump)
- `docs` — documentation changes only
- `style` — formatting, whitespace (no code logic change)
- `refactor` — code change that neither fixes a bug nor adds a feature
- `perf` — performance improvement
- `test` — adding or correcting tests
- `build` — changes to the build system or dependencies
- `ci` — CI configuration changes
- `chore` — other changes that don't modify src or test files

A `!` after the type/scope (or a `BREAKING CHANGE:` footer) indicates a breaking change and triggers a MAJOR version bump.

```bash
# Simple feature commit
git commit -m "feat(auth): add Google OAuth login button"

# Bug fix with scope
git commit -m "fix(forms): prevent double submission on checkout form"

# Breaking change (two ways to mark it)
git commit -m "feat(api)!: remove deprecated getUserById endpoint"

# Or equivalently with a footer
git commit -m "feat(api): remove deprecated getUserById endpoint

BREAKING CHANGE: getUserById has been removed. Use getUserByEmail instead."

# Documentation update
git commit -m "docs(readme): update Angular version requirements to v21"

# Performance improvement
git commit -m "perf(dashboard): defer chart rendering with @defer block"

# Dependency update
git commit -m "build(deps): upgrade Angular to 21.0.0"
```

> **Best Practice:** Write the subject line (first line) in the **imperative mood** — "add feature" not "added feature" or "adds feature". Think of it as completing the sentence: "If applied, this commit will **add Google OAuth login button**." Keep the subject line under 72 characters.

---

### CHANGELOG Generation

Once your team commits follow Conventional Commits format, you can generate a CHANGELOG automatically. The most popular tools are `conventional-changelog-cli` and `semantic-release`.

```bash
# Install
npm install --save-dev conventional-changelog-cli

# Generate CHANGELOG.md (appending new entries)
npx conventional-changelog -p angular -i CHANGELOG.md -s
```

The generated CHANGELOG groups commits by type and produces output like:

```markdown
## [2.1.0] - 2026-03-10

### Features
- **auth:** add Google OAuth login button (a1b2c3d)
- **dashboard:** add real-time notification feed (e4f5g6h)

### Bug Fixes
- **forms:** prevent double submission on checkout form (i7j8k9l)
- **router:** fix memory leak in navigation guard (m1n2o3p)

### Performance Improvements
- **dashboard:** defer chart rendering with @defer block (q1r2s3t)
```

---

### Pull Request Best Practices

A Pull Request (PR) or Merge Request (MR) is the central unit of collaboration in modern development. Here is how to write one that gets reviewed quickly and merged smoothly:

**From the author's perspective:**

Keep PRs small and focused. A PR that changes fewer than 400 lines of code gets reviewed quickly and thoroughly. A PR that changes 2,000 lines across 40 files will either sit unreviewed for days or get a rubber-stamp "LGTM" without real scrutiny. If a feature is large, break it into a sequence of smaller PRs — one to set up the data layer, one for the service, one for the component UI.

Write a meaningful PR description. Explain **why** the change is being made (not just what — the code shows what). Include screenshots or recordings for UI changes. Link to the relevant issue or ticket. List any testing steps reviewers should try.

```markdown
## Summary
Adds JWT token refresh logic to prevent users from being unexpectedly logged out.
The previous implementation only validated tokens on login, not on subsequent API calls.

Closes #247

## Changes
- `AuthService`: Added `refreshToken()` method that calls `/auth/refresh`
- `AuthInterceptor`: Added retry logic that refreshes the token on 401 responses
- `TokenStorageService`: Added token expiry parsing to detect stale tokens

## How to Test
1. Log in with any test account
2. Wait or manually expire the token in DevTools > Application > Cookies
3. Make any API call — it should silently refresh and succeed
4. Check the Network panel for the `/auth/refresh` call

## Screenshots
[screenshot of Network panel showing the refresh call]
```

**From the reviewer's perspective:**

Review the PR within 24 hours. Long wait times for reviews are one of the biggest frustrations in development. If you cannot review thoroughly right away, at least acknowledge the PR and give a rough timeline.

Be specific in your feedback. Instead of "this is wrong", say "This `switchMap` should be `concatMap` here because HTTP calls can arrive out of order, and `switchMap` would cancel in-flight requests (which would abandon form submissions). See [link] for an explanation."

Distinguish between blocking feedback and suggestions. Use language like: "**Blocker:** This will cause a memory leak because…" vs "**Suggestion:** You might consider using the `@defer` block here for better performance, but it's not a blocker."

Approve when the code is good enough to ship, not when it is perfect. Perfect is the enemy of shipped.

---

### Code Review Etiquette

Some principles that keep code review healthy and productive across teams:

**Review the code, not the person.** Say "This function is doing too many things" not "You wrote this function in a confusing way."

**Explain your reasoning.** Don't just say "change this to X" — say "change this to X because Y." Explanations turn reviews into learning opportunities.

**Ask questions instead of making demands.** "What happens if `user` is null here?" is more effective than "This will throw if user is null. Fix it."

**Use the Conventional Comments spec.** Many teams adopt a comment labelling convention:

```
praise:    Great catch using OnPush here — saves unnecessary renders.
nitpick:   Minor style thing, feel free to ignore: we usually alphabetise imports.
suggestion: Consider extracting this into a pipe so other components can reuse it.
issue:     This will throw a TypeError if items is undefined. We need a null check.
question:  Why is this using BehaviorSubject instead of signal() here?
blocker:   This interceptor will intercept ALL requests including to third-party APIs.
           We need to filter by domain. Blocking merge until resolved.
```

---

### GitHub/GitLab Features: Actions, Issues, Projects

**GitHub Actions** is a CI/CD and automation platform built directly into GitHub. Workflows are defined in YAML files in `.github/workflows/`. An Angular project typically has workflows for:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci   # uses package-lock.json for deterministic installs

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test -- --watch=false --browsers=ChromeHeadless

      - name: Build
        run: npm run build -- --configuration production
```

**GitHub Issues** are the standard way to track bugs, feature requests, and tasks. Best practice is to link every PR to an issue with `Closes #issue-number` in the PR description (or commit message), which auto-closes the issue when the PR merges.

**GitHub Projects** (and GitLab Boards) are Kanban-style project management tools. For frontend teams, a typical board has columns: Backlog → In Design → In Development → In Review → Done. Linking Issues and PRs to project cards gives the entire team visibility into the state of every piece of work.

---

## 7.5 — Developer Tools (Browser DevTools)

### Why DevTools Mastery Is Non-Negotiable

Chrome DevTools (and its equivalent in Firefox, Edge, and Safari) is the single most important tool in a frontend developer's daily workflow. It is where you debug JavaScript, diagnose layout problems, analyse performance, inspect network traffic, and profile memory usage. Spending a week truly learning DevTools will save you months of debugging time over the course of a career.

---

### The Elements Panel

The Elements panel gives you a live, editable view of the DOM tree and the CSS applied to each element.

**Inspecting and editing the DOM:** Right-click any element on the page and choose "Inspect" to jump to it in the Elements panel. You can double-click any attribute, text node, or tag name to edit it live. This is invaluable for rapid CSS iteration — you can try styles directly in the browser before writing them in your editor. Changes are not permanent (a page reload resets them), but they let you verify a fix before committing it.

**The Computed Styles pane** shows the final resolved value of every CSS property for the selected element, after all cascade and inheritance rules have been applied. When you cannot understand why an element looks the way it does, check Computed Styles — it shows the exact value and which rule applied it.

**Box Model visualiser:** At the bottom of the Computed pane is an interactive box model diagram showing exact pixel values for margin, border, padding, and content dimensions. Hover over any area and it highlights on the page.

**Forcing element states:** The `:hov` toggle button in the Styles pane lets you force the `:hover`, `:focus`, `:active`, `:focus-within`, `:focus-visible`, and `:visited` pseudo-states on an element. This is essential for debugging hover styles without having to keep your mouse perfectly still.

```
Elements panel tips:
- Press H on a selected element to toggle visibility (display:none / visible)
- Press Delete to remove the element from the DOM
- Drag and drop elements to reorder them in the DOM
- Right-click → "Store as global variable" to get a JS reference (saved as $temp)
- $0 in the Console always refers to the currently selected element
```

---

### The Console Panel

The Console is a JavaScript REPL (Read-Eval-Print Loop) that runs in the context of your page. Every `console.log()` in your application appears here, but it is far more capable than a log viewer.

**Console methods:**

```javascript
// Basic logging
console.log('Simple message');
console.warn('This is a warning');   // yellow
console.error('This is an error');   // red, with stack trace
console.info('Informational');

// Group related logs
console.group('HTTP Request');
console.log('URL:', url);
console.log('Params:', params);
console.groupEnd();

// Collapsible group (starts collapsed)
console.groupCollapsed('Auth flow');
console.log('Token fetched');
console.groupEnd();

// Log with CSS styling (useful for making important logs stand out)
console.log('%cAngular App Started', 'color: blue; font-size: 20px; font-weight: bold;');

// Table output for arrays of objects
const users = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];
console.table(users);

// Measure time
console.time('data-processing');
processLargeDataset();
console.timeEnd('data-processing');  // Output: data-processing: 142.3ms

// Count how many times a point is reached
console.count('render');  // render: 1, render: 2, etc.
console.countReset('render');

// Assert — only logs if the condition is FALSE
console.assert(users.length > 0, 'No users found!');

// Stack trace from current point
console.trace('called from here');
```

**The REPL:** You can type and execute any JavaScript directly in the Console. The `$0` shortcut refers to the currently selected element in the Elements panel, making it trivial to manipulate DOM nodes. `$()` is an alias for `document.querySelector()` and `$$()` for `document.querySelectorAll()`.

```javascript
// Useful Console shortcuts
$0                      // currently selected DOM element
$0.style.background = 'red'  // instantly style it
$('#app-root')          // querySelector shorthand
$$('button')            // querySelectorAll shorthand (returns array)
copy($$('img').map(img => img.src))  // copy all image URLs to clipboard
```

---

### The Network Panel

The Network panel shows every network request made by the page — HTML, CSS, JS, fonts, images, API calls — along with timing information, request headers, response headers, response body, and status codes.

**Key features and workflows:**

**Filtering requests:** Use the filter bar to show only specific types (Fetch/XHR for API calls, JS, CSS, Img, WS for WebSockets) or search by URL. The red dot next to WS shows active WebSocket connections.

**Request timing waterfall:** Each request shows a colour-coded bar representing its phases: DNS lookup (dark green), initial connection (orange), SSL negotiation (purple), time to first byte / TTFB (green), content download (blue). A long green bar (TTFB) means your server is slow to respond. A long blue bar means the response is large.

**Throttling:** The network speed dropdown lets you simulate slow connections — "Slow 3G", "Fast 3G", "Offline". This is essential for testing how your Angular app behaves on mobile networks. Always test your app under simulated throttling before shipping.

**Inspecting request and response:** Click any request to see: Headers (request and response), Payload (request body), Preview (formatted JSON/HTML response), Response (raw response text), Timing (detailed phase breakdown), Initiator (what code triggered this request — with a clickable call stack).

**Recording HAR files:** You can export the entire network log as a HAR (HTTP Archive) file — useful for sharing network debugging sessions with teammates or attaching to bug reports.

```
Network panel tips:
- "Disable cache" checkbox — essential during development to prevent cached responses
- Right-click a request → "Copy as cURL" — paste in terminal to replay the request
- Right-click a request → "Block request URL" — simulate a resource failing to load
- Shift + hover a request — highlights its initiator (red) and its dependencies (green)
```

---

### The Performance Panel

The Performance panel is where you diagnose runtime performance problems — janky animations, slow interactions, and excessive JavaScript execution.

**Recording a performance trace:** Click the record button, interact with your page, then stop. DevTools captures a detailed flame chart showing exactly what happened on the main thread, GPU, and network.

**Reading the flame chart:** Each bar in the flame chart represents a JavaScript function call. The width represents duration. Bars are stacked vertically — a tall stack means function A called B called C. Red triangles in the top bar indicate Long Tasks (> 50ms). Your goal is to eliminate Long Tasks or break them up.

**The Main thread lane:** Shows all JavaScript execution, style recalculations, layout, and paint events. Look for long purple (layout) and green (paint) bars — these indicate expensive rendering work.

**The Frames lane:** Shows each rendered frame. Frames above 60fps are green. Frames below are yellow or red, indicating dropped frames and visible jank.

**Performance panel in Angular debugging:**

```
Common Angular-specific things to look for:
- Long change detection cycles (look for [Angular] CD in the flame chart)
- Components that re-render unnecessarily (use OnPush + Signals to fix)
- Heavy ngFor re-renders without trackBy / track expressions
- Third-party scripts blocking the main thread
```

---

### The Memory Panel

The Memory panel helps you find memory leaks — situations where your app continually consumes more memory without releasing it, eventually slowing down or crashing.

**Heap Snapshots:** Take a snapshot of the JavaScript heap at a point in time. Compare two snapshots to see what objects were created between them and never garbage collected. Objects prefixed with `(detached)` in the snapshot are DOM nodes that have been removed from the document but are still referenced somewhere in JavaScript — a common memory leak pattern.

```
Memory leak debugging workflow:
1. Open Memory panel
2. Take Heap Snapshot 1 (baseline)
3. Perform the action you suspect is leaking (e.g., navigate to a component 10 times)
4. Take Heap Snapshot 2
5. In Snapshot 2, select "Comparison" view and compare to Snapshot 1
6. Sort by "# Delta" (new objects) — look for Angular component instances that should have been destroyed
```

**Allocation instrumentation:** Record allocations over time to see exactly which functions are allocating memory. This is useful for finding the source of a leak rather than just confirming one exists.

---

### The Application Panel

The Application panel exposes everything about the current origin's storage, service workers, and web app manifest.

**Storage inspection:** View and edit `localStorage`, `sessionStorage`, Cookies, IndexedDB, and the Cache API storage directly. You can add, edit, and delete values — invaluable for testing edge cases like "what happens if the auth token is malformed?" or "what happens if the cart data is corrupted?"

**Service Workers:** See registered service workers, their status (installing / waiting / active), and control them — you can update, unregister, or skip waiting from here. The "Update on reload" checkbox forces the latest service worker to activate on every page load, which is essential during SW development (otherwise the SW lifecycle means your changes take two page loads to take effect).

**Manifest:** Shows your Web App Manifest's parsed content and validates it. If you click "Add to homescreen", you can trigger the installation prompt.

**Background Services:** Logs Background Sync events, Push events, and Notification interactions — very useful for testing PWA functionality.

---

### Lighthouse Audits

Lighthouse is an automated auditing tool built into DevTools (also available as a CLI and CI integration). It audits your page across five dimensions: **Performance**, **Accessibility**, **Best Practices**, **SEO**, and **PWA**. Each is scored 0–100.

```
Running Lighthouse:
1. Open DevTools → Lighthouse tab
2. Select "Mobile" or "Desktop" (always test Mobile — it uses CPU throttling)
3. Check all categories
4. Click "Analyze page load"
5. Read the detailed report and fix the flagged issues
```

For an Angular application, the most impactful Lighthouse improvements are typically: reducing unused JavaScript (code splitting with `@defer` and lazy routes), optimising images (NgOptimizedImage + WebP), improving First Contentful Paint (inline critical CSS, preconnect to API origins), and implementing SSR for better initial LCP.

---

### The Coverage Panel

The Coverage panel records which JavaScript and CSS bytes are actually used during page load or user interaction. Unused bytes mean you are shipping dead code to users — increasing bundle size and slowing downloads.

```
Using Coverage:
1. Open Coverage panel (More Tools → Coverage)
2. Click the record button
3. Reload the page and interact normally
4. Stop recording
5. The table shows each file with a % usage bar and red/green line highlighting
   (red = not executed, green = executed)
```

For Angular apps, high CSS coverage is achievable through Angular's component-scoped styles. Low JS coverage on initial load is expected (and desirable) — it means your code splitting is working and unused routes haven't been loaded yet.

---

### The Sources Panel

The Sources panel is your in-browser debugger. It shows you the source files (or source-mapped originals) and lets you set breakpoints, step through code, and inspect state at any point.

**Setting breakpoints:**

```
Types of breakpoints:
- Line breakpoint: Click the line number in any source file
- Conditional breakpoint: Right-click line number → "Add conditional breakpoint"
  → Only pauses when expression is truthy (e.g., user.id === 42)
- Logpoint: Right-click → "Add logpoint" → Logs a message without pausing
  (like adding console.log without modifying the code)
- DOM breakpoint: Elements panel → right-click a node → "Break on subtree modifications"
- Event listener breakpoint: Sources → Event Listener Breakpoints panel
- XHR/Fetch breakpoint: Break when a URL matching a pattern is requested
```

**Navigating while paused:**

When the debugger pauses at a breakpoint, you can:
- **Step Over** (F10) — execute the current line and pause at the next
- **Step Into** (F11) — if the current line calls a function, step into that function
- **Step Out** (Shift+F11) — run until the current function returns
- **Continue** (F8) — run until the next breakpoint
- **Inspect variables** in the Scope panel — see local variables, closure variables, and globals
- **Add Watch expressions** — evaluate any expression at every pause
- **Edit and re-run code** — modify source in the editor, save (Ctrl+S) to apply changes without reloading (live edit, subject to limitations)

**Source maps:** Angular's build system generates source maps that tell DevTools how to map compiled JavaScript back to your original TypeScript files. When enabled (the default in development), breakpoints and stack traces show TypeScript line numbers — exactly what you wrote in your editor.

---

### Mobile Device Simulation

The Device Toolbar (Ctrl+Shift+M or the phone/tablet icon in DevTools) simulates mobile devices with configurable screen dimensions, pixel density (device pixel ratio), touch events, and user agent strings.

```
Device simulation capabilities:
- Choose from preset devices (iPhone 15, Pixel 8, iPad Pro, etc.)
- Set custom width/height
- Rotate between portrait and landscape
- Throttle CPU (1x / 4x / 6x slowdown to simulate mid-range phones)
- Throttle network (Slow 3G, Fast 3G)
- Simulate touch events (tap, swipe, pinch-zoom)
- Show media query breakpoints as vertical rulers
```

> **Best Practice:** Always test your Angular application in device simulation mode with **Mid-tier Mobile** preset — CPU throttling at 4x, network at Fast 3G — before considering a feature complete. Performance problems that are invisible on a developer laptop are immediately obvious under these constraints.

---

## 7.6 — IDE & Extensions (VS Code Focus)

### Why VS Code Is the Standard

Visual Studio Code has become the overwhelming favourite editor for frontend development, with over 75% of developers using it according to the 2024 Stack Overflow survey. It is fast, open-source, and has an unmatched ecosystem of extensions. For Angular development specifically, Microsoft's deep integration between VS Code and the TypeScript language server (which also powers Angular's compiler) makes the development experience uniquely productive.

---

### Workspace Settings vs User Settings

VS Code has two levels of configuration that apply in a specific order: **User Settings** are personal preferences that apply globally across all projects on your machine (font size, theme, keybindings). **Workspace Settings** are committed to the repository in `.vscode/settings.json` and apply only when that project is open — they override user settings and ensure every team member uses the same formatting rules and file associations.

```json
// .vscode/settings.json (committed to the repository — shared with team)
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[html]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.updateImportsOnFileMove.enabled": "always",
  "files.exclude": {
    "**/.git": true,
    "**/node_modules": true,
    "**/.angular": true
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.angular": true
  }
}
```

```json
// .vscode/extensions.json (committed — recommends extensions to new team members)
{
  "recommendations": [
    "angular.ng-template",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "eamodio.gitlens",
    "usernamehw.errorlens",
    "christian-kohler.path-intellisense",
    "nrwl.angular-console"
  ]
}
```

When a developer opens the project and VS Code sees `.vscode/extensions.json`, it automatically suggests installing the recommended extensions. This is how you standardise the team's tooling without mandating anything.

---

### Essential Extensions for Angular Development

**Angular Language Service (`angular.ng-template`)** is the single most important extension for Angular developers. It brings the Angular compiler's knowledge directly into the editor, enabling autocomplete in templates, type-checking of template expressions, navigation (Go to Definition works across components and templates), and real-time error highlighting. Without it, you are essentially writing Angular templates blind.

```
Angular Language Service provides:
- Autocomplete for component inputs, directives, pipes, and DOM properties
- Type errors in templates (e.g., passing a string where a number is expected)
- Go to Definition: Ctrl+click an @Input() name in a template → jumps to definition
- Find all references across templates and TypeScript
- Rename refactoring (rename a signal, it updates all template uses)
- Hover documentation for Angular-specific syntax (@if, @for, etc.)
```

**ESLint (`dbaeumer.vscode-eslint`)** integrates ESLint into VS Code, showing linting errors as you type and enabling fix-on-save. Works with `@angular-eslint` rules to enforce Angular-specific best practices.

**Prettier (`esbenp.prettier-vscode`)** formats your code automatically on save according to the project's `.prettierrc` configuration. Eliminates style arguments in code review — formatting is automatic and consistent.

**GitLens (`eamodio.gitlens`)** supercharges VS Code's built-in Git support. Its killer feature is **inline blame** — hovering over any line shows who last changed it, when, and with what commit message (with a click to see the full commit). It also provides a visual commit graph, file history timeline, and branch/PR comparisons without ever leaving the editor.

**Error Lens (`usernamehw.errorlens`)** displays ESLint, TypeScript, and compiler errors inline on the line where they occur — you see the error message directly in the editor without needing to hover or open the Problems panel. This makes errors impossible to miss.

**Path Intellisense (`christian-kohler.path-intellisense`)** provides autocomplete for file paths in `import` statements. Eliminates the common mistake of importing from the wrong relative path.

---

### Debugging Angular in VS Code (launch.json)

You can set breakpoints in your TypeScript source files and debug directly in VS Code — no browser DevTools required. This requires a `.vscode/launch.json` configuration.

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Chrome against localhost",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:4200",
      "webRoot": "${workspaceFolder}",
      "sourceMaps": true,
      "sourceMapPathOverrides": {
        "webpack:/*": "${webRoot}/*",
        "/./*": "${webRoot}/*",
        "/src/*": "${webRoot}/*",
        "/*": "*",
        "/./~/*": "${workspaceFolder}/node_modules/*"
      }
    },
    {
      "name": "Attach to Chrome",
      "type": "chrome",
      "request": "attach",
      "port": 9222,
      "webRoot": "${workspaceFolder}",
      "sourceMaps": true
    },
    {
      "name": "Debug Jest Tests",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "node",
      "runtimeArgs": [
        "--experimental-vm-modules",
        "${workspaceFolder}/node_modules/.bin/jest"
      ],
      "args": ["--runInBand"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

**Workflow:**

1. Start `ng serve` in the integrated terminal (Ctrl+`)
2. Press F5 (or Run → Start Debugging) to launch Chrome via the debugger
3. VS Code opens Chrome pointed at `localhost:4200`
4. Set breakpoints in any `.ts` file by clicking the gutter
5. Interact with the page — VS Code pauses when the breakpoint is hit
6. Use the Debug toolbar (Step Over, Step Into, Continue) or hover over variables to inspect state

This full-stack TypeScript debugging experience — being able to inspect an Angular service's state or step through an RxJS pipe — is far more efficient than relying on `console.log` statements.

---

### Multi-Root Workspaces

A multi-root workspace lets you open multiple separate repository folders as a single VS Code session. This is useful when you have a frontend Angular app and a backend Node.js API in separate repositories but work on them together.

```json
// my-project.code-workspace
{
  "folders": [
    {
      "name": "Frontend (Angular)",
      "path": "./frontend"
    },
    {
      "name": "Backend (Node.js)",
      "path": "./backend"
    },
    {
      "name": "Shared Types",
      "path": "./shared"
    }
  ],
  "settings": {
    "editor.formatOnSave": true
  }
}
```

Open with `code my-project.code-workspace` (or File → Open Workspace from File). All folders appear in the Explorer side panel, global search spans all folders, and the integrated terminal can be opened rooted in any folder.

---

### Tasks and Terminal Integration

VS Code's Tasks system allows you to run shell commands (like starting Angular dev server, running tests, or building) directly from the editor, with results shown in the integrated terminal. You can bind tasks to keyboard shortcuts or run them from the Command Palette.

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "ng serve",
      "type": "shell",
      "command": "ng serve --open",
      "group": "build",
      "isBackground": true,  // task runs continuously (doesn't block)
      "problemMatcher": {
        "owner": "typescript",
        "pattern": "$tsc",
        "background": {
          "activeOnStart": true,
          "beginsPattern": "Building...",
          "endsPattern": "compiled successfully"
        }
      },
      "presentation": {
        "reveal": "always",
        "panel": "new"
      }
    },
    {
      "label": "ng test (watch)",
      "type": "shell",
      "command": "ng test",
      "group": "test",
      "isBackground": true,
      "presentation": { "panel": "new" }
    },
    {
      "label": "ng build (production)",
      "type": "shell",
      "command": "ng build --configuration production",
      "group": {
        "kind": "build",
        "isDefault": true  // Ctrl+Shift+B runs this
      }
    }
  ]
}
```

The integrated terminal (Ctrl+`) is more than just a terminal — it understands the workspace context, has clickable file paths (Ctrl+click to open), and persists history between sessions. You can split the terminal to run `ng serve`, `ng test`, and git commands simultaneously.

---

### Remote Development (Dev Containers and Remote SSH)

VS Code's remote development extensions allow you to open a folder that lives on a remote machine or inside a Docker container as if it were local. This means every team member develops in an identical, reproducible environment.

**Dev Containers** (`.devcontainer/devcontainer.json`) define a Docker container as the development environment. The container has Node.js, Angular CLI, and any other tools pre-installed at pinned versions. Developers run VS Code locally, but all code execution, IntelliSense, and terminal commands happen inside the container.

```json
// .devcontainer/devcontainer.json
{
  "name": "Angular Development",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:22",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "angular.ng-template",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ],
      "settings": {
        "editor.formatOnSave": true
      }
    }
  },
  "postCreateCommand": "npm ci && npm install -g @angular/cli",
  "forwardPorts": [4200, 4201],  // Angular dev server port forwarded to localhost
  "remoteUser": "node"
}
```

Any developer who opens this repository in VS Code with the Dev Containers extension will be prompted to "Reopen in Container". VS Code builds the Docker image, installs the extensions inside it, and connects — the developer sees the same environment as every other team member, regardless of their host OS. This eliminates the "works on my machine" class of problems entirely.

**GitHub Codespaces** is built on the same `.devcontainer` specification — your repository is automatically available as a cloud-hosted development environment that any team member can launch from a browser with zero local setup.

---

## Summary and Key Takeaways

Understanding Git deeply is what separates a developer who can use version control from one who can wield it confidently during a production incident, a complex rebase, or a merge conflict at 11pm before a release. The commands are learnable in days, but the mental model — branches as pointers, commits as immutable snapshots, the three areas of the working directory, staging, and repository — takes weeks of real use to internalise.

Choosing the right branching strategy for your team is an architectural decision with long-term consequences. GitHub Flow is the right default for most product teams deploying continuously. GitFlow remains relevant for versioned software with long release cycles.

Writing Conventional Commits transforms your commit history from a personal scratch pad into shared, machine-readable documentation. Combined with GitHub Actions, it enables fully automated releases, versioning, and changelogs.

Mastering DevTools will pay dividends every single day. The Elements, Network, and Sources panels are the tools you reach for most often, but knowing the Performance and Memory panels is what separates developers who can diagnose hard bugs from those who just guess.

A properly configured VS Code environment with Angular Language Service, ESLint, Prettier, and GitLens is the foundation of productive daily work. Investing an hour in configuring `.vscode/settings.json` and `.vscode/extensions.json` for your team eliminates entire categories of inconsistency and tooling friction.

| Topic | The One Thing to Internalise |
|---|---|
| Git Fundamentals | Git stores snapshots, not diffs. A branch is just a pointer. |
| Git Advanced | The reflog is your safety net — almost nothing in Git is truly lost. |
| Branching Strategies | GitHub Flow is the right default. GitFlow is for scheduled-release software. |
| Code Review | Keep PRs small, explain your reasoning, and use Conventional Commits. |
| DevTools | `$0`, conditional breakpoints, and the Performance flame chart are your three biggest productivity unlocks. |
| VS Code | Angular Language Service + Error Lens + lint-on-save is the minimum viable Angular setup. |

---

*Phase 7 Complete — Next: Phase 8 — Package Managers & Module Systems*

*Document version: March 2026 | Angular 21 · TypeScript 5.9 · VS Code · Chrome DevTools*
