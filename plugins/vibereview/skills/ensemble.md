---
description: "Multi-sampling strategy for review agents. Runs multiple instances of each agent to improve coverage and confidence."
---

# Ensemble Review Strategy

Run multiple instances of each agent to maximize finding coverage and increase confidence.

## Why Ensemble?

LLMs are non-deterministic — different runs find different issues:

```
Run 1: Finds issues A, B, C
Run 2: Finds issues A, D, E
Run 3: Finds issues B, C, F

Union: A, B, C, D, E, F (nothing missed!)
Consensus: A, B, C (high confidence — found multiple times)
```

## Configuration

```yaml
ensemble:
  samples: 3              # Number of parallel instances per agent
  strategy: "union"       # union | consensus | weighted
  consensus_threshold: 2  # For consensus: minimum agreeing instances
  temperature_variance: true  # Use different temperatures per instance
```

## Strategies

### 1. Union Strategy (Default)

Combine ALL findings from all instances. Best for **not missing anything**.

```
Instance 1: [A, B, C]
Instance 2: [A, D]
Instance 3: [B, E, F]

Result: [A, B, C, D, E, F]
Confidence: A=2, B=2, C=1, D=1, E=1, F=1
```

**Use when:** You want comprehensive review, willing to review more findings

### 2. Consensus Strategy

Only keep findings that multiple instances agree on. Best for **high precision**.

```
Instance 1: [A, B, C]
Instance 2: [A, D]
Instance 3: [B, A, F]

Consensus (threshold=2): [A, B]
(A found 3 times, B found 2 times)
```

**Use when:** You only want high-confidence issues, less noise

### 3. Weighted Strategy

Assign confidence based on how many instances found the issue.

```
Instance 1: [A, B, C]
Instance 2: [A, D]
Instance 3: [B, A, F]

Result with weights:
- A: confidence × 1.5 (found by 3/3)
- B: confidence × 1.2 (found by 2/3)
- C, D, F: confidence × 1.0 (found by 1/3)
```

**Use when:** You want all findings but prioritized by agreement

## Temperature Variance

Run instances with different temperatures to encourage diversity:

```yaml
Instance 1: temperature = 0.3  # Conservative, obvious issues
Instance 2: temperature = 0.5  # Balanced
Instance 3: temperature = 0.7  # Creative, edge cases
```

This ensures instances don't all find the same obvious issues.

## Similarity Detection

To merge findings, detect when two instances found the "same" issue:

### Exact Match
Same file + same line range + same issue type

### Fuzzy Match
- Same file
- Overlapping line ranges (within 5 lines)
- Similar issue description (>80% semantic similarity)

### Example Merge

```yaml
Instance 1:
  file: UserCard.tsx
  lines: 45-52
  issue: "Nested ternary hard to read"
  fix: "Use early returns"

Instance 2:
  file: UserCard.tsx
  lines: 44-53
  issue: "Complex conditional logic"
  fix: "Extract to separate conditions"

# Detected as SAME issue (fuzzy match)
# Merged result:
  file: UserCard.tsx
  lines: 44-53
  issue: "Nested ternary / complex conditional"
  fixes:
    - option_a: "Use early returns" (instance 1)
    - option_b: "Extract to conditions" (instance 2)
  confidence: boosted (found by 2 instances)
```

## Parallel Execution

Launch all instances simultaneously:

```
┌────────────────────────────────────────────────────────┐
│                    WAVE 1                               │
│                                                         │
│  simplifier ──┬── instance 1 ──┐                       │
│               ├── instance 2 ──┼──► merge ──► findings │
│               └── instance 3 ──┘                       │
│                                                         │
│  deduplicator ─┬── instance 1 ──┐                      │
│                ├── instance 2 ──┼──► merge ──► findings│
│                └── instance 3 ──┘                      │
└────────────────────────────────────────────────────────┘
```

## Mode-Aware Configuration

Ensemble behavior changes based on review mode:

| Mode | Samples | Strategy | Temperature Variance | Rationale |
|------|---------|----------|---------------------|-----------|
| **quick** | 1 | N/A | false | Speed over coverage |
| **standard** | 1 | N/A | false | Balanced, single pass |
| **thorough** | 3 | configurable | true | Maximum coverage |

### Quick Mode

Single instance per agent. No ensemble overhead.

```yaml
# Effective configuration in quick mode
ensemble:
  enabled: false
  samples: 1
```

**Behavior:**
- Each agent runs once
- Findings used directly (no merge)
- Fastest execution (~2-3 min total)

### Standard Mode

Single instance per agent. Same as quick but with scoring loop.

```yaml
# Effective configuration in standard mode
ensemble:
  enabled: false
  samples: 1
```

**Behavior:**
- Each agent runs once per cycle
- Scoring loop iterates until all agents ≥9
- Moderate execution (~5-10 min total)

### Thorough Mode

Full ensemble with 3 instances per agent.

```yaml
# Effective configuration in thorough mode
ensemble:
  enabled: true
  samples: 3
  strategy: "union"  # or from config
  temperature_variance: true
```

**Behavior:**
- Each agent spawns 3 parallel instances
- Temperature: 0.3, 0.5, 0.7
- Findings merged using configured strategy
- Maximum coverage (~15-20 min total)

### Overriding Mode Defaults

User can override via `--samples` flag:

```bash
# Quick mode but with 2 samples
/vibereview src/ --mode=quick --samples=2

# Standard mode with ensemble
/vibereview src/ --mode=standard --samples=3
```

When `--samples` is specified explicitly, it overrides mode default.

## Cost Consideration

More samples = more API calls = higher cost.

| Samples | Coverage | Cost | Recommended For |
|---------|----------|------|-----------------|
| 1 | ~70% | 1x | Quick/standard modes |
| 2 | ~85% | 2x | Light ensemble |
| 3 | ~95% | 3x | Thorough mode (default) |
| 5 | ~99% | 5x | Critical systems |

## Output Format

After ensemble merge:

```yaml
Ensemble Results:
  strategy: union
  samples_per_agent: 3

  findings:
    - issue: "Nested ternary in UserCard.tsx:45"
      found_by: [instance_1, instance_2, instance_3]
      agreement: 3/3 (100%)
      confidence: 9.5 (boosted from 8.0)

    - issue: "Duplicate formatDate in utils"
      found_by: [instance_1, instance_3]
      agreement: 2/3 (67%)
      confidence: 8.2 (boosted from 7.5)

    - issue: "Unused import in api.ts"
      found_by: [instance_2]
      agreement: 1/3 (33%)
      confidence: 6.0 (original)

  stats:
    total_unique_findings: 15
    high_agreement (3/3): 4
    medium_agreement (2/3): 6
    single_instance: 5
```

## Integration with Voting

When fixer resolves conflicts, ensemble agreement adds weight:

```
Conflict: Inline vs Extract function

Votes:
- simplifier (3/3 agree on inline): +3 votes
- decomposer (2/3 agree on extract): +2 votes
- readability (1/3 found issue): +1 vote

Result: Inline wins (3 vs 2)
```
