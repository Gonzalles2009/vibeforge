# VibeForge

> A curated collection of powerful Claude Code plugins for developers who care about code quality.

## Installation

Add VibeForge to your Claude Code:

```bash
/plugin marketplace add Gonzalles2009/vibeforge
```

Then install any plugin:

```bash
/plugin install vibereview@vibeforge
```

---

## Plugins

### vibereview

**Multi-agent iterative code review with regression detection.**

5 specialized AI agents analyze your code from different angles, fix issues, and iterate until all agents score your code >= 9/10.

```bash
/vibereview src/           # Standard review
/vibereview src/ --mode=quick     # Fast check (~2-3 min)
/vibereview src/ --mode=thorough  # Full analysis with regression detection (~15-20 min)
```

**Features:**
- 5 specialized agents: simplifier, deduplicator, decomposer, readability, consistency
- Iterative scoring loop until quality threshold met
- Ensemble analysis (3x sampling in thorough mode)
- Regression detection: behavior, logic, contracts
- Session persistence with history

**Agents:**
| Agent | Focus |
|-------|-------|
| Simplifier | Reduce complexity, remove nested logic |
| Deduplicator | Find and eliminate code duplication |
| Decomposer | Break down large functions/components |
| Readability | Improve naming, structure, clarity |
| Consistency | Enforce patterns across codebase |

**Modes:**
| Mode | Time | Features |
|------|------|----------|
| `quick` | ~2-3 min | Single pass, basic checks |
| `standard` | ~5-10 min | Scoring loop until >= 9 |
| `thorough` | ~15-20 min | Ensemble + regression detection |

---

## Coming Soon

More plugins are in development:

- **vibetest** — AI-powered test generation
- **vibedocs** — Automated documentation
- **vibeperf** — Performance analysis

---

## For Developers

Want to contribute or create your own plugin?

```bash
# Clone the repo
git clone https://github.com/Gonzalles2009/vibeforge.git

# Test locally
claude --plugin-dir ./vibeforge/plugins/vibereview

# Validate marketplace
/plugin validate ./vibeforge
```

### Plugin Structure

```
plugins/your-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
├── agents/
├── skills/
└── hooks/
```

---

## License

MIT License - see [LICENSE](LICENSE) for details.

---

**Made with vibes by [Gonzalles2009](https://github.com/Gonzalles2009)**
