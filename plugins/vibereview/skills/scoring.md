---
description: "Scoring system and report generation for Vibereview. Aggregates agent scores, manages voting, detects regressions, generates final reports."
---

# Vibereview Scoring System

This skill provides the scoring framework for multi-agent code review.

## Score Aggregation

When collecting scores from agents, use this format:

```yaml
Agent Scores:
  simplifier: 8.5
  deduplicator: 9.0
  decomposer: 7.5
  readability: 9.2
  consistency: 8.0

Aggregate:
  average: 8.44
  minimum: 7.5 (decomposer)
  threshold_met: false (min < 9)
```

## Voting System

When agents propose conflicting changes to the same code:

### Voting Rules

1. **Simple Majority** — 3+ agents agree → change applies
2. **Tie (2-2-1)** — defer to the agent with highest domain relevance:
   - Naming conflict → `readability` wins
   - Structure conflict → `decomposer` wins
   - Pattern conflict → `consistency` wins
3. **50/50 Split** — ask the user with both options presented

### Conflict Detection

```markdown
## Conflict Detected

**Location:** `src/components/UserCard.tsx:45-62`

**Option A (simplifier, deduplicator):**
Inline the `formatUserName` function

**Option B (readability, decomposer):**
Extract to `utils/formatters.ts`

**Votes:** 2-2 (tie)
**Resolution:** Ask user preference
```

## Regression Detection

After changes, run checks and categorize:

### Check Commands

```bash
# TypeScript
tsc --noEmit

# Tests (detect test runner)
npm test || yarn test || pnpm test

# Lint (detect linter)
npx eslint . || npx biome check .
```

### Categorization

```markdown
## Regression Analysis

### ✅ Safe Changes (High Confidence)
Changes with no detected issues:
- Renamed `usr` → `user` in 5 files
- Extracted `calculateTotal` to utils
- Removed unused imports

### ⚠️ Uncertain Changes (Needs Review)
Changes where agents disagreed or logic changed:
- Simplified conditional in `auth.ts:42` — verify edge cases
- Combined two useEffect hooks — verify timing

### 🚨 Detected Regressions
Breaking changes found:
- Type error in `UserService.ts:78`
- Test failure: `user.spec.ts` — "should handle null user"
- Lint error: unused variable introduced
```

## Final Report Template

Generate this report after review completion:

```markdown
# 🔍 Vibereview Report

**Generated:** [timestamp]
**Target:** [files/folders reviewed]
**Cycles:** [N] (threshold: all scores ≥ 9)

---

## 📊 Final Scores

| Agent | Initial | Final | Δ |
|-------|---------|-------|---|
| Simplifier | 6.5 | 9.1 | +2.6 |
| Deduplicator | 7.0 | 9.3 | +2.3 |
| Decomposer | 5.5 | 9.0 | +3.5 |
| Readability | 8.0 | 9.5 | +1.5 |
| Consistency | 7.5 | 9.2 | +1.7 |

**Overall: 9.22/10** ✅

---

## 📁 Files Modified

| File | Changes | Impact |
|------|---------|--------|
| `src/components/UserCard.tsx` | 3 | High |
| `src/utils/formatters.ts` | 1 (new) | Medium |
| `src/hooks/useUser.ts` | 2 | Medium |

---

## ✅ Applied Changes (High Confidence)

### Simplification
- **UserCard.tsx:45** — Simplified nested ternary to early returns
- **api.ts:112** — Replaced callback chain with async/await

### Deduplication
- **Created `utils/formatters.ts`** — Unified 4 duplicate formatDate implementations
- **UserList.tsx, ProductList.tsx** — Extracted shared `ListItem` component

### Decomposition
- **Dashboard.tsx** — Split 400-line component into 5 focused components
- **orderService.ts** — Extracted payment logic to `paymentService.ts`

### Readability
- **Multiple files** — Renamed `d` → `date`, `u` → `user`, etc.
- **auth.ts** — Added named constant for magic number `86400000` → `ONE_DAY_MS`

### Consistency
- **Standardized imports** — All now use `@/` alias format
- **Error handling** — Unified to throw pattern across services

---

## ⚠️ Uncertain Changes (Please Review)

These changes may need human verification:

1. **auth.ts:67-72** — Simplified auth check logic
   - *Concern:* Edge case for expired tokens unclear
   - *Agents disagreed:* simplifier (apply) vs consistency (keep)

2. **useCart.ts:34** — Combined two useEffects
   - *Concern:* Timing of cart sync may have changed
   - *Suggestion:* Test checkout flow manually

---

## 🚨 Regressions Detected

**Action Required:**

1. **Type Error** — `UserService.ts:78`
   ```
   Type 'null' is not assignable to type 'User'
   ```
   *Caused by:* Simplification of null check
   *Fix:* Add null guard or update return type

2. **Test Failure** — `user.spec.ts:42`
   ```
   Expected: "Anonymous"
   Received: undefined
   ```
   *Caused by:* Changed default value logic
   *Fix:* Update test expectation or restore default

---

## 💡 Recommendations (Not Applied)

Additional improvements identified but not auto-applied:

1. **Consider React Query** for data fetching (manual migration)
2. **Add error boundary** around Dashboard components
3. **Extract hardcoded API URLs** to environment config

---

## 📈 Metrics

- **Lines reviewed:** 2,847
- **Lines changed:** 342 (12%)
- **Issues found:** 47
- **Issues fixed:** 42
- **Issues deferred:** 5

---

*Generated by Vibereview v1.0.0*
```

## Progressive Disclosure

For large reports, use progressive disclosure:

1. **Summary first** — scores and high-level stats
2. **Expandable sections** — details on demand
3. **Critical items first** — regressions before improvements

## Confidence Levels

Assign confidence to each change:

| Confidence | When to Use | Action |
|------------|-------------|--------|
| **High (9-10)** | All agents agree, no logic change, tests pass | Auto-apply |
| **Medium (6-8)** | Most agents agree, minor logic change | Apply with note |
| **Low (3-5)** | Agents disagree, logic changes | Defer to user |
| **Very Low (1-2)** | Breaking change detected | Do not apply |
