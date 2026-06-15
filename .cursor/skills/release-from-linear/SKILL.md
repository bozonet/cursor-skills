---
name: release-from-linear
description: Creates a selective develop-to-master release from the current sprint's Ready to Deploy Linear tickets. Finds Linear MCP (e.g. user-Linear) when available; uses it to list issues, then cherry-picks corresponding PRs onto a release branch and documents the release. Use when the user asks to create a release, release from Linear, or ready-to-deploy release.
---

# Release from Linear (Ready to Deploy)

Creates a **selective** release: only current sprint's **Ready to Deploy** tickets that have PRs in this repo. Not a full develop → master merge.

> **Conflict rule (read first):** Cherry-picks are point-in-time snapshots; `origin/develop` is the newest integrated truth for files touched by multiple release PRs. Never use merge strategies (`-X ours` / `-X theirs`) or `git add -A` during conflicts. Resolve per-file, then sync conflicted/multi-PR files from develop, then run `npm run typecheck` before pushing.

## Workflow

### 0. Find Linear MCP

Linear may be enabled at **user level** (all projects) or **project level**. The agent only sees MCP servers that Cursor has loaded for the current workspace.

- **Try common identifiers**: call `list_teams` with server **`user-Linear`** first (Cursor often exposes Linear under this id). If that fails, try **`Linear`**. Use whichever succeeds for all Linear tools.
- **If you get "MCP server does not exist"**: the error lists "Available servers: ...". Use the Linear-related name shown there (e.g. `user-Linear`, `Linear`).
- **Discover from disk (for server id only)**: under `~/.cursor/projects/` (or the workspace’s `.cursor`), search for a folder or `SERVER_METADATA.json` that mentions "Linear", or for tool files `list_teams.json` / `list_cycles.json` / `list_issues.json`. The parent folder name (e.g. `user-Linear`) or `serverIdentifier` in that project’s `mcps/<name>/SERVER_METADATA.json` is the server to use. Only call the tool if that server appears in "Available servers"; otherwise Cursor has not loaded it for this project.
- **If Linear is not available**: tell the user Linear MCP is not loaded for this workspace. They can (a) add a project-level config so this repo gets Linear: create `.cursor/mcp.json` in the repo with `{"mcpServers":{"Linear":{"url":"https://mcp.linear.app/mcp","headers":{}}}}`, then reload the project; or (b) enable Linear in Cursor Settings → MCP (global or for this folder); or (c) provide the list of PR numbers so the release can be built without Linear.

Required Linear tools: `list_teams`, `list_cycles`, `list_issues`. Optional: `get_issue` or issue details for attachments (PR links).

### 1. Get current sprint and Ready to Deploy issues

- **list_teams** (server: e.g. `user-Linear`) → find the relevant team (e.g. Engineering), note its `id`.
- **list_cycles** with that `teamId`, `type: "current"` → get current cycle id.
- **list_issues** with `team`, `cycle`, `state: "Ready to Deploy"` (limit 100).

### 2. Filter to this repo only

Get the repo's GitHub org/repo from `git remote get-url origin`. Include only issues whose **attachments** contain a PR URL for that repo: `https://github.com/<org>/<repo>/pull/<number>`. Extract PR number from each. Skip issues with no such attachment or with different status (e.g. QA Feedback).

### 3. Find merge commits on develop

```bash
git fetch origin develop master
git log origin/develop --oneline -150
```

Match lines like `abc123 Title (#1733)` to get commit hashes for each PR number. Order commits **oldest first** (by position in log) for cherry-pick.

### 4. Create release branch and cherry-pick

- Branch name: `release/<short-date>` (e.g. `release/feb-2`).
- Create from `origin/master`: `git checkout -B release/<date> origin/master`.
- Cherry-pick commits one by one (oldest first). Use `git commit --no-verify` if pre-commit hooks block. Do not commit `RELEASE_*.md`, `.cursor/`, or other local-only paths.

#### On conflict (per file — no auto-merge)

1. List conflicts: `git diff --name-only --diff-filter=U`
2. Take the **incoming PR commit's** version (not git's half-merged hunks): `git show <commit>:<path> > <path>` then `git add <path>`
3. modify/delete on master → keep deleted: `git rm <path>`
4. Finish: `git commit --no-verify -F .git/MERGE_MSG` (or `git cherry-pick --skip` if the pick is now empty)

**FORBIDDEN:** `-X theirs`, `-X ours`, `git add -A` during conflicts, leaving `<<<<<<<` markers.

#### After ALL picks — develop sync (required)

Cherry-picks are point-in-time; `origin/develop` is the newest integrated truth for any file touched by more than one release PR (or by a much older pick like a multi-month-old feature).

1. Collect files changed by included commits:
   `for c in <commit-list>; do git diff-tree --no-commit-id --name-only -r $c; done | sort -u`
2. For each file that **conflicted** OR was **modified by 2+ included commits**, sync to develop:
   `git checkout origin/develop -- <path>`
3. Verify it now matches develop (should be empty): `git diff origin/develop -- <path>`
4. Keep a master-only version only when it is a **deliberate** master deletion or a hotfix already on master (see edge cases below).

#### Validate before push (do NOT push if any fail)

- `npm run typecheck`
- No real conflict markers: `rg '<<<<<<< HEAD|>>>>>>> '` returns nothing
- No master code silently vanished: spot-check `git diff origin/master -- <high-conflict files>` and confirm only intended changes

#### Edge cases

- **Stacked PRs:** if a Linear ticket's PR was merged into another PR's branch (not directly into develop), include it via the **parent PR's develop merge commit**, not the stacked PR's standalone commit. Note this in the RELEASE doc.
- **Master hotfixes already shipped:** if a ticket's fix already landed on master (e.g. a hotfix PR targeting master), it is already on the release base — **skip the duplicate cherry-pick** and mark "already on master" in the RELEASE doc.

### 5. Document the release

Create **RELEASE_&lt;date&gt;.md** in repo root with (do **not** commit it—leave as a local file only for the user to copy into the PR description):

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

## Conflict resolution (decision table)

| Situation | Resolution |
| --- | --- |
| Code from the PR being picked | Take that commit's version: `git show <commit>:path` |
| File touched by multiple release PRs | After all picks: `git checkout origin/develop -- path` (develop = newest truth) |
| modify/delete on master | Keep deleted: `git rm <path>` |
| Old pick conflicts with newer state | **develop wins** — never resolve to a stale old-commit version |
| Hotfix already on master | Keep master, skip the duplicate pick |
| Missing context/variables after pick | Add the referenced var/type/provider so it compiles; re-run `npm run typecheck` |

**Never** use `-X ours` / `-X theirs` or `git add -A` to resolve conflicts — both silently discard the wrong side. Resolve per-file, then develop-sync, then typecheck.

### High-conflict files in CareSuite (always develop-sync after picks)

- `apps/care/src/core/case/export-job.service.ts`
- `apps/care/src/core/case/case.module.ts`
- `apps/provider-portal/src/core/annual-update/*.ts`
- `clients/apps/provider-portal/src/layouts/SectionRenderer.tsx`
- `clients/apps/provider-portal/src/pages/AnnualUpdateReview.tsx`
- `clients/apps/provider-portal/src/state/attribute.tsx`

## Output summary for user

- List of included Linear issues and PR numbers.
- Release branch name and that it's based on master.
- Reminder: push branch and merge via PR; do not push to master.
- Path to `RELEASE_<date>.md` for the PR description (file is intentionally uncommitted).
