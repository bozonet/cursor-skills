---
name: pr-review
description: Reviews pull requests focusing only on critical issues and must-fix items. Queries memory for context, handles large PRs efficiently with scoped reviews and filters. Use when reviewing pull requests, code changes, or when asked to review a PR.
---

# PR Review

## Core Principle

**Only report critical issues and must-fix items.** Skip suggestions, style preferences, and minor improvements. Focus on bugs, security vulnerabilities, breaking changes, and issues that will cause production problems.

## Before Starting: Sync Repo and Fetch PR Branch

**Do this first**, before reading files or running the review:

1. **Pull latest develop**, then **fetch and checkout the PR branch** so the review runs on up-to-date code:
   ```bash
   git fetch origin develop && git pull origin develop
   git fetch origin <branch-name> && git checkout <branch-name>
   ```
   Use the repo's default remote (usually `origin`). If the user specified another remote, use that.
2. **Identify the PR branch** from context (PR URL, branch name mentioned by the user, or current branch).
3. **If the branch cannot be found** (fetch fails, or `git rev-parse` doesn't find the branch):
   - **Ask the user**: "I couldn't find the PR branch. Please give me the branch name, or confirm it's pushed to the remote."
   - Do not proceed with the review until the branch is available or the user provides the correct branch name.

Only after the branch is checked out (or the user has confirmed which branch to use) should you proceed with the review.

## Linear Ticket Context (user-Linear MCP)

If the PR references a Linear ticket (e.g. branch name like `ENG-123`, or PR title/body with ticket ID), call **user-Linear** MCP **get_issue** (arg: `id`) to load the ticket before reviewing. Use the ticket title, description, and acceptance criteria to align the review with intended scope and edge cases.

## Review Strategy

### 1. Query Memory First

Before starting the review, query memory for:
- Related PRs or issues
- Recent changes to affected files
- Team conventions or patterns
- Known issues or technical debt in the area

### 2. Handle Large PRs Efficiently

For PRs with many files:
- **Scope the review**: Focus on changed files, not entire codebase
- **Use filters**: Review by file type or directory first
- **Batch processing**: Review related files together
- **Prioritize**: Start with critical paths (auth, data access, API endpoints)

### 3. Memory and Response Limits

- **Query memory strategically**: Only query for relevant context
- **Chunk large responses**: Break reviews into sections if needed
- **Use filters**: Filter file lists by pattern before reading
- **Summarize findings**: Group similar issues together

## Critical Checklist

Review these areas systematically:

### Security (MUST FIX)
- [ ] No hardcoded secrets, passwords, or API keys
- [ ] Authentication/authorization checks are present
- [ ] Input validation and sanitization for user data
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (proper escaping)
- [ ] Sensitive data not exposed in logs or responses

### Breaking Changes (MUST FIX)
- [ ] API contract changes documented
- [ ] Database migrations are backward compatible or have rollback plan
- [ ] Environment variable changes documented
- [ ] Breaking changes clearly marked in PR description

### Data Integrity (MUST FIX)
- [ ] Database transactions used for multi-step operations
- [ ] Proper error handling prevents partial updates
- [ ] Foreign key constraints respected
- [ ] Soft delete patterns followed (if applicable)

### Error Handling (MUST FIX)
- [ ] Unhandled exceptions caught appropriately
- [ ] Error messages don't expose sensitive information
- [ ] Failed operations don't leave system in inconsistent state
- [ ] Proper HTTP status codes used

### Performance (CRITICAL)
- [ ] N+1 query problems avoided
- [ ] Large data sets paginated
- [ ] No infinite loops or blocking operations
- [ ] Proper indexing for database queries

## Pattern-Based Checks

Use file and content patterns to identify areas needing review:

### File Pattern Checks

Apply checks based on file names:

```typescript
filePatterns: [
  { 
    pattern: /prisma|schema\.prisma/i, 
    item: "- [ ] Prisma schema changes have been validated and migrations created if needed" 
  },
  { 
    pattern: /\.tsx?$|\.jsx?$/i, 
    item: "- [ ] Component changes have been tested in different browsers/devices" 
  },
  {
    pattern: /api\/|server\/|route\./i,
    item: "- [ ] API changes have been tested and documented"
  },
  {
    pattern: /\.css$|\.scss$|tailwind/i,
    item: "- [ ] UI changes look good on all screen sizes"
  },
  {
    pattern: /\.test\.|\.spec\.|\.e2e/i,
    item: "- [ ] Tests are meaningful and cover critical paths"
  },
  {
    pattern: /\.env|config|secrets/i,
    item: "- [ ] No secrets committed, environment variables documented"
  }
]
```

### Content Pattern Checks

Scan file contents for critical patterns:

```typescript
contentPatterns: [
  {
    pattern: /useEffect|useState|useContext/i,
    item: "- [ ] React hooks usage has been reviewed for potential issues (dependencies, cleanup)"
  },
  {
    pattern: /fetch\(|axios\.|http\./i,
    item: "- [ ] API calls include proper error handling and timeout configuration"
  },
  {
    pattern: /password|token|secret|key|auth|jwt|session/i,
    item: "- [ ] Security-sensitive code has been reviewed for vulnerabilities"
  },
  {
    pattern: /prisma\.|\.create\(|\.update\(|\.delete\(/i,
    item: "- [ ] Database operations use transactions where needed and handle errors"
  },
  {
    pattern: /console\.(log|error|warn)/i,
    item: "- [ ] Debug statements removed or replaced with proper logging"
  },
  {
    pattern: /TODO|FIXME|XXX|HACK/i,
    item: "- [ ] Technical debt markers addressed or justified"
  },
  {
    pattern: /any\s|@ts-ignore|@ts-expect-error/i,
    item: "- [ ] Type safety maintained, type assertions justified"
  }
]
```

## Review Workflow

1. **Sync repo and fetch PR branch** (see "Before Starting: Sync Repo and Fetch PR Branch" above). If the branch isn't found, ask the user for the branch name and stop until they respond.

2. **Initial Assessment**
   - Read PR description and check for breaking changes
   - Identify affected areas (API, database, frontend, etc.)
   - Query memory for related context

3. **Scoped File Review**
   - Filter files by type/pattern
   - Review critical files first (auth, data access, API routes)
   - Use grep/search to find pattern matches

4. **Pattern Application**
   - Apply file pattern checks
   - Scan content for critical patterns
   - Verify checklist items

5. **Issue Reporting**
   - Group similar issues
   - Prioritize by severity (Critical > Must Fix > Important)
   - Provide specific file locations and line numbers
   - Suggest concrete fixes, not vague suggestions

## Response Format

Structure your review as:

```markdown
## 🔴 Critical Issues (Must Fix)

[Issue 1]
- **File**: `path/to/file.ts:123`
- **Issue**: [Specific problem]
- **Fix**: [Concrete solution]

## 🟡 Important Issues (Should Fix)

[Issue 2]
- **File**: `path/to/file.ts:456`
- **Issue**: [Specific problem]
- **Fix**: [Concrete solution]

## ✅ Pattern Checks

[Generated checklist based on file/content patterns found]
```

## Customization

The pattern lists above can be extended. Add new patterns to `filePatterns` or `contentPatterns` arrays as needed. Patterns are evaluated in order, so place more specific patterns first.

## Anti-Patterns to Avoid

- ❌ Don't comment on code style unless it causes bugs
- ❌ Don't suggest refactoring unless it fixes a critical issue
- ❌ Don't nitpick variable names or formatting
- ❌ Don't request changes for "best practices" that aren't critical
- ✅ Do focus on bugs, security, and production risks
- ✅ Do provide actionable fixes, not vague concerns
