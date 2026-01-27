# PR Review Reference Guide

## Memory Query Strategies

### When to Query Memory

Query memory **before** starting the review for:
- Recent changes to files being modified
- Related PRs that might conflict
- Team decisions about patterns or conventions
- Known issues in the codebase area

Query memory **during** review for:
- Similar patterns in other files
- Previous fixes for similar issues
- Context about why certain patterns exist

### Efficient Memory Queries

```typescript
// Good: Specific, scoped queries
"Recent changes to authentication middleware"
"PRs that modified user.service.ts"
"Team decisions about error handling patterns"

// Avoid: Too broad queries
"Everything about the codebase"
"All PRs"
```

## Handling Large PRs

### Strategy 1: File Type Filtering

Review by category:
1. **Critical paths first**: Auth, data access, API routes
2. **Frontend components**: UI changes
3. **Tests**: Verify test quality
4. **Config/migrations**: Schema and configuration

### Strategy 2: Directory-Based Scoping

```bash
# Review only API changes
grep -r "api/" --include="*.ts" changed-files.txt

# Review only frontend changes
grep -r "clients/" --include="*.tsx" changed-files.txt

# Review only backend services
grep -r "apps/" --include="*.ts" changed-files.txt
```

### Strategy 3: Incremental Review

For very large PRs:
1. Review first 20-30 files
2. Report critical issues found
3. Continue with next batch if needed
4. Summarize findings at end

## Pattern Matching Implementation

### Reading patterns.json

When reviewing, load patterns from `patterns.json`:

```typescript
// Pseudo-code for pattern matching
const patterns = require('./patterns.json');

for (const file of changedFiles) {
  // Check file patterns
  for (const filePattern of patterns.filePatterns) {
    if (new RegExp(filePattern.pattern, filePattern.flags).test(file.name)) {
      checklist.add(filePattern.item);
    }
  }
  
  // Check content patterns
  const content = readFile(file.path);
  for (const contentPattern of patterns.contentPatterns) {
    if (new RegExp(contentPattern.pattern, contentPattern.flags).test(content)) {
      checklist.add(contentPattern.item);
    }
  }
}
```

### Extending Patterns

To add new patterns, edit `patterns.json`:

```json
{
  "filePatterns": [
    {
      "pattern": "your-pattern-here",
      "flags": "i",
      "item": "- [ ] Your checklist item"
    }
  ],
  "contentPatterns": [
    {
      "pattern": "your-content-pattern",
      "flags": "i",
      "item": "- [ ] Your content checklist item"
    }
  ]
}
```

## Response Size Management

### Chunking Large Reviews

If review exceeds response limits:

1. **Split by severity**:
   - First response: Critical issues only
   - Follow-up: Important issues
   - Final: Pattern checklist

2. **Split by area**:
   - First: Security and auth
   - Second: Data access and API
   - Third: Frontend and UI

3. **Use summaries**:
   - List all files reviewed
   - Group similar issues
   - Provide file:line references

### Example Chunked Response

```markdown
## Review Summary

**Files reviewed**: 45 files across 8 directories
**Critical issues**: 3
**Important issues**: 7

## Critical Issues (Part 1/2)

[First 2 critical issues]

---

*Review continues in next message due to size limits*
```

## Critical Issue Examples

### Security Issues

**Hardcoded secrets:**
```typescript
// ❌ BAD
const apiKey = "sk_live_1234567890";

// ✅ GOOD
const apiKey = process.env.STRIPE_API_KEY;
```

**Missing auth check:**
```typescript
// ❌ BAD
@Get('/users/:id')
async getUser(@Param('id') id: string) {
  return this.userService.findOne(id);
}

// ✅ GOOD
@Get('/users/:id')
@UseGuards(AuthGuard)
async getUser(@Param('id') id: string, @CurrentUser() user: User) {
  return this.userService.findOne(id, user.organizationId);
}
```

### Breaking Changes

**API contract change:**
```typescript
// ❌ BAD - Breaking change without versioning
@Get('/users')
async getUsers() {
  return this.userService.findAll(); // Changed response format
}

// ✅ GOOD - Documented or versioned
// PR description: "BREAKING: User response now includes 'email' field"
// Or use API versioning: /v2/users
```

### Data Integrity

**Missing transaction:**
```typescript
// ❌ BAD
async createOrder(orderData: CreateOrderDto) {
  const order = await this.orderService.create(orderData);
  await this.inventoryService.decrement(order.itemId); // What if this fails?
  return order;
}

// ✅ GOOD
async createOrder(orderData: CreateOrderDto) {
  return this.prisma.$transaction(async (tx) => {
    const order = await tx.order.create({ data: orderData });
    await tx.inventory.update({
      where: { id: order.itemId },
      data: { quantity: { decrement: 1 } }
    });
    return order;
  });
}
```

## Common False Positives to Avoid

These patterns might trigger checks but are often fine:

- `console.log` in test files
- `TODO` comments with issue numbers
- `any` types in test mocks
- `@ts-expect-error` with explanatory comments
- Environment variables in example/config files (documented)

Use context to determine if these are actual issues.
