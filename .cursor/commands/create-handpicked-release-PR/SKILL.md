---
name: create-handpicked-release-PR
description: Creates a master release from develop branch by selectively including only specified merged PRs. Use when creating a release, merging develop to master, or when the user wants to include only specific PRs in a release.
---

# Create Selective Release

Creates a master release from develop by including only the PRs you specify. This ensures releases contain exactly what you intend, not everything merged to develop.

## Workflow Overview

1. **Identify PRs**: User provides PR numbers/identifiers to include
2. **Create release branch**: Branch from master
3. **Selectively merge**: Cherry-pick or merge only specified PRs
4. **Verify**: Confirm only intended changes are included
5. **Create release**: Merge to master and tag

## Step 1: Gather PR Information

When the user shares PRs to include, extract:

- PR numbers (e.g., #123, #456)
- PR titles (for release notes)
- Merge commit SHAs (for cherry-picking)

**Example user input:**

```
Include these PRs in the release:
- #123: Add user authentication
- #456: Fix date formatting bug
- #789: Update API documentation
```

## Step 2: Create Release Branch

Create a release branch from the current master:

```bash
# Ensure you're on master and up to date
git checkout master
git pull origin master

# Create release branch
git checkout -b release/vX.Y.Z
```

Replace `vX.Y.Z` with your version number (e.g., `v1.2.3`).

## Step 3: Identify PR Commits

For each PR to include, find its merge commit:

```bash
# View recent merge commits on develop
git log develop --merges --oneline --grep="Merge pull request"

# Or find specific PR by number
git log develop --merges --oneline | grep "#123"

# Get full commit SHA
git log develop --merges --format="%H %s" | grep "#123"
```

**Alternative**: If PRs are identified by commit messages or SHAs, use those directly.

## Step 4: Cherry-pick PRs

Cherry-pick each PR's merge commit onto the release branch:

```bash
# Stay on release branch
git checkout release/vX.Y.Z

# Cherry-pick each PR (use merge commit SHA)
git cherry-pick <merge-commit-sha-1>
git cherry-pick <merge-commit-sha-2>
git cherry-pick <merge-commit-sha-3>

# If conflicts occur, resolve them:
# 1. Fix conflicts in affected files
# 2. git add <resolved-files>
# 3. git cherry-pick --continue
```

**Note**: Use merge commits, not individual feature commits, to preserve PR history.

## Step 5: Verify Release Contents

Confirm only intended PRs are included:

```bash
# Compare release branch to master
git log master..release/vX.Y.Z --oneline

# View files changed
git diff master...release/vX.Y.Z --stat

# Verify specific PRs are present
git log release/vX.Y.Z --grep="#123"
git log release/vX.Y.Z --grep="#456"
```

**Checklist**:

- [ ] All specified PRs are present
- [ ] No unexpected commits included
- [ ] Files changed match expectations

## Step 6: Generate Release Notes

Create release notes from included PRs:

```bash
# Extract PR titles and numbers
git log master..release/vX.Y.Z --merges --format="- %s" | grep -E "#[0-9]+"
```

Format as markdown:

```markdown
## Release vX.Y.Z

### Included PRs

- #123: Add user authentication
- #456: Fix date formatting bug
- #789: Update API documentation

### Changes

[List key changes or use git diff summary]
```

## Step 7: Create Release

Once verified, merge to master:

```bash
# Switch to master
git checkout master

# Merge release branch
git merge release/vX.Y.Z --no-ff -m "Release vX.Y.Z"

# Tag the release
git tag -a vX.Y.Z -m "Release vX.Y.Z: [brief description]"

# Push to remote
git push origin master
git push origin vX.Y.Z

# Optionally push release branch for reference
git push origin release/vX.Y.Z
```

## Alternative: Interactive Selection

If PRs aren't pre-identified, use interactive rebase to select commits:

```bash
# Create release branch from develop
git checkout -b release/vX.Y.Z develop

# Interactive rebase from master
git rebase -i master

# In editor: pick commits to include, drop others
# Save and close to apply
```

## Troubleshooting

### Conflict Resolution

If cherry-picking causes conflicts:

1. Review conflict markers
2. Resolve manually or use merge tools
3. `git add` resolved files
4. `git cherry-pick --continue`

### Missing Dependencies

If a PR depends on another not included:

- Either include the dependency PR
- Or manually apply required changes

### Verify Before Merging

Always verify release branch contents before merging to master:

```bash
git log master..release/vX.Y.Z
git diff master...release/vX.Y.Z
```

## Quick Reference

```bash
# Full workflow summary
git checkout master && git pull
git checkout -b release/vX.Y.Z
git cherry-pick <sha1> <sha2> <sha3>
git log master..release/vX.Y.Z  # Verify
git checkout master
git merge release/vX.Y.Z --no-ff -m "Release vX.Y.Z"
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin master && git push origin vX.Y.Z
```
