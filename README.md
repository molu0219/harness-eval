# harness-eval

A Claude Code skill for evaluating AI development systems — scores design quality across 6 dimensions regardless of implementation.

Works with: Claude Code harness, Cursor rules, Windsurf, Aider, custom CLAUDE.md setups, or any AI-assisted development configuration.

## Install

```bash
# Add to your Claude Code skills directory
mkdir -p ~/.claude/skills/harness-eval
cp SKILL.md ~/.claude/skills/harness-eval/
```

Or with the Claude Code skill marketplace, install directly from this repo.

## Usage

```
/harness-eval              # evaluate current project's AI dev system
/harness-eval <path>       # evaluate a specific system at given path
/harness-eval compare <p1> <p2>  # compare two systems
```

## What it evaluates

| Dimension | Question |
|-----------|----------|
| **Endurance** | How long can the AI work effectively before degrading? |
| **Context Efficiency** | Is what's in context actually needed right now? |
| **Overhead** | How much does the system cost vs. what it delivers? |
| **Adherence** | Can the rules actually be enforced? |
| **Robustness** | Does the system break gracefully or catastrophically? |
| **Collaboration** | Is the human↔AI interaction well designed? |

## How it works

1. **Discover** — reads all AI instruction files, hooks, state files, and config
2. **Simulate** — traces the full development lifecycle, token by token
3. **Stress test** — challenges every component's assumptions against current model capabilities
4. **Score** — 6 dimensions → letter grade → weighted overall score
5. **Recommend** — max 5 actionable fixes, prioritized by impact

## Output

```
═══════════════════════════════════════════════════════════
  AI DEVELOPMENT SYSTEM EVALUATION
  System: {name}
  Components: N instruction files, M hooks, K state files
  Total always-loaded: ~X tokens
═══════════════════════════════════════════════════════════

┌─────────────────────┬───────┬──────────────────────────────┐
│ Dimension           │ Score │ Key Finding                  │
├─────────────────────┼───────┼──────────────────────────────┤
│ 1. Endurance        │  A    │ ...                          │
│ 2. Context Eff.     │  B    │ ...                          │
│ 3. Overhead         │  C    │ ...                          │
│ 4. Adherence        │  A    │ ...                          │
│ 5. Robustness       │  B    │ ...                          │
│ 6. Collaboration    │  B    │ ...                          │
├─────────────────────┼───────┼──────────────────────────────┤
│ OVERALL             │  B    │                              │
└─────────────────────┴───────┴──────────────────────────────┘
```

## Design principles

- **Implementation-agnostic** — evaluates the design, not whether it worked once
- **Assumption stress testing** — challenges whether each component's premise still holds with current models
- **Flow simulation** — traces token cost through the full lifecycle before scoring
- **Move impact check** — before recommending any restructure, verifies the move won't break trigger chains
