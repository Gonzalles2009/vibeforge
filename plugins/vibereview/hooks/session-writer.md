---
event: "Stop"
---

# Session Writer Hook

This hook persists the Vibereview session data when a review completes.

## Trigger

Runs when the Vibereview session ends (Stop event).

## Purpose

1. Ensure `.vibereview/` folder structure exists
2. Write session JSON to `.vibereview/sessions/`
3. Update baseline (thorough mode on main branch)
4. Create `.gitignore` to exclude sessions from git

## Instructions

### Step 1: Check for Vibereview Context

Only run if this was a Vibereview session. Check for:
- Session state in conversation context
- Presence of review findings, scores, changes

If not a Vibereview session, exit silently.

### Step 2: Ensure Folder Structure

```bash
# Create .vibereview/ structure if not exists
mkdir -p .vibereview/sessions
mkdir -p .vibereview/baselines
```

### Step 3: Create Config if Missing

If `.vibereview/config.json` doesn't exist, create default:

```json
{
  "defaultMode": "standard",
  "scoreThreshold": 9,
  "maxCycles": 5,
  "ensemble": {
    "defaultSamples": 3,
    "defaultStrategy": "union"
  },
  "regression": {
    "runTests": true,
    "runTypecheck": true,
    "runLint": true
  }
}
```

### Step 4: Create .gitignore if Missing

If `.vibereview/.gitignore` doesn't exist:

```
# Ignore session history (contains timestamps, may be large)
sessions/

# Keep config and baselines
!config.json
!baselines/
```

### Step 5: Write Session JSON

Generate session filename from timestamp:
```
.vibereview/sessions/{YYYY-MM-DDTHH-mm-ss}.json
```

Session JSON structure:

```json
{
  "id": "2024-01-15T10-30-00",
  "version": "1.0.0",
  "target": "src/**/*.tsx",
  "mode": "standard",
  "focus": "all",

  "timestamps": {
    "started": "2024-01-15T10:30:00.000Z",
    "completed": "2024-01-15T10:45:23.456Z",
    "duration_ms": 923456
  },

  "stack": {
    "framework": "React",
    "typescript": true,
    "testRunner": "vitest",
    "linter": "eslint"
  },

  "phases": [
    {
      "name": "INIT",
      "status": "completed",
      "started": "2024-01-15T10:30:00.000Z",
      "completed": "2024-01-15T10:30:15.234Z",
      "duration_ms": 15234
    },
    {
      "name": "ANALYSIS",
      "status": "completed",
      "started": "2024-01-15T10:30:15.234Z",
      "completed": "2024-01-15T10:33:45.678Z",
      "duration_ms": 210444,
      "agents_run": ["simplifier", "deduplicator", "decomposer", "readability", "consistency"],
      "instance_count": 1
    }
  ],

  "findings": [
    {
      "id": "f-001",
      "agent": "simplifier",
      "file": "src/components/UserCard.tsx",
      "line_start": 45,
      "line_end": 52,
      "issue": "Nested ternary expression",
      "suggestion": "Use early return or mapping",
      "confidence": 9,
      "agreement": 1
    }
  ],

  "changes": [
    {
      "id": "c-001",
      "finding_id": "f-001",
      "file": "src/components/UserCard.tsx",
      "type": "modify",
      "applied": true,
      "confidence": 9
    }
  ],

  "scores": {
    "initial": {
      "simplifier": 6.5,
      "deduplicator": 7.0,
      "decomposer": 5.5,
      "readability": 7.5,
      "consistency": 6.0
    },
    "final": {
      "simplifier": 9.1,
      "deduplicator": 9.3,
      "decomposer": 9.2,
      "readability": 9.0,
      "consistency": 9.4
    }
  },

  "cycles": 2,

  "regressions": [],

  "summary": {
    "files_reviewed": 15,
    "issues_found": 47,
    "issues_fixed": 43,
    "issues_skipped": 4,
    "final_score": 9.2,
    "status": "completed"
  }
}
```

### Step 6: Update Baseline (Thorough Mode)

If mode was `thorough` AND on main/master branch:

```bash
# Check current branch
git branch --show-current
```

If on main or master, copy baseline:
```
.vibereview/baselines/main.json
```

Baseline contains snapshot data for future regression comparison.

### Step 7: Report to User

Output confirmation:

```
✓ Session saved: .vibereview/sessions/2024-01-15T10-30-00.json

View history: /vibereview --history
Compare reviews: /vibereview --compare "session1 session2"
```

## Error Handling

### Write Failure
If unable to write session file:
1. Log error to console
2. Display session summary in output (don't lose data)
3. Suggest manual save location

### Missing Data
If session data is incomplete:
1. Write what we have
2. Mark session as `"status": "incomplete"`
3. Note missing phases in session JSON

## Data Collection

The orchestrator agent maintains session state throughout the review.
This hook reads that state and persists it.

Expected state fields:
- `session.id` — unique identifier
- `session.mode` — quick/standard/thorough
- `session.target` — glob pattern
- `session.phases[]` — phase execution history
- `session.findings[]` — all issues found
- `session.changes[]` — all modifications made
- `session.scores` — initial and final scores
- `session.regressions[]` — detected issues
