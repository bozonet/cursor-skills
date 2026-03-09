---
name: release-from-linear
description: Creates a selective develop-to-master release from the current sprint's Ready to Deploy Linear tickets. Finds Linear MCP (e.g. user-Linear) when available; uses it to list issues, then cherry-picks corresponding PRs onto a release branch and documents the release. Use when the user asks to create a release, release from Linear, or ready-to-deploy release.
---

# Release from Linear (Ready to Deploy)

Creates a **selective** release: only current sprint's **Ready to Deploy** tickets that have PRs in this repo. Not a full develop → master merge.

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
- Create from `origin/master`: `git checkout -b release/<date> origin/master`.
- Cherry-pick commits one by one (oldest first). Resolve conflicts; keep master's deletions for removed files (e.g. coverage workflow). Use `git commit --no-verify` if pre-commit hooks block. Unstage and do not commit `.cursor/` or other local-only paths.
- If a commit touches files deleted on master (e.g. `.github/workflows/coverage.yml`): resolve by keeping them deleted (`git rm` those paths), then commit.

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

## Conflict resolution tips

- **modify/delete**: Keep master's version (deleted) → `git rm <path>`, then continue.
- **Content conflicts**: Prefer the **incoming** (cherry-picked) change for feature code; only keep master when it's a deliberate removal or branch-specific fix.
- **Missing context/variables**: If a cherry-pick references a variable or type the branch doesn't have, add it to the relevant context, provider, or types so the code compiles and runs.

## Output summary for user

- List of included Linear issues and PR numbers.
- Release branch name and that it's based on master.
- Reminder: push branch and merge via PR; do not push to master.
- Path to `RELEASE_<date>.md` for the PR description (file is intentionally uncommitted).
