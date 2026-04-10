# harness-eval

A Claude Code skill for evaluating AI development systems — scores design quality across 6 dimensions, 40+ criteria.

Not "did it work this time?" but "is this design capable of working well consistently?"

Works with: Claude Code harness, Cursor rules, Windsurf, Aider, CLAUDE.md setups, or any AI-assisted development configuration.

## Language Versions

| File | Language |
|------|----------|
| `SKILL.md` / `SKILL-en.md` | English |
| `SKILL-zh.md` | Traditional Chinese (繁體中文) |

## Install

```bash
mkdir -p ~/.claude/skills/harness-eval
cp SKILL.md ~/.claude/skills/harness-eval/
```

## Usage

```
/harness-eval              # evaluate current project's AI dev system
/harness-eval <path>       # evaluate a specific system at given path
/harness-eval compare <p1> <p2>  # compare two systems
```

**Scope note**: `/harness-eval` evaluates only project-level files (`./CLAUDE.md`, `./hooks/`, `./rules/`, `./settings.json`). Global `~/.claude/` files are excluded unless you are evaluating a harness repo that installs to global. Without this scoping, evaluation picks up the user's entire Claude Code environment instead of the target system.

---

## The 6 Dimensions

### 1. Endurance — Can AI sustain effective work across long sessions?

> How long can AI work effectively under this system before starting to degrade?

| Criteria | What it checks |
|----------|---------------|
| **Context Budget** | What fraction of the model's context window do always-loaded instructions occupy? |
| **Active Ceiling** | Does the system prevent active context from filling past the ~40% quality degradation threshold during long tasks? |
| **Growth Bound** | Does injected context grow without bound as work progresses, or is there a cap mechanism? |
| **Recovery Design** | After a context reset (compact/new session), how much state can the system recover? |
| **Recovery Chain** | Does each step of the recovery mechanism have an explicit trigger, or does it rely on AI remembering? |
| **Continuity Design** | When crossing sessions, can work continue seamlessly via handoff and state files? |
| **Degradation Curve** | Is quality degradation gradual (tiered lite/full mechanisms) or a sudden cliff? |

### 2. Context Efficiency — Is what's in context actually what's needed right now?

> At any moment, is what's in context what the AI currently needs — and what it doesn't already know?

| Criteria | What it checks |
|----------|---------------|
| **Redundancy** | How much injected information does the AI already know from training or prior context? |
| **Map vs Manual** | Is the main instruction file a short index pointing to details, or a giant encyclopedia that explains everything itself? |
| **Signal-to-Noise** | What fraction of always-loaded content is relevant to the current task (not just to rare complex scenarios)? |
| **Timeliness** | Is information injected when needed, or front-loaded regardless of whether it's relevant? |
| **Freshness** | Is the instruction file updated when agents fail (living feedback loop), or static since it was first written? |
| **Layering** | Are static rules, dynamic state, and reference docs separated into distinct layers? |
| **Deduplication** | Is the same concept defined (not just referenced) in multiple places? |
| **Output Signal Control** | Are hook/CI/tool outputs designed for AI consumption — single-line errors, file-logged, aggregated stats rather than raw dumps? |

### 3. Overhead — How much does the system cost vs. what it delivers?

> How much does this system consume on its own? Is it worth the value it provides?

| Criteria | What it checks |
|----------|---------------|
| **Fixed Tax** | How many tokens does the system add to every API call unconditionally? |
| **Variable Tax** | How many tokens does the system append after each human message? |
| **Ceremony** | How many process steps (not code-writing steps) are required to officially complete a task? |
| **Boot Sequence** | How many files must be read at session start before the AI can begin actual work? |
| **Idle Components** | What fraction of hooks/components run but produce no output in typical usage? |
| **Resource Guardrails** | Are there mechanisms preventing agents from spending disproportionate time on low-value activities (test sampling, iteration limits, timeouts)? |
| **Trigger Placement** | Is each reminder/check placed at the trigger point closest to where the action occurs, rather than firing on every prompt? |

### 4. Adherence — Can the rules actually be followed? Is there enforcement?

> Can the system's rules actually be followed? Is there an enforcement mechanism?

| Criteria | What it checks |
|----------|---------------|
| **Enforceability** | Are critical rules enforced by hard gates (hook block/lint) or are they soft suggestions? |
| **Error Remediation** | When a hook blocks the AI, does the error message include specific actionable fix instructions? |
| **Phase Separation** | Does the system enforce a distinct planning phase (with human checkpoint) before code is written? |
| **E2E Verification Gate** | Does the system require end-to-end functional verification before a task is marked complete — not just "code was written"? |
| **Backpressure Design** | Does the system apply both upstream pressure (guiding AI toward correct patterns) AND downstream pressure (rejecting incorrect work)? |
| **Observability** | Can you verify after the fact whether rules were actually followed? |
| **Consistency** | Are there contradictions between rules, and can priority be determined if so? |
| **Proportionality** | Do important rules have strong enforcement while unimportant rules are lighter? |
| **Escape Hatch** | When rules conflict with reality, is there an explicit override or PAUSE mechanism? |
| **Cross-Reference** | Do instruction file rules match what hooks actually check? (No phantom references or unhandled outputs.) |

### 5. Robustness — Does the system break gracefully or catastrophically?

> Can the system itself break? What happens when it does?

| Criteria | What it checks |
|----------|---------------|
| **Graceful Degradation** | If one component fails, do others continue working, or does failure cascade? |
| **Error Isolation** | Does one hook erroring interrupt the entire session, or are errors caught and session continues? |
| **State Consistency** | How many sources of truth exist, and can they drift out of sync? |
| **Empty State** | Does the system work normally on a fresh repo with no SPEC, no DECISION, nothing initialized? |
| **Mid-Session Change** | If a human changes code, requirements, or config mid-session, does the system detect and adapt? |
| **Repo as Record** | Is all knowledge the AI needs to make decisions in version-controlled files, not external tools? |
| **Entropy Defense** | Are there mechanisms preventing AI from copying bad patterns, AND periodically cleaning AI-generated low-quality code? |
| **Dependency Liveness** | Are all files, scripts, and paths referenced by hooks and instructions still alive? |
| **State Format Resilience** | Is task/feature state stored in machine-parseable format (JSON/YAML) that resists accidental AI modification? |

### 6. Collaboration — Is the human-AI interaction well designed?

> Is the interaction design between human and AI good?

| Criteria | What it checks |
|----------|---------------|
| **Right Info at Right Time** | Does the system inject information only when the AI actually needs it, using state-aware injection? |
| **Noise Level** | How much system output appears when there is nothing actionable to report? |
| **Autonomy Gradient** | Is there a clear tier of what the AI does automatically vs. asks the human about (deny/ask/allow)? |
| **Transparency** | Can the human see what the AI is doing via progress tracking and event logs? |
| **Multi-Agent Design** | Can multiple agents coordinate, and does each agent have tool permissions scoped to its role? |
| **Agent Observability Access** | Can the AI agent query logs, metrics, runtime state, and DOM to self-diagnose — not just wait for human intervention? |
| **Agent Readability** | Does the stack use technology the AI knows well (stable APIs, good training coverage), with per-worktree isolation? |
| **Failure Analysis** | When something goes wrong, does the system guide "what context/tool/constraint is missing" rather than "try again"? |
| **Config Alignment** | Do permission settings (deny/ask lists) match the prohibited behaviors in instruction files? |

---

## How it works

1. **Scope** — defines evaluation boundary (project files only, not global config)
2. **Discover** — reads all AI instruction files, hooks, state files, and config in scope
3. **Flow Simulate** — traces the full development lifecycle: session start → coding → task complete → verification → compact
4. **Assumption Stress Test** — challenges each component: "what does this assume the model can't do? still true today?"
5. **Score** — 6 dimensions × weighted criteria → letter grade (A–F) → weighted overall
6. **Recommend** — max 5 actionable fixes, prioritized by impact

## Scoring weights

| Dimension | Weight | Rationale |
|-----------|--------|-----------|
| Robustness | ×2 | if the system breaks, nothing else matters |
| Endurance | ×1.5 | determines maximum useful work per session |
| Adherence | ×1.5 | determines quality of work produced |
| Context Efficiency | ×1 | |
| Overhead | ×1 | |
| Collaboration | ×1 | |

## Output format

```
═══════════════════════════════════════════════════════════
  AI DEVELOPMENT SYSTEM EVALUATION
  System: {name}
  Components: {N} instruction files, {M} hooks, {K} state files
  Total always-loaded: {bytes} bytes (~{tokens} tokens)
═══════════════════════════════════════════════════════════

┌─────────────────────┬───────┬─────────────────────────────┐
│ Dimension           │ Score │ Key Finding                 │
├─────────────────────┼───────┼─────────────────────────────┤
│ 1. Endurance        │ {A-F} │ {one-line}                  │
│ 2. Context Eff.     │ {A-F} │ {one-line}                  │
│ 3. Overhead         │ {A-F} │ {one-line}                  │
│ 4. Adherence        │ {A-F} │ {one-line}                  │
│ 5. Robustness       │ {A-F} │ {one-line}                  │
│ 6. Collaboration    │ {A-F} │ {one-line}                  │
├─────────────────────┼───────┼─────────────────────────────┤
│ OVERALL             │ {A-F} │                             │
└─────────────────────┴───────┴─────────────────────────────┘
```

## Design principles

- **Implementation-agnostic** — evaluates design quality, not whether it worked once
- **Scoped** — evaluates the target system only, not the entire Claude environment
- **Assumption stress testing** — challenges whether each component's assumption still holds with current models
- **Flow simulation** — traces token cost through the full lifecycle before scoring
- **Known limitations** — functional correctness and brownfield/legacy codebase applicability are outside scope

## System archetypes

| Archetype | Example | Endurance | Context Eff. | Overhead | Adherence | Robustness | Collab |
|-----------|---------|-----------|---------|----------|-----------|------------|--------|
| Bare | No config at all | A | F | A | F | A | F |
| Single-file | One CLAUDE.md | B | C | B | D | A | D |
| Rules-based | .cursorrules + rules/ | B | B | B | D | A | C |
| Hooks-enhanced | CLAUDE.md + hooks | C | B | C | B | C | B |
| Full harness | SPEC + hooks + multi-agent | D | B | D | A | C | A |
| Over-engineered | Everything + kitchen sink | F | D | F | B | D | B |

The sweet spot depends on project complexity:
- Solo weekend project → Single-file or Rules-based
- Team project, 1–3 months → Hooks-enhanced
- Complex multi-agent, long-running → Full harness
- If Overhead score < Adherence score → probably over-engineered
