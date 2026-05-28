---
name: git-recovery
description: Recovering from git mistakes safely (undo commits, fix bad commits, resolve conflicts, recover lost work via reflog, undo merges/rebases/resets). TRIGGER when the user wants to "undo" anything in git, has merge/rebase conflicts, lost commits/branches, accidentally pushed/reset/discarded work, or asks how to safely revert changes. Emphasizes non-destructive recovery patterns and uses reflog before resorting to dangerous operations.
---

# git-recovery

Most "I broke git" problems are recoverable if you stop and check `git reflog` before doing anything destructive. The two rules:

1. **Local mistakes**: almost always reversible via `reflog` — git keeps every HEAD movement for ~90 days even if commits look "lost".
2. **Pushed mistakes**: prefer `git revert` (creates new commit that undoes) over `git push --force` (rewrites history, breaks others' clones).

## Step 0: Don't Panic — Check Reflog First

```bash
git reflog                # last 30 HEAD movements
git reflog --all          # also includes branches, stashes, etc.
git reflog show <branch>  # specific branch's history of moves
```

Reflog entries show every commit HEAD has been at. To recover any of them:
```bash
git checkout <hash>          # detached HEAD at that commit
git branch recovered <hash>  # create a branch pointing at it
git reset --hard <hash>      # move current branch back to it (DESTRUCTIVE — see below)
```

If you can find the SHA in reflog, you can recover. **Don't `git reset --hard` or `git clean -fd` until reflog has been checked.**

## Undoing the Most Recent Commit

```bash
# Keep changes in working tree (most common)
git reset --soft HEAD~1     # uncommit, keeps staged
git reset --mixed HEAD~1    # uncommit, unstages, keeps file changes
git reset HEAD~1            # = mixed (default)

# Throw away the commit's changes entirely
git reset --hard HEAD~1     # DESTRUCTIVE: lost edits go to reflog only

# If commit was already pushed: don't reset, revert
git revert HEAD             # creates a new commit that undoes the previous one
```

## Fixing the Most Recent Commit (not pushed)

```bash
# Wrong commit message
git commit --amend -m "correct message"

# Forgot to add a file
git add forgotten.py
git commit --amend --no-edit

# Wrong author
git commit --amend --author="Name <email>"
```

**Never `--amend` a commit that's already pushed** to a shared branch — it rewrites history and forces others to deal with it. (Exception: your own un-merged feature branch is fine; just use `git push --force-with-lease`.)

## Recovering "Lost" Branches / Commits

```bash
# Branch was deleted accidentally
git reflog                              # find the SHA where the branch was
git branch <name> <sha>                 # restore it

# Hard-reset away from work and now want it back
git reflog                              # find SHA before the reset
git reset --hard <sha>                  # restore to that point

# Stash got "lost"
git fsck --no-reflog | grep "dangling commit"   # finds orphaned stashes
git stash list                                  # check existing first
```

## Pushed Mistakes — Prefer Revert

```bash
# Single bad commit on main / shared branch
git revert <sha>                        # creates "Revert ..." commit
git push

# Multiple bad commits (say last 3)
git revert HEAD~2..HEAD                 # creates 3 revert commits
git push

# Bad merge commit
git revert -m 1 <merge-sha>             # `-m 1` keeps the parent that was the target
git push
```

If you MUST rewrite shared history (rare, requires team coordination):
```bash
git push --force-with-lease             # safer than --force; aborts if remote moved
```

## Conflict Resolution

### Merge conflicts
```bash
git status                              # see which files have conflicts
# Edit each file, look for <<<<<<< / ======= / >>>>>>> markers
# Pick one side, both, or write something new

git add <resolved-file>
git merge --continue                    # or git commit if it doesn't ask

# Abort the merge entirely
git merge --abort
```

### Rebase conflicts
```bash
git rebase --continue                   # after fixing files
git rebase --skip                       # skip this commit (rare)
git rebase --abort                      # back to where you started
```

### Pull conflicts
```bash
git pull                                # complains about conflicts
# Same flow as merge above; or
git pull --rebase                       # cleaner if you have only local commits
```

### Resolution shortcuts
```bash
git checkout --ours <file>              # accept "current branch" version
git checkout --theirs <file>            # accept "incoming" version
# After --ours/--theirs you still need: git add + git merge --continue
```

## Bad State After Various Operations

### "I rebased and lost commits"
```bash
git reflog | head -20                   # find the pre-rebase HEAD
git reset --hard <pre-rebase-sha>
```

### "I `git pull` ed and got a mess"
```bash
git reflog                              # find HEAD before the pull
git reset --hard <sha>                  # back to that
# Then re-attempt: git fetch + git merge / rebase mannually
```

### "I `git checkout .` and lost uncommitted work"
- Working tree changes that were NEVER staged: gone, reflog can't help.
- Staged changes (was `git add`'d): `git fsck --lost-found` MIGHT find them as dangling blobs in `.git/lost-found/`. Tedious.
- **Lesson**: `git stash` before `git checkout .`, always.

### "I committed huge file / secret by mistake"
Not pushed yet:
```bash
git reset HEAD~1            # uncommit
git rm --cached <file>      # unstage
echo "<file>" >> .gitignore
git add .gitignore
git commit -m "remove huge file"
```

Already pushed (= secret leaked):
- **Rotate the secret immediately** — git history rewrites alone don't help; assume the secret is compromised.
- Then `git filter-branch` or `git-filter-repo` to scrub history.
- This is a security event, not just a git event.

## Stash Patterns

```bash
git stash                               # stash tracked changes
git stash -u                            # also include untracked files
git stash push -m "wip auth refactor"   # named stash

git stash list                          # see all stashes
git stash show -p stash@{1}             # diff of a specific stash
git stash apply stash@{1}               # apply, keep stash
git stash pop                           # apply newest, drop stash
git stash drop stash@{1}                # delete a specific stash
git stash clear                         # nuke all stashes (DESTRUCTIVE)
```

## Working with Tags

```bash
# Move a misplaced tag
git tag -d v1.2.3                       # delete locally
git push origin --delete v1.2.3         # delete remote
git tag v1.2.3 <correct-sha>            # create at right place
git push origin v1.2.3                  # republish
```

## Anti-Patterns / Don't

- **Don't `git push --force` on shared branches** — use `--force-with-lease` if you must rewrite, and coordinate with the team.
- **Don't `git reset --hard` without checking `git status` and `git reflog` first** — every reset is reversible only via reflog, and reflog isn't infinite.
- **Don't `git clean -fd`** when you have untracked work you care about — same loss profile as rm.
- **Don't `--amend` after push** unless it's your private branch and you know how to `--force-with-lease`.
- **Don't commit to fix a typo in the previous commit** when `git commit --amend` does it cleaner (assuming not pushed).
- **Don't combine `git pull` + `git rebase` and then panic** — the reflog has every state. Stop, breathe, find the pre-pull SHA, reset.

## When Stuck — Diagnostic Commands

```bash
git status                              # current state
git log --oneline -20                   # recent local commits
git log --all --oneline --graph -20     # all branches + merges visually
git reflog | head -30                   # HEAD movement history
git branch -av                          # all branches + remote tracking
git fsck --full --no-reflog             # check repo integrity (rare)
git diff @{u}..HEAD                     # what's local-only vs upstream
git diff HEAD..@{u}                     # what's upstream-only vs local
```
