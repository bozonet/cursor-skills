---
name: release-from-linear
description: Creates a selective develop-to-master release from the current sprint's Ready to Deploy Linear tickets. Uses Linear MCP to list issues, cherry-picks corresponding PRs onto a release branch, and documents the release. Use when the user asks to create a release, release from Linear, or ready-to-deploy release.
---

# Release from Linear (Ready to Deploy)

Creates a **selective** release: only current sprint's **Ready to Deploy** tickets that have PRs in this repo. Not a full develop → master merge.

## Workflow

### 1. Get current sprint and Ready to Deploy issues

- **list_teams** → find the relevant team (e.g. Engineering), note its `id`.
- **list_cycles** with that `teamId`, `type: "current"` → get current cycle id.
- **list_issues** with `team`, `cycle`, `state: "Ready to Deploy"` (limit 100).

### 2. Filter to this repo only

Get the repo’s GitHub org/repo from `git remote get-url origin`. Include only issues whose **attachments** contain a PR URL for that repo: `https://github.com/<org>/<repo>/pull/<number>`. Extract PR number from each. Skip issues with no such attachment or with different status (e.g. QA Feedback).

### 3. Find merge commits on develop

```bash
git fetch origin develop master
git log origin/develop --oneline -150
```

Match lines like `abc123 Title (#1733)` to get commit hashes for each PR number. Order commits **oldest first** (by position in log) for cherry-pick.

### 4. Create release branch and cherry-pick

- Branch name: `release/<short-date>` (e.g. `release/feb-2`).
- Create from `origin/master`: `git checkout -b release/<date> origin/master`.
- Cherry-pick commits one by one (oldest first). Resolve conflicts; keep master’s deletions for removed files (e.g. coverage workflow). Use `git commit --no-verify` if pre-commit hooks block. Unstage and do not commit `.cursor/` or other local-only paths.
- If a commit touches files deleted on master (e.g. `.github/workflows/coverage.yml`): resolve by keeping them deleted (`git rm` those paths), then commit.

### 5. Document the release

Create **RELEASE_&lt;date&gt;.md** in repo root with:

- Title: `# Release, <Date>` (e.g. `Release, Feb 2`).
- Short note: selective release from current sprint Ready to Deploy only.
- **Table**: Linear id, Title, PR (#num).
- **Not included**: list any Ready to Deploy issues skipped (no PR link, wrong status).
- **Next steps**:
  1. Push: `git push -u origin release/<date>`
  2. Open PR: base **master**, head **release/&lt;date&gt;**
  3. PR title: **Release, &lt;Date&gt;**
  4. Do **not** push to master directly; master is protected and requires a PR and status checks.

### 6. Push and open PR (user)

Agent cannot push without credentials. Tell the user to run `git push -u origin release/<date>` and open the PR on GitHub (base: master, compare: release branch).

## Conflict resolution tips

- **modify/delete**: Keep master’s version (deleted) → `git rm <path>`, then continue.
- **Content conflicts**: Prefer the **incoming** (cherry-picked) change for feature code; only keep master when it’s a deliberate removal or branch-specific fix.
- **Missing context/variables**: If a cherry-pick references a variable or type the branch doesn’t have, add it to the relevant context, provider, or types so the code compiles and runs.

## Output summary for user

- List of included Linear issues and PR numbers.
- Release branch name and that it’s based on master.
- Reminder: push branch and merge via PR; do not push to master.
- Path to `RELEASE_<date>.md` for the PR description.
