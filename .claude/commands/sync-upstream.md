---
description: Fetch upstream OpenCLAW changes, test in Docker, and push to a sync branch
argument-hint: (no arguments)
---

# Sync Upstream

---

## Your Mission

Fetch the latest changes from the upstream `openclaw/openclaw` repository, merge them into the local main branch, validate the build and tests inside a Docker container, and push the result to a dated branch on origin. Production must never be affected until the user explicitly rebuilds and redeploys.

---

## Phase 1: PREFLIGHT

Verify git state is clean and the upstream remote exists.

```bash
git status --short
git remote -v
```

- If working tree is dirty, stop and ask the user to commit or stash first.
- If the `upstream` remote does not exist, add it:

```bash
git remote add upstream https://github.com/openclaw/openclaw.git
```

Ensure you are on the `main` branch:

```bash
git checkout main
```

---

## Phase 2: FETCH & COMPARE

```bash
git fetch upstream
```

Show divergence between local and upstream using left-right notation (`<` = our commits, `>` = upstream commits we're missing):

```bash
git log --oneline --left-right main...upstream/main | head -30
```

Also get a summary count:

```bash
gh api "repos/TrueNorthTeamsAI/personal-2nd-brain-openclaw/compare/main...openclaw:openclaw:main" \
  --jq "{status: .status, ahead_by: .ahead_by, behind_by: .behind_by}"
```

If there are no new upstream commits, stop and report that the fork is already up to date.

Report the number of new commits and notable highlights (security fixes, breaking changes, version bumps).

---

## Phase 3: MERGE

```bash
git merge upstream/main --no-edit
```

- If merge conflicts occur, stop and report them to the user. Do NOT force-resolve.
- Use this table to guide conflict resolution when the user chooses to resolve:

| File pattern | Resolution guidance |
|---|---|
| `package.json` | Take upstream deps, keep local scripts if needed |
| `pnpm-lock.yaml` | Accept upstream version, regenerate with `pnpm install` |
| `*.patch` files | Usually take upstream version |
| Source files | Merge logic carefully, prefer upstream structure |

- If clean, continue.

---

## Phase 4: BUILD

Build a **test** Docker image (never overwrite the production `openclaw:local` tag):

```bash
docker build -t openclaw:test .
```

If the build fails, stop and report the error.

---

## Phase 5: TEST

Run the fast unit test suite inside the test image:

```bash
docker run --rm openclaw:test pnpm test:fast
```

Analyze the results:

1. Count total passed vs failed.
2. Classify each failure:
   - **Environment-specific** (missing Chrome extension, missing macOS Swift files, PTY spawn in container) — these are expected and not blockers.
   - **Code regression** — a test that passed before and now fails. This IS a blocker.
3. If any code regressions are found, stop and report them. Do NOT push.

---

## Phase 6: PUSH TO BRANCH

Create a dated branch and push:

```bash
git checkout -b chore/sync-upstream-$(date +%Y-%m-%d)
git push -u origin chore/sync-upstream-$(date +%Y-%m-%d)
```

Switch back to main:

```bash
git checkout main
```

---

## Phase 7: REPORT

```markdown
## Upstream Sync Report

**Date**: {date}
**Upstream commits merged**: {count}
**Version**: {version if bumped}

### Build
- Docker image: PASS/FAIL

### Tests
- **Passed**: {passed}/{total}
- **Failed**: {failed} ({classification: environment-only / regression})

### Failures (if any)
| Test | Reason | Blocker? |
|------|--------|----------|
| ... | ... | No/Yes |

### Branch
- Pushed to: `origin/chore/sync-upstream-{date}`
- Local main: ahead of origin/main by {n} commits
- **Production is unaffected** — running image `openclaw:local` was not modified

### Next Steps
- Review the branch, then merge to main: `git push origin main`
- When ready for production: rebuild `openclaw:local` and restart the container
```
