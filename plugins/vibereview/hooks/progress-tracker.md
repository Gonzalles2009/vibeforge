---
event: "PostToolUse"
match_tool: ["Edit", "Write"]
---

# Progress Tracker Hook

When code is modified during a Vibereview session, track the change for the final report.

## Instructions

After each Edit or Write operation during review:

1. Note the file and line numbers modified
2. Categorize the change type:
   - `simplification` — code made simpler
   - `deduplication` — repeated code unified
   - `decomposition` — large unit split
   - `readability` — naming/structure improved
   - `consistency` — patterns unified

3. Assess confidence level (1-10)

4. Check for potential regressions:
   - Does this change affect exports?
   - Does it modify function signatures?
   - Could it break dependent code?

## Output

Add to internal tracking (no user output unless issue detected):

```yaml
Change:
  file: [path]
  lines: [start-end]
  type: [category]
  description: [brief]
  confidence: [1-10]
  regression_risk: [low|medium|high]
```

If regression risk is high, warn the user:

```markdown
⚠️ **High Regression Risk**
Change to `[file]:[lines]` may affect:
- [list of potential impacts]

Continue? (Changes can be reverted)
```
