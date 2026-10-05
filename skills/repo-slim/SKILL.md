---
name: repo-slim
description: Keep a git repo small. Stop committing build output (gitignore + untrack), and optionally purge old bulky files from history with git-filter-repo. Use when a push warns about large files, a repo's .git is huge, or the user asks to clean up old commits.
---

# Repo slim

Two jobs. Job 1 is safe and routine. Job 2 rewrites history and needs the user's yes every time.

## Job 1: stop committing build output (safe)

1. Find what is generated. Read the build script (e.g. `scripts/build-site.mjs`) for every path it writes. Anything it writes is build output. Hand-made files (templates, partials, data, scripts, source images) are source.
2. Measure: `git count-objects -vH`, and the biggest blobs:
   `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | awk '$1=="blob"' | sort -k3 -n | tail -20`
3. Add the output paths to `.gitignore` with a comment naming the script that makes them. Re-include sources with `!` (e.g. `site/*.html` then `!site/*.template.html`).
4. Untrack with `git rm -r --cached <paths>`. **Trap:** a git *pathspec* glob like `'site/*.html'` matches recursively. In one repo it caught `site/partials/*.html` too. A `.gitignore` `*` does not match `/`. Run `git status --short` and confirm no source file shows as `D` or `??` before committing.
5. Commit and push. The live deploy is unaffected: wrangler/hosting deploys from the local build, never from git.

## Job 2: remove old bulky files from history (destructive, ask first)

Job 1 only stops growth. Old copies stay in every past commit. Removing them rewrites every commit hash.

**Before running, tell the user plainly and wait for a yes:**
- Which paths get erased from all history, and the expected size saved.
- It force-pushes. Every other clone (other PC, other session, worktree) must re-clone or `git fetch && git reset --hard origin/<branch>`; uncommitted work there is at risk.
- Old commit hashes quoted in docs and handoff notes stop resolving.

Steps (on yes):
1. `pip install git-filter-repo` (not installed by default).
2. Back up: `git clone --mirror <repo-path> ../<name>-backup.git`
3. Save HEAD's file list: `git ls-files > /tmp/before.txt`
4. Erase exact paths only, using `--path` / `--path-regex` anchored to the directory (e.g. `--invert-paths --path site/dist/ --path-regex '^site/[^/]+\.html$'`, then re-check that templates were not matched: exclude them in the regex or verify). Run `git filter-repo --analyze` first to see what is big.
5. Compare `git ls-files` with `/tmp/before.txt`. Only the intended build files may be missing. If any source file is gone, restore from the mirror backup and stop.
6. filter-repo drops `origin`; re-add it, then `git push --force --all` and `git push --force --tags`.
7. Report `.git` size before and after. GitHub can keep old objects on its servers for a while; that is normal.

A leaked secret is not fixed by purging history: rotate the key regardless.
