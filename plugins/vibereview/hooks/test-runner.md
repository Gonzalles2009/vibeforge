---
event: "Stop"
---

# Test Runner Hook

When Vibereview completes a review cycle, automatically run tests to detect regressions.

## Instructions

Before finalizing the review:

1. **Detect test runner:**
   ```bash
   # Check package.json for test script
   # Or detect: jest, vitest, mocha, etc.
   ```

2. **Run tests:**
   ```bash
   npm test --passWithNoTests 2>&1 || true
   ```

3. **Run type check:**
   ```bash
   npx tsc --noEmit 2>&1 || true
   ```

4. **Run linter:**
   ```bash
   npx eslint . --max-warnings=0 2>&1 || npx biome check . 2>&1 || true
   ```

5. **Collect results** and add to final report:
   - Test failures → Regressions section
   - Type errors → Regressions section
   - New lint warnings → Uncertain changes section

## Regression Report Format

```markdown
## Automated Checks

### Tests
- **Status:** ✅ Passing | ⚠️ X failures
- **Failures:**
  - `test/user.spec.ts` — "should validate email"

### TypeScript
- **Status:** ✅ No errors | ⚠️ X errors
- **Errors:**
  - `src/api.ts:42` — Type 'string' is not assignable to 'number'

### Lint
- **Status:** ✅ Clean | ⚠️ X issues
- **New Issues:**
  - `src/utils.ts:15` — Unused variable 'temp'
```
