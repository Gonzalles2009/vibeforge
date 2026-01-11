---
description: "Multi-agent iterative code review with auto-fix. Runs until all agents score >= 9/10."
arguments:
  - name: target
    description: "File, folder, or glob pattern to review (e.g., 'src/**/*.ts')"
    required: false
  - name: mode
    description: "Review mode: quick (~2-3 min), standard (~5-10 min), thorough (~15-20 min)"
    required: false
    default: "standard"
  - name: focus
    description: "Focus areas: all, simplify, dedupe, readability, decompose, consistency"
    required: false
    default: "all"
  - name: samples
    description: "Number of parallel instances per agent (1-5). Only applies to thorough mode."
    required: false
    default: "3"
  - name: strategy
    description: "Ensemble strategy: union (all findings), consensus (agreed only), weighted (prioritized). Only applies to thorough mode."
    required: false
    default: "union"
  - name: history
    description: "View past review sessions"
    type: boolean
    required: false
  - name: compare
    description: "Compare two session IDs (e.g., '2024-01-15T10-30-00 2024-01-16T14-22-15')"
    required: false
---

# Vibereview Command

Multi-agent iterative code review with automatic fixes.

## Modes

| Mode | Time | Agents | Scoring | Regression | Best For |
|------|------|--------|---------|------------|----------|
| **quick** | ~2-3 min | 5 × 1 | No loop | tsc + lint | Small changes |
| **standard** | ~5-10 min | 5 × 1 | Until ≥9 | + tests | Regular PRs |
| **thorough** | ~15-20 min | 5 × 3 | Until ≥9 | + behavior | Critical code |

## Special Modes

### History Mode

If `--history` flag is provided:

1. Read all sessions from `.vibereview/sessions/`
2. Display summary table:

```
┌────────────────────┬────────────────┬──────────┬───────┬──────────┐
│ Session ID         │ Target         │ Mode     │ Score │ Duration │
├────────────────────┼────────────────┼──────────┼───────┼──────────┤
│ 2024-01-16T14-22   │ src/**/*.tsx   │ standard │ 9.2   │ 7m 23s   │
│ 2024-01-15T10-30   │ src/api/       │ thorough │ 9.5   │ 18m 45s  │
│ 2024-01-14T09-15   │ src/hooks/     │ quick    │ 8.1   │ 2m 10s   │
└────────────────────┴────────────────┴──────────┴───────┴──────────┘
```

3. Allow user to select a session for detail view
4. Exit without running review

### Compare Mode

If `--compare` argument is provided with two session IDs:

1. Load both sessions from `.vibereview/sessions/`
2. Display comparison:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SESSION COMPARISON                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Session A: 2024-01-15T10-30-00                                 │
│  Session B: 2024-01-16T14-22-15                                 │
│                                                                  │
│  Scores:                                                        │
│  ┌─────────────┬─────────┬─────────┬────────┐                   │
│  │ Agent       │ A       │ B       │ Change │                   │
│  ├─────────────┼─────────┼─────────┼────────┤                   │
│  │ Simplifier  │ 7.5     │ 9.2     │ +1.7 ↑ │                   │
│  │ Deduplicator│ 8.0     │ 9.1     │ +1.1 ↑ │                   │
│  │ Decomposer  │ 6.5     │ 9.3     │ +2.8 ↑ │                   │
│  │ Readability │ 8.5     │ 9.0     │ +0.5 ↑ │                   │
│  │ Consistency │ 7.0     │ 9.4     │ +2.4 ↑ │                   │
│  └─────────────┴─────────┴─────────┴────────┘                   │
│                                                                  │
│  Issues: 47 → 12 (-35, 74% reduction)                           │
│  Fixes applied: 43 total                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

3. Exit without running review

## Normal Execution

### Step 1: Mode Selection

If `--mode` is not provided or invalid, ask user:

```
Choose review mode:

1. **quick** (~2-3 min)
   Fast check, no scoring loop, basic regression
   Best for: small changes, quick validation

2. **standard** (~5-10 min) [default]
   Full review with scoring loop until ≥9
   Best for: regular PRs, feature branches

3. **thorough** (~15-20 min)
   Ensemble (3 instances), behavior snapshot, full regression
   Best for: critical code, releases, legacy refactoring
```

### Step 2: Target Selection

If `--target` is not provided, ask user:

```
What should I review?

Suggestions:
- Recent changes: `git diff --name-only HEAD~5`
- Modified files: [list if available]
- Full directory: `src/`

Enter file, folder, or glob pattern:
```

### Step 3: Delegate to Orchestrator

Launch the orchestrator agent with collected parameters:

```markdown
Launch Task with:
- subagent_type: "vibereview:orchestrator"
- prompt: |
    Execute Vibereview with these parameters:

    Target: {target}
    Mode: {mode}
    Focus: {focus}
    Samples: {samples} (thorough only)
    Strategy: {strategy} (thorough only)

    Follow the state machine:
    INIT → ANALYSIS → [SNAPSHOT] → FIX → [SCORE] → REGRESSION → REPORT

    Refer to skills/state-machine.md for phase rules.
```

### Step 4: Relay Results

When orchestrator completes:
1. Display the final report
2. Confirm session was saved to `.vibereview/sessions/`
3. Provide summary statistics

## .vibereview/ Folder

The command ensures this structure exists:

```
.vibereview/
├── config.json           # User preferences
│   {
│     "defaultMode": "standard",
│     "scoreThreshold": 9,
│     "maxCycles": 5
│   }
│
├── .gitignore            # Ignore sessions, keep config
│   sessions/
│
├── sessions/             # Review history
│   └── {timestamp}.json
│
└── baselines/            # For thorough mode regression
    └── main.json
```

If `.vibereview/` doesn't exist, create it on first run.

## Backward Compatibility

These existing usage patterns continue to work:

```bash
# Default to standard mode
/vibereview src/

# Focus on specific aspects
/vibereview --focus=simplify src/

# Samples/strategy only apply to thorough mode
/vibereview --mode=thorough --samples=3 --strategy=consensus src/
```

## Quick Reference

```bash
# Quick check before commit
/vibereview src/ --mode=quick

# Full review for PR
/vibereview src/ --mode=standard

# Critical code review
/vibereview src/ --mode=thorough

# View past reviews
/vibereview --history

# Compare two reviews
/vibereview --compare "2024-01-15T10-30 2024-01-16T14-22"
```

## Available Agents

The orchestrator coordinates these agents:

**Analysis (ANALYZE/SCORE modes):**
- `vibereview:simplifier` — complexity reduction
- `vibereview:deduplicator` — duplication elimination
- `vibereview:decomposer` — monolith breaking
- `vibereview:readability` — naming/clarity
- `vibereview:consistency` — pattern unification

**Execution:**
- `vibereview:fixer` — applies fixes with voting

**Regression (thorough only):**
- `vibereview:snapshot` — captures baseline
- `vibereview:regression-behavior` — compares outputs
- `vibereview:regression-logic` — traces logic paths
- `vibereview:regression-contracts` — checks API contracts

**Orchestration:**
- `vibereview:orchestrator` — manages state machine
