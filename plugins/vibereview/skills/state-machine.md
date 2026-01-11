---
description: "State machine definitions for Vibereview phases. Defines phases, valid transitions, validation requirements, and mode-specific behavior."
---

# State Machine Skill

This skill defines the state machine that governs Vibereview's execution flow.

## Phases

| Phase | ID | Description | Entry Condition |
|-------|-----|-------------|-----------------|
| Initialization | `INIT` | Target detection, stack scan, session creation | Start of review |
| Analysis | `ANALYSIS` | Run 5 review agents in waves | INIT completed |
| Snapshot | `SNAPSHOT` | Capture baseline behavior (thorough only) | ANALYSIS completed, mode=thorough |
| Fix | `FIX` | Apply proposed fixes via fixer agent | ANALYSIS/SNAPSHOT completed |
| Score | `SCORE` | Evaluate fix quality with specialists | FIX completed, mode!=quick |
| Regression | `REGRESSION` | Detect breaking changes | SCORE passed OR mode=quick |
| Report | `REPORT` | Generate final report, save session | REGRESSION completed |

## Mode-Specific Phase Matrix

```
Phase        │ quick │ standard │ thorough
─────────────┼───────┼──────────┼──────────
INIT         │  ✓    │    ✓     │    ✓
ANALYSIS     │  1x   │    1x    │    3x (ensemble)
SNAPSHOT     │  ✗    │    ✗     │    ✓
FIX          │  ✓    │    ✓     │    ✓
SCORE        │  ✗    │    ✓     │    ✓
REGRESSION   │ basic │   full   │ full + behavior
REPORT       │  ✓    │    ✓     │    ✓
```

### Mode Details

**quick** (~2-3 min):
- Single instance per agent
- No scoring loop (one-shot fixes)
- Basic regression: tsc + lint only
- Best for: small changes, quick checks

**standard** (~5-10 min):
- Single instance per agent
- Scoring loop until all >= 9 (max 5 cycles)
- Full regression: tsc + tests + lint
- Best for: regular PRs, feature branches

**thorough** (~15-20 min):
- 3 instances per agent (ensemble)
- Scoring loop until all >= 9 (max 5 cycles)
- Behavior snapshot before fixes
- Full regression + behavior comparison
- Best for: critical code, releases, legacy

## Transition Rules

```
INIT ─────────────────────────────────▶ ANALYSIS
                                              │
                                              ▼
                         ┌────── mode=thorough? ──────┐
                         │                            │
                        YES                          NO
                         │                            │
                         ▼                            │
                     SNAPSHOT                         │
                         │                            │
                         └──────────┬─────────────────┘
                                    │
                                    ▼
                                   FIX
                                    │
                                    ▼
                         ┌──── mode=quick? ────┐
                         │                     │
                        YES                   NO
                         │                     │
                         │                     ▼
                         │                  SCORE
                         │                     │
                         │      ┌──────────────┴──────────────┐
                         │      │                             │
                         │   any < 9                      all >= 9
                         │   AND cycles < 5               OR cycles >= 5
                         │      │                             │
                         │      ▼                             │
                         │   FIX (loop)                       │
                         │                                    │
                         └──────────────┬─────────────────────┘
                                        │
                                        ▼
                                   REGRESSION
                                        │
                                        ▼
                                     REPORT
```

## Phase Validation

Each phase must emit a completion signal before transition is allowed.

### INIT Completion
```yaml
required:
  - target_files: string[]      # List of files to review
  - target_pattern: string      # Original glob/path
  - stack:
      framework: string         # React, Vue, Angular, etc.
      typescript: boolean
      test_runner: string | null
  - session_id: string          # UUID for this session
  - mode: "quick" | "standard" | "thorough"
```

### ANALYSIS Completion
```yaml
required:
  - findings: Finding[]
    Finding:
      agent: string             # simplifier, deduplicator, etc.
      file: string
      line_start: number
      line_end: number
      issue: string
      fix_code: string
      confidence: number        # 1-10
      agreement: number         # 1-3 (ensemble only)
  - agents_run: string[]        # Which agents completed
  - instance_count: number      # 1 or 3
```

### SNAPSHOT Completion (thorough only)
```yaml
required:
  - baseline:
      functions: FunctionSignature[]
      exports: ExportMap[]
      api_contracts: APIContract[]
      component_props: PropsDefinition[]
  - captured_at: timestamp
```

### FIX Completion
```yaml
required:
  - changes: Change[]
    Change:
      file: string
      type: "create" | "modify" | "delete"
      finding_ids: string[]
      confidence: number
  - applied_count: number
  - skipped_count: number
  - conflict_resolutions: ConflictResolution[]
```

### SCORE Completion
```yaml
required:
  - scores:
      simplifier: number
      deduplicator: number
      decomposer: number
      readability: number
      consistency: number
  - remaining_issues: Issue[]   # If any score < 9
  - verdict: "pass" | "retry"
  - cycle: number               # Current cycle (1-5)
```

### REGRESSION Completion
```yaml
required:
  - checks:
      typescript: "pass" | "fail" | "skipped"
      tests: "pass" | "fail" | "skipped"
      lint: "pass" | "fail" | "skipped"
      behavior: "pass" | "fail" | "skipped"  # thorough only
  - regressions: Regression[]
    Regression:
      type: "type_error" | "test_failure" | "lint_error" | "behavior_change"
      file: string
      message: string
      severity: "error" | "warning"
  - safe_changes: string[]      # Files with no issues
```

### REPORT Completion
```yaml
required:
  - report_generated: boolean
  - session_saved: boolean
  - session_path: string        # .vibereview/sessions/...
  - summary:
      files_reviewed: number
      issues_found: number
      issues_fixed: number
      final_score: number
      cycles: number
```

## State Transitions in Code

The orchestrator should maintain state like this:

```typescript
interface SessionState {
  phase: Phase;
  mode: Mode;
  cycle: number;

  // Accumulated data
  target: TargetInfo;
  stack: StackInfo;
  findings: Finding[];
  baseline: Baseline | null;
  changes: Change[];
  scores: ScoreHistory;
  regressions: Regression[];
}

type Phase = 'INIT' | 'ANALYSIS' | 'SNAPSHOT' | 'FIX' | 'SCORE' | 'REGRESSION' | 'REPORT';
type Mode = 'quick' | 'standard' | 'thorough';
```

## Error Handling

### Phase Failure
If a phase fails:
1. Log the error with context
2. Attempt retry (max 2 retries per phase)
3. If still failing, transition to REPORT with partial results
4. Mark session as "incomplete" in saved JSON

### Agent Timeout
If an agent doesn't respond:
1. Wait up to 5 minutes
2. Mark agent findings as "timeout"
3. Continue with available findings
4. Note in final report

### Regression Found
If critical regression detected:
1. DO NOT proceed to REPORT automatically
2. Ask user: "Regression found. Rollback changes?"
3. If yes: revert changes, regenerate report
4. If no: proceed with warning in report

## Usage by Orchestrator

The orchestrator references this skill to:
1. Determine valid next phase
2. Validate phase completion before transition
3. Apply mode-specific logic
4. Handle errors appropriately

```markdown
When transitioning phases:
1. Check current phase completion signal
2. Validate all required fields present
3. Determine next phase based on mode and state
4. Update TodoWrite with new phase
5. Execute next phase
```
