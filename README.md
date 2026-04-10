<div align="right">

[繁體中文](README-zh.md) | **English**

</div>

# harness-eval

> Evaluate AI development systems by design, not by luck.

[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet?style=flat-square)](https://claude.ai/code)
[![Evaluates Cursor](https://img.shields.io/badge/evaluates-Cursor_rules-blue?style=flat-square)](https://cursor.sh)
[![Evaluates Windsurf](https://img.shields.io/badge/evaluates-Windsurf_rules-teal?style=flat-square)](https://windsurf.ai)
[![npx install](https://img.shields.io/badge/install-npx_skills_add-orange?style=flat-square)](https://www.npmjs.com/package/skills)

A **Claude Code** skill that scores any AI-assisted development configuration across **6 dimensions** and **40+ criteria** — answering not "did it work this time?" but "is this design capable of working well consistently?"

Runs in: Claude Code — Can evaluate: Claude Code harness · Cursor rules · Windsurf rules · Aider · CLAUDE.md setups · any AI dev config

---

## Who this is for

| You are… | Why this helps |
|----------|----------------|
| **Adopting Claude Code** for a team | Audit your harness design before rolling out to everyone |
| **Hitting recurring failures** — context degradation, AI skipping rules, agent loops | Diagnose the root cause in the *design*, not the model |
| **Authoring a harness** | Verify your setup holds up under real workloads, not just the happy path |
| **Comparing two setups** | Get objective scores side by side |
| **After a model upgrade** | Catch stale scaffolding — assumptions that held with old models but no longer do |

---

## Install

```bash
npx skills add molu0219/harness-eval
```

The [Skills CLI](https://www.npmjs.com/package/skills) auto-detects your AI tool and installs to the right directory.

<details>
<summary>Manual install</summary>

```bash
mkdir -p ~/.claude/skills/harness-eval
curl -fsSL https://raw.githubusercontent.com/molu0219/harness-eval/main/SKILL.md \
  -o ~/.claude/skills/harness-eval/SKILL.md
```

For Traditional Chinese:
```bash
curl -fsSL https://raw.githubusercontent.com/molu0219/harness-eval/main/SKILL-zh.md \
  -o ~/.claude/skills/harness-eval/SKILL.md
```

</details>

---

## Usage

Open Claude Code, then:

```
/harness-eval                    # evaluate current project (project-level only)
/harness-eval <path>             # evaluate a specific system at given path
/harness-eval --global           # current project + global ~/.claude/
/harness-eval <path> --global    # path + global ~/.claude/
/harness-eval compare <p1> <p2>  # compare two systems side by side
```

### Scope

By default, only **project-level files** are evaluated: `./CLAUDE.md`, `./hooks/`, `./rules/`, `./settings.json`, `./.claude/`

`~/.claude/` is **never included** unless you explicitly pass `--global`. Use `--global` when evaluating a harness repo that deploys to `~/.claude/` and you want to assess the full installed system.

---

## Output

```
═══════════════════════════════════════════════════════════
  AI DEVELOPMENT SYSTEM EVALUATION
  System: my-project
  Components: 5 instruction files, 7 hooks, 2 state files
  Total always-loaded: 14,380 bytes (~3,595 tokens)
═══════════════════════════════════════════════════════════

┌─────────────────────┬───────┬───────────────────────────────────────┐
│ Dimension           │ Score │ Key Finding                           │
├─────────────────────┼───────┼───────────────────────────────────────┤
│ 1. Endurance        │   B   │ always-loaded 1.8%, no active ceiling │
│ 2. Context Eff.     │   B   │ good layering, CLAUDE.md too manual   │
│ 3. Overhead         │   C   │ no resource guardrails on agents      │
│ 4. Adherence        │   B   │ hard gates work, E2E gate is soft     │
│ 5. Robustness       │   B   │ error isolation strong, no entropy GC │
│ 6. Collaboration    │   A   │ autonomy gradient clear               │
├─────────────────────┼───────┼───────────────────────────────────────┤
│ OVERALL             │   B   │                                       │
└─────────────────────┴───────┴───────────────────────────────────────┘
```

Followed by: flow simulation trace · assumption stress test · top 5 prioritized recommendations

---

## The 6 Dimensions

### 1. Endurance — Can AI sustain effective work across long sessions?

| Criteria | What it checks |
|----------|----------------|
| **Context Budget** | What fraction of the model's context window do always-loaded instructions occupy? |
| **Active Ceiling** | Does the system prevent active context from filling past the ~40% quality degradation threshold? |
| **Growth Bound** | Does injected context grow without bound, or is there a cap mechanism? |
| **Recovery Design** | After a context reset (compact/new session), how much state can the system recover? |
| **Recovery Chain** | Does each step of the recovery mechanism have an explicit trigger, or does it rely on AI remembering? |
| **Continuity Design** | When crossing sessions, can work continue via handoff and state files? |
| **Degradation Curve** | Is quality degradation gradual (tiered lite/full) or a sudden cliff? |

### 2. Context Efficiency — Is what's in context actually what's needed right now?

| Criteria | What it checks |
|----------|----------------|
| **Redundancy** | How much injected information does the AI already know from training or prior context? |
| **Map vs Manual** | Is the main instruction file a short index, or a giant encyclopedia? |
| **Signal-to-Noise** | What fraction of always-loaded content is relevant to the current task? |
| **Timeliness** | Is information injected when needed, or front-loaded regardless of relevance? |
| **Freshness** | Is the instruction file updated when agents fail, or static since first written? |
| **Layering** | Are static rules, dynamic state, and reference docs in distinct layers? |
| **Deduplication** | Is the same concept defined in multiple places? |
| **Output Signal Control** | Are hook outputs designed for AI consumption — grep-friendly, file-logged, aggregated? |

### 3. Overhead — How much does the system cost vs. what it delivers?

| Criteria | What it checks |
|----------|----------------|
| **Fixed Tax** | How many tokens are added to every API call unconditionally? |
| **Variable Tax** | How many tokens are appended after each human message? |
| **Ceremony** | How many process steps (not code-writing) are required to complete a task? |
| **Boot Sequence** | How many files must be read before the AI can begin actual work? |
| **Idle Components** | What fraction of hooks run but produce no output in typical usage? |
| **Resource Guardrails** | Are there mechanisms preventing agents from spending disproportionate time on low-value work? |
| **Trigger Placement** | Is each check placed at the trigger point closest to where the action occurs? |

### 4. Adherence — Can the rules actually be followed? Is there enforcement?

| Criteria | What it checks |
|----------|----------------|
| **Enforceability** | Are critical rules enforced by hard gates (hook block/lint) or soft suggestions? |
| **Error Remediation** | When a hook blocks the AI, does the message include specific fix instructions? |
| **Phase Separation** | Is there a distinct planning phase (with human checkpoint) before code is written? |
| **E2E Verification Gate** | Is end-to-end verification required before a task is marked complete? |
| **Backpressure Design** | Does the system apply both upstream guidance AND downstream rejection? |
| **Observability** | Can you verify after the fact whether rules were actually followed? |
| **Consistency** | Are there contradictions between rules? Can priority be determined if so? |
| **Proportionality** | Do important rules have strong enforcement while unimportant ones are lighter? |
| **Escape Hatch** | When rules conflict with reality, is there an explicit override or PAUSE mechanism? |
| **Cross-Reference** | Do instruction file rules match what hooks actually check? |

### 5. Robustness — Does the system break gracefully or catastrophically?

| Criteria | What it checks |
|----------|----------------|
| **Graceful Degradation** | If one component fails, do others continue working? |
| **Error Isolation** | Does one hook erroring interrupt the entire session? |
| **State Consistency** | How many sources of truth exist, and can they drift out of sync? |
| **Empty State** | Does the system work on a fresh repo with nothing initialized? |
| **Mid-Session Change** | If a human changes code or requirements mid-session, does the system adapt? |
| **Repo as Record** | Is all knowledge the AI needs in version-controlled files, not external tools? |
| **Entropy Defense** | Are there mechanisms preventing AI from copying bad patterns or accumulating low-quality code? |
| **Dependency Liveness** | Are all files and paths referenced by hooks and instructions still alive? |
| **State Format Resilience** | Is task state stored in machine-parseable format that resists accidental AI modification? |

### 6. Collaboration — Is the human-AI interaction well designed?

| Criteria | What it checks |
|----------|----------------|
| **Right Info at Right Time** | Does the system inject information only when the AI actually needs it? |
| **Noise Level** | How much system output appears when there is nothing actionable to report? |
| **Autonomy Gradient** | Is there a clear tier of what AI does automatically vs. asks the human (deny/ask/allow)? |
| **Transparency** | Can the human see what AI is doing via progress tracking and event logs? |
| **Multi-Agent Design** | Can multiple agents coordinate, with tool permissions scoped to each role? |
| **Agent Observability Access** | Can the AI agent query logs, metrics, and runtime state to self-diagnose? |
| **Agent Readability** | Does the stack use technology the AI knows well, with per-worktree isolation? |
| **Failure Analysis** | When something goes wrong, does the system guide "what's missing" rather than "try again"? |
| **Config Alignment** | Do permission settings (deny/ask lists) match the prohibited behaviors in instruction files? |

---

## Scoring

### Weights

| Dimension | Weight | Rationale |
|-----------|--------|-----------|
| Robustness | ×2 | if the system breaks, nothing else matters |
| Endurance | ×1.5 | determines maximum useful work per session |
| Adherence | ×1.5 | determines quality of work produced |
| Context Efficiency | ×1 | |
| Overhead | ×1 | |
| Collaboration | ×1 | |

### System archetypes for reference

| Archetype | Endurance | Context Eff. | Overhead | Adherence | Robustness | Collab |
|-----------|-----------|---------|----------|-----------|------------|--------|
| Bare (no config) | A | F | A | F | A | F |
| Single-file CLAUDE.md | B | C | B | D | A | D |
| Rules-based (.cursorrules) | B | B | B | D | A | C |
| Hooks-enhanced | C | B | C | B | C | B |
| Full harness (SPEC + hooks + agents) | D | B | D | A | C | A |
| Over-engineered | F | D | F | B | D | B |

> The sweet spot depends on project complexity. If Overhead score < Adherence score, you're probably over-engineered.

---

## Design principles

- **Implementation-agnostic** — evaluates design quality, not whether it worked once
- **Scoped** — evaluates the target system only, not your entire Claude environment
- **Assumption stress testing** — challenges whether each component's assumption still holds with current models
- **Flow simulation** — traces token cost through the full lifecycle before scoring

## Known limitations

- Functional correctness is out of scope — this evaluates design, not whether AI produces correct code
- Some criteria assume a relatively clean project structure; results may be less actionable on heavily legacy setups
- Assumption stress tests are calibrated for current models — re-run after major upgrades
