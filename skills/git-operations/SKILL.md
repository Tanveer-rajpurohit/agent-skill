---
name: git-operations
description: >
  Git workflows, commit conventions, reverting commits, viewing history,
  resolving merge conflicts, branch management, PR prep. Trigger for:
  "commit this", "revert last commit", "how do I undo", "merge conflict",
  "view git history", "what changed", "create a PR", "branch strategy",
  "git log", "cherry pick", "stash", "rebase", "push to remote",
  "see my last commits", "squash commits". Based on gitflow conventions
  and Tanveer's conventional-commit standard.
---

# Git Operations Skill

Reference: PatrickJS/awesome-cursorrules gitflow.mdc + Tanveer's conventions.

## Commit Format (Always Use This)

```
type(scope): short description in imperative mood

[optional body: what and why, not how]

[optional footer: BREAKING CHANGE, closes #issue]
```

### Types
| Type | When |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that's not a feat or fix |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `docs` | Documentation only |
| `chore` | Build, deps, config — no production code |
| `ci` | CI/CD pipeline changes |

### Good Commit Examples
```bash
git commit -m "feat(auth): add refresh token rotation"
git commit -m "fix(websocket): close dead connections on ping timeout"
git commit -m "perf(rag): reduce embedding latency by batching requests"
git commit -m "refactor(user): extract password hashing to separate service"
git commit -m "chore(deps): update mediasoup to v3.14.0"
```

### Bad Commits (Never)
```bash
git commit -m "fix"           # no context
git commit -m "WIP"           # not a real commit
git commit -m "changes"       # says nothing
git commit -m "Updated code"  # says nothing
```

---

## Viewing History

```bash
# Last 10 commits, one line each
git log --oneline -10

# Full log with author and date
git log --pretty=format:"%h %ad %s" --date=short -10

# What changed in the last commit
git show HEAD

# What changed in a specific commit
git show abc1234

# All commits that touched a specific file
git log --oneline -- src/handlers/auth.go

# Who changed what line in a file (blame)
git blame src/handlers/auth.go

# Graph view of branches
git log --oneline --graph --all -15

# See difference between two branches
git diff main..feature/my-branch --stat
```

---

## Undoing Changes

### Not committed yet
```bash
# Discard changes to one file (dangerous — unrecoverable)
git checkout -- src/handlers/auth.go

# Discard ALL unstaged changes (dangerous)
git checkout -- .

# Unstage a file (keep changes, just remove from staging)
git reset HEAD src/handlers/auth.go

# Save work for later without committing (stash)
git stash
git stash pop          # bring it back
git stash list         # see all stashes
git stash drop         # delete top stash
```

### Undo the last commit (already committed, not pushed)
```bash
# Keep all changes, just undo the commit
git reset --soft HEAD~1

# Undo commit AND unstage everything (changes still in files)
git reset --mixed HEAD~1

# Nuclear: undo commit AND discard all changes (UNRECOVERABLE)
git reset --hard HEAD~1
```

### Undo a commit that's already pushed (SAFE — creates new commit)
```bash
# Creates a new "revert" commit that undoes the target commit
git revert abc1234

# Revert without immediately committing (review first)
git revert --no-commit abc1234
git status     # check what changed
git commit -m "revert: undo broken auth middleware"
```

### Undo multiple commits (already pushed)
```bash
# Revert a range (oldest first in the range notation)
git revert abc1234..def5678

# Or revert the last 3 commits
git revert HEAD~3..HEAD
```

### Rule: reset vs revert
- `git reset` — rewrites history. OK for local, **never** on pushed branches
- `git revert` — adds a new commit. **Always safe** on shared branches

---

## Merge Conflicts

### When you see this:
```
CONFLICT (content): Merge conflict in src/handlers/auth.go
```

### Step-by-step resolution
```bash
# 1. See what's conflicted
git status

# 2. Open the conflicted file — look for markers
<<<<<<< HEAD
// Your version
const token = jwt.sign(payload, process.env.JWT_SECRET!)
=======
// Their version  
const token = jwt.sign(payload, config.jwt.secret, { expiresIn: '15m' })
>>>>>>> feature/jwt-config

# 3. Edit the file to keep what's correct
# (remove ALL markers — <<<, ===, >>>)
const token = jwt.sign(payload, config.jwt.secret, { expiresIn: '15m' })

# 4. Stage the resolved file
git add src/handlers/auth.go

# 5. Continue the merge
git merge --continue
# OR if you were rebasing:
git rebase --continue

# 6. If you want to abort and start over
git merge --abort
git rebase --abort
```

### Conflict prevention habits
```bash
# Before starting work, always pull latest
git fetch origin
git merge origin/main     # or rebase
# or shorter:
git pull --rebase origin main

# Keep branches short-lived — long-lived branches = more conflicts
```

---

## Branch Strategy (Gitflow)

```
main          ← production only, tagged releases
develop       ← integration branch
feature/*     ← new features, branch from develop
hotfix/*      ← urgent prod fix, branch from main
release/*     ← release prep, branch from develop
```

```bash
# Start a feature
git checkout develop
git pull origin develop
git checkout -b feature/123-add-room-persistence

# Finish a feature
git checkout develop
git merge --no-ff feature/123-add-room-persistence
git push origin develop
git branch -d feature/123-add-room-persistence

# Hotfix on production
git checkout main
git checkout -b hotfix/fix-transport-leak
# ... fix it ...
git checkout main && git merge --no-ff hotfix/fix-transport-leak
git checkout develop && git merge --no-ff hotfix/fix-transport-leak
git tag -a v1.2.1 -m "fix: transport memory leak"
```

---

## PR Prep

Before opening a PR, always run:
```bash
# 1. Rebase on latest main/develop (cleaner history than merge)
git fetch origin
git rebase origin/main

# 2. Run tests and lint
npm run lint && npm run test
# or for Go:
go vet ./... && go test ./...

# 3. Review your own diff
git diff origin/main..HEAD

# 4. Squash WIP commits into clean ones (if needed)
git rebase -i origin/main
# mark "squash" on WIP commits, keep meaningful ones

# 5. Push
git push origin feature/your-branch
```

### PR Description Template
```markdown
## What
[1-2 lines: what this PR does]

## Why
[why this change is needed]

## How
[key technical decisions, if non-obvious]

## Testing Done
- [ ] Unit tests pass
- [ ] Manual testing: [what you tested]
- [ ] Handles error cases: [what errors you checked]
```

---

## Cherry Pick (Take One Commit From Another Branch)
```bash
# Copy commit abc1234 from another branch into current branch
git cherry-pick abc1234

# Cherry pick a range
git cherry-pick abc1234..def5678

# Cherry pick without committing (review first)
git cherry-pick --no-commit abc1234
```

---

## Stash — Save Work Without Committing
```bash
git stash                          # stash everything
git stash push -m "WIP: auth fix"  # stash with a name
git stash list                     # see all stashes
git stash pop                      # apply and delete top stash
git stash apply stash@{1}          # apply specific stash (keep it)
git stash drop stash@{1}           # delete specific stash
git stash clear                    # delete ALL stashes
```
