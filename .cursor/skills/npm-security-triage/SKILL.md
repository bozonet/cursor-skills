---
name: npm-security-triage
description: Monthly npm audit triage for monorepos without Dependabot — bucket critical patches, open thin safe PRs, and optionally create a short Linear In Review ticket. Use when the user asks for dependency triage, npm audit, security package updates, or safe backend dependency patches.
---

# npm Security Triage

Monthly dependency security triage for monorepos **without Dependabot/Renovate**. Goal: surface actionable high/critical patches, ship thin safe PRs, optionally file a short Linear ticket.

> **Axios / shared HTTP clients:** Before bumping `axios` (or similar HTTP clients) across package trees, check the repo’s PR history for prior upgrade regressions. Do **not** batch frontend and backend axios bumps in one PR without that check. Same-major security patches within a single tree are usually fine; park or split anything that crosses trees or majors.

## When to use

- User asks for npm audit, dependency triage, security package updates, or safe backend patches
- Recurring monthly hygiene (optional alternative to Dependabot/Renovate)

## Out of scope

- Enabling Dependabot/Renovate by default (offer only if the user asks)
- Nest / Prisma / React / Nx **major** upgrade playbooks
- One-off triage scripts unless parsing becomes painful later

## Workflow

### 1. Triage (30–60 min)

Run production audits on **each** package tree (root + nested, e.g. `clients/`):

```bash
npm audit --omit=dev
# nested trees:
(cd clients && npm audit --omit=dev)
```

Optional — only for critical-path **direct** deps (auth, HTTP, DB, uploads):

```bash
npm outdated
(cd clients && npm outdated)
```

**Focus:**

- Prefer **direct** high/critical findings
- Ignore most transitive noise unless a patch is one hop away and same-major
- Note each finding: package, current → fixed, severity, direct vs transitive, which tree

### 2. Bucket results

Present a short table to the user:

| Package | Tree | Current → Fixed | Severity | Bucket | Why |
| --- | --- | --- | --- | --- | --- |

**Do now**

- Same-major / non-breaking security patches
- Direct high/critical with a clear patch version
- Same-tree axios (or similar) security patches when PR history has no known regression for that tree

**Park**

- Majors: Nest, Prisma, React, Nx, and other framework leaps
- Packages with known upgrade regressions in this repo’s PR history
- Cross-tree axios/HTTP client bumps without a prior history check
- Transitive-only noise with no safe direct upgrade path
- Anything needing a coordinated major + migration

Do **not** open PRs for Park items unless the user explicitly asks.

### 3. Ship thin PRs (when asked)

One theme per PR. Examples of good themes: “backend axios patch”, “multer/upload stack”, “root qs/follow-redirects”. Bad: mixing clients lockfile + backend axios + Nest bump.

```bash
git fetch origin develop
git checkout -B chore/npm-security-<short-theme> origin/develop
```

(Use the repo’s default integration branch if it isn’t `develop`.)

Update only the targeted packages (prefer exact patched versions from audit):

```bash
npm install <pkg>@<version>   # or -w / nested cd as needed
# nested tree separately when that is the theme:
(cd clients && npm install <pkg>@<version>)
```

Validate before opening a PR:

```bash
npm run typecheck
# if nested clients (or equivalent) touched:
(cd clients && npm run typecheck)   # or repo-equivalent check
```

**PR rules**

- Branch from the repo’s default integration branch (often `origin/develop`)
- Title: `chore(deps): <theme> security patches` (or similar)
- Body: list packages + versions, audit severities addressed, what was parked and why
- Manual QA checklist in the PR (per package / surface touched) — auth login, a representative API call, uploads if upload stack, etc.
- **Never** combine frontend and backend axios (or other shared HTTP clients) in one PR without checking upgrade history first

Push and create PR only when the user asks (follow repo PR / commit user rules).

### 4. Linear ticket (only when asked)

Short plain-language ticket for the batch that shipped (or is about to).

1. Resolve Linear MCP server id (`user-Linear` first, then `Linear` — same discovery pattern as `release-from-linear`).
2. **list_teams** → use the user’s Linear team (e.g. Engineering if that is their team).
3. **list_cycles** `type: "current"` for that team.
4. **list_issue_statuses** / create with state **In Review**.
5. Create issue:
   - **Assignee:** me
   - **Cycle:** current
   - **State:** In Review
   - **Team:** user’s Linear team
   - **Title:** e.g. `npm security: <theme> patches`
   - **Description:** 3–6 bullets — what upgraded, what parked, link to PR(s)
6. Attach the PR URL to the issue (attachment or description link).

If Linear MCP is unavailable, give the user the title/body/PR link to paste manually.

## Monorepo notes

- **No Dependabot/Renovate by default** — prefer this monthly triage unless the user wants automation.
- **Package trees:** audit every lockfile (e.g. root backends and nested `clients/` frontends) separately.
- **Axios / HTTP clients:** check PR history before bumping across trees; don’t batch frontend + backend blindly.
- Prefer thin, reviewable PRs over a monthly kitchen-sink bump.

## Output summary for user

After triage (and after any PR / Linear step):

1. **Do now** list (package, tree, version bump, severity)
2. **Park** list with one-line reason each
3. If PR opened: branch name, PR URL, QA checklist reminder
4. If Linear created: issue id/URL, cycle, In Review confirmation
5. Explicit callout if axios (or similar) appeared in audit results and was parked pending history check
