# VibeForge Development Guide

## Overview

VibeForge is a Claude Code plugin marketplace. Currently contains one plugin: **vibereview**.

## Project Structure

```
vibeforge/
├── .claude-plugin/
│   └── marketplace.json       # Marketplace manifest (lists all plugins)
├── plugins/
│   └── vibereview/            # Multi-agent code review plugin
│       ├── .claude-plugin/
│       │   └── plugin.json    # Plugin manifest
│       ├── agents/            # 11 AI agents
│       ├── commands/          # Slash commands
│       ├── hooks/             # Event hooks
│       └── skills/            # Reusable knowledge
├── README.md
├── LICENSE
└── CLAUDE.md                  # You are here
```

## Vibereview Plugin

### Architecture

```
/vibereview src/ --mode=standard

INIT → ANALYSIS → FIX → SCORE ↺ → REGRESSION → REPORT
         │                │            │
    5 agents run    until all ≥9   tsc/tests/lint
```

### Agents (11 total)

| Agent | Purpose |
|-------|---------|
| **orchestrator** | Central coordinator, manages state machine |
| **simplifier** | Reduce complexity, flatten nested logic |
| **deduplicator** | Find and eliminate code duplication |
| **decomposer** | Break down large functions/components |
| **readability** | Improve naming, structure, clarity |
| **consistency** | Enforce patterns across codebase |
| **fixer** | Apply fixes from findings |
| **snapshot** | Capture baseline before fixes (thorough mode) |
| **regression-behavior** | Compare function I/O before/after |
| **regression-logic** | Verify business logic paths |
| **regression-contracts** | Check API/props backward compatibility |

### Commands

| Command | Description |
|---------|-------------|
| `/vibereview <target>` | Run code review |
| `/vibereview --mode=quick` | Fast review (~2-3 min) |
| `/vibereview --mode=standard` | Default with scoring loop (~5-10 min) |
| `/vibereview --mode=thorough` | Full ensemble + regression (~15-20 min) |
| `/vibereview --history` | View past review sessions |
| `/vibereview --compare id1 id2` | Compare two sessions |

### Skills

| Skill | Purpose |
|-------|---------|
| **state-machine** | Phase definitions and transitions |
| **ensemble** | Multi-sampling strategy (3x in thorough) |
| **scoring** | Agent scoring methodology (1-10 scale) |
| **stack-detection** | Detect framework/language/tools |

### Hooks

| Hook | Trigger | Purpose |
|------|---------|---------|
| **progress-tracker** | During review | Show progress updates |
| **test-runner** | After fixes | Run tests to verify |
| **session-writer** | On Stop | Persist session to .vibereview/ |

### Review Modes

| Mode | Agents | Samples | Scoring Loop | Regression |
|------|--------|---------|--------------|------------|
| **quick** | 5 | 1 | No | basic (tsc+lint) |
| **standard** | 5 | 1 | Yes (until ≥9) | full (+tests) |
| **thorough** | 5 | 3 | Yes (until ≥9) | full + behavior |

## Development Workflow

### Testing Locally

```bash
# From vibeforge directory
claude --plugin-dir ./plugins/vibereview

# Or add as local marketplace
/plugin marketplace add ./
/plugin install vibereview@vibeforge
```

### Validating Plugin

```bash
/plugin validate ./plugins/vibereview
```

### Making Changes

1. Edit files in `plugins/vibereview/`
2. Test with `/vibereview` command
3. Commit and push to GitHub
4. Create new release tag if needed

### Creating a Release

```bash
# Switch to correct account
gh auth switch --user Gonzalles2009

# Tag and push
git tag -a v0.2.0 -m "Description"
git push origin v0.2.0

# Create release
gh release create v0.2.0 --title "VibeForge v0.2.0" --generate-notes

# Switch back
gh auth switch --user alexkudrnd
```

## Adding New Plugins

### 1. Create Plugin Folder

```bash
mkdir -p plugins/new-plugin/.claude-plugin
```

### 2. Create plugin.json

```json
{
  "name": "new-plugin",
  "description": "What it does",
  "version": "0.1.0",
  "author": { "name": "Gonzalles2009" }
}
```

### 3. Add to marketplace.json

Edit `.claude-plugin/marketplace.json`, add to `plugins` array:

```json
{
  "name": "new-plugin",
  "source": "./plugins/new-plugin",
  "description": "What it does",
  "version": "0.1.0"
}
```

### 4. Create Components

```
plugins/new-plugin/
├── commands/       # /command-name
├── agents/         # Specialized AI agents
├── skills/         # Reusable knowledge
└── hooks/          # Event handlers
```

## File Conventions

### Agent Files (agents/*.md)

```markdown
---
description: "Short description for agent selection"
tools: ["Read", "Glob", "Grep", "Edit"]
---

# Agent Name

System prompt content here...
```

### Command Files (commands/*.md)

```markdown
---
description: "Shown in /help"
arguments:
  - name: target
    description: "What to review"
    required: true
---

# Command Name

Instructions for execution...
```

### Skill Files (skills/*.md)

```markdown
---
description: "When this skill applies"
---

# Skill Name

Knowledge and procedures...
```

### Hook Files (hooks/*.md)

```markdown
---
event: "PreToolUse" | "PostToolUse" | "Stop" | "SessionStart"
---

# Hook Name

What to do when triggered...
```

## Session Persistence

Vibereview stores sessions in `.vibereview/` folder:

```
.vibereview/
├── config.json          # User preferences
├── sessions/            # Review history (gitignored)
│   └── 2025-01-11T10-30-00.json
└── baselines/           # For regression detection
    └── main.json
```

## GitHub Repository

- **URL:** https://github.com/Gonzalles2009/vibeforge
- **Main branch:** main
- **Releases:** https://github.com/Gonzalles2009/vibeforge/releases

### Two GitHub Accounts

| Account | Purpose |
|---------|---------|
| `Gonzalles2009` | VibeForge owner, for pushing/releasing |
| `alexkudrnd` | Daily development |

Switch with: `gh auth switch --user <name>`

## Roadmap

### Planned Plugins

- **vibetest** — AI-powered test generation
- **vibedocs** — Automated documentation
- **vibeperf** — Performance analysis

### Vibereview Improvements

- [ ] Test on real projects
- [ ] Tune agent prompts based on feedback
- [ ] Add more focus areas (security, accessibility)
- [ ] Improve regression detection accuracy

## Quick Reference

```bash
# Run review
/vibereview src/

# Run quick review
/vibereview src/ --mode=quick

# Run thorough review with ensemble
/vibereview src/ --mode=thorough

# View history
/vibereview --history

# Test plugin locally
claude --plugin-dir ./plugins/vibereview

# Validate marketplace
/plugin validate .

# Push to GitHub (as Gonzalles2009)
gh auth switch --user Gonzalles2009
git add . && git commit -m "message" && git push
gh auth switch --user alexkudrnd
```
