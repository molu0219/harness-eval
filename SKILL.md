---
name: harness-eval
description: General evaluation framework for AI development systems — scores design quality across 6 dimensions regardless of implementation
---

# /harness-eval

Evaluate any AI development system's design effectiveness. Not "did it work this time" but "is this design capable of working well?"

Applicable to: Claude Code harness, Cursor rules, Windsurf, Aider, custom CLAUDE.md setups, or any AI-assisted development configuration.

## Step 0: Parse Arguments

`/harness-eval` → evaluate current project's AI development system
`/harness-eval <path>` → evaluate a specific system at given path
`/harness-eval compare <path1> <path2>` → compare two systems

### Evaluation Scope (Critical)

The evaluation must be scoped to the **target system only** — not the entire Claude Code environment.

| Invocation | Scope |
|------------|-------|
| `/harness-eval` | Current project directory: `./CLAUDE.md`, `./hooks/`, `./rules/`, `./settings.json`, `./.claude/` |
| `/harness-eval <path>` | Files strictly within `<path>` only |
| `/harness-eval compare <p1> <p2>` | Each path independently scoped |

**Global `~/.claude/` files are NOT in scope** unless:
- The target system IS a harness repo that installs to `~/.claude/` (e.g., via `install.sh` or symlinks), AND
- You are evaluating the harness as a whole system

In that case, label global-installed components separately as **[shared infrastructure]** vs **[project-level]** in the component list.

> Why this matters: without scoping, `/harness-eval` picks up the user's global CLAUDE.md and rules, which inflates the "always-loaded" token count and conflates multiple systems. You'd be evaluating "the user's entire Claude Code setup" instead of "this specific harness."

---

## Step 1: Discover System Components

Read all AI instruction files **within the evaluation scope** (defined in Step 0). Don't assume any specific structure — discover what exists.

Look for (in order):
1. **System prompts / instructions**: CLAUDE.md, .cursorrules, .windsurfrules, .aider*, rules/*.md, any file referenced by AI config
2. **Automation**: hooks, scripts, pre/post processors, CI integrations
3. **State management**: task tracking files (SPEC.md, TODO.md, etc.), progress files
4. **Knowledge**: architecture docs, decision logs, graphify output, wiki
5. **Configuration**: settings.json, .claude/, .cursor/, tool configs

For each component found, note:
- **Load timing**: always (every API call), per-session (once), per-event (conditional), on-demand (explicit read)
- **Size**: bytes
- **Purpose**: instruction, enforcement, state, reference

If the system installs to a global location (e.g., `~/.claude/`), trace the install mechanism and evaluate the **source files**, not the installed copies.

---

## Step 1.5: Flow Simulation (Development Flow Simulation)

> Before evaluating dimensions, walk through the development flow. Not actually running code, but tracing: what happens at each step, what gets called, what's in context, how many tokens are consumed.

### Simulation Method

Draw the complete development lifecycle, marking at each transition point:

```
[Phase] → (what gets triggered) → {context state} → ~token cost
```

### Standard Development Lifecycle

The following are phases that any AI development system goes through. For each phase, derive what happens from the system's design documents (CLAUDE.md, hooks, settings):

#### Phase 0: Session Start
- AI loads system instructions (instruction files + rules) → record token cost
- Hooks trigger? What do they output? → add to context
- How many files does AI need to read before starting work? → list them
- **Check**: how many steps from session start to first line of code? Is each step necessary?

#### Phase 1: Development
- AI writes code → which hooks trigger? How often?
- What is the hook overhead per Edit/Write in tokens?
- Are there hooks that run in this phase but produce no output? (idle cost)
- **Check**: is the ratio of hook frequency to code output reasonable?

#### Phase 2: Task Completion
- AI completes task → how many ceremony steps are needed?
- Does each step have the previous step's output as input? (chain integrity)
- Which steps trigger hooks? block or warn?
- **Check**: is there any circular dependency between ceremony steps?

#### Phase 3: Verification (feature group)
- All tasks complete → who triggers Evaluator? How?
- Evaluator spawn → AI writes to {task tracking file} → {completion gate hook} re-triggers → could this loop infinitely?
- Same for Review
- **Check**: does the trigger chain converge (eventually ends at archive), or could it diverge?

#### Phase 4: Session End / Compact
- AI compact → what is preserved? what is lost?
- How much can the next session recover? How?
- **Check**: is the information gap before/after compact within acceptable range?

### Flow Checklist

| Check | Question | Fail Condition |
|-------|----------|---------------|
| **Step Count** | How many steps from session start to first line of code? | >5 steps = too heavy |
| **Circular Dependency** | hook A triggers → AI action → hook A triggers again → ... could this loop infinitely? | Any non-converging loop |
| **Dead End** | Is there any phase whose output has no downstream consumers? | Output exists but nobody reads it |
| **Missing Link** | What input does the next step of phase N need? Did the previous step produce it? | Needs X but nobody produces X |
| **Token Proportionality** | What percentage of total tokens does ceremony consume for one task? | Ceremony > 30% of total |
| **Tool Appropriateness** | Is the tool/hook called at each step the most appropriate? | Using UserPromptSubmit hook for something that should be PostToolUse |
| **Observability Coverage** | Does every phase leave a trackable log/artifact? | Any phase with no record = fail |

### Observability Coverage Tracking

For each Phase, ask three questions:
1. **What happened in this phase?** → Is there a structured event/artifact recording it?
2. **If something went wrong, can we trace back?** → Is there enough information to reproduce or diagnose?
3. **Is the record automatic or does the AI have to remember to write it?** → Automatic = reliable, Manual = may be missed

| Phase | Expected Log | Automatic/Manual | Consequence if Missing |
|-------|-------------|-----------------|----------------------|
| **0. Boot** | boot event (which project, which model, which mode) | Automatic (hook) | Don't know when session started or with what config |
| **1. Development** | git commits (what changed, why) | Manual (AI commits) | Don't know what decisions were made along the way |
| **2. Task Complete** | task completion artifact (what was done, how it was verified) | Manual but gated | Skipping artifact → {completion gate hook} blocks |
| **3. Verification** | verification log (Evaluator/Review PASS/FAIL) | Manual but gated | Skipping → {completion gate hook} warns |
| **4. Session End** | handoff note / decision log / retro event | Manual but checked | {session hook} checks handoff; {compact hook} forces decision write |
| **Hook Trigger** | hook debug log + timing event | Automatic (hook utils) | Don't know how long hooks took or if they errored |

Scoring:
- All phases have automated logs = A
- Critical phases (2,3) are gated, others have manual + check = B
- Any phase with no record at all = C
- Most phases have no record = F

### Token Flow Tracking

For a typical task (write one function + test), estimate token cost per phase:

```
Phase 0: Boot
  Always-loaded: {N} tokens (CLAUDE.md + rules)
  Hook output: {M} tokens ({context injection hook})
  File reads: {K} tokens (SPEC + optional)
  → Subtotal: {X} tokens before first code

Phase 1: Development (~20 API calls for simple task)
  Per-call fixed: {N} tokens × 20 calls = {Y}
  Hook outputs: ~{Z} tokens total (most idle)

Phase 2: Completion
  Ceremony: verify + log + mark = ~{C} tokens
  Hook: {completion gate hook} block/action prompt = ~{H} tokens

Phase 3: Verification (if >3 tasks)
  Evaluator spawn: ~{E} tokens
  Reviewer spawn: ~{R} tokens

Total: {GRAND TOTAL} tokens per task
Overhead ratio: (Phase 0 + hooks + ceremony) / total = {RATIO}%
```

### Flow Simulation Output

```
FLOW: Session Start → [N steps] → First Code → [20 edits, 2 hooks/edit] → Task Done → [3 ceremony] → Feature Group Complete → [action prompt → Evaluator → Review → Archive] → Compact

Token breakdown:
  Boot:       {N} tokens (X%)
  Per-call:   {N} tokens (X%)  ← largest, unavoidable
  Hooks:      {N} tokens (X%)
  Ceremony:   {N} tokens (X%)
  Verify:     {N} tokens (X%)

Issues found:
  ✓ No circular dependency
  ✓ No dead ends
  ⚠ {completion gate hook} action prompt → AI writes {task tracking file} → hook re-triggers (but converges: second trigger won't produce new action prompt since Evaluator result already exists)
```

Flow Simulation results affect Step 2 scoring:
- Circular dependency → Robustness downgraded
- Too many steps → Overhead downgraded
- Token proportionality too high → Overhead downgraded
- Missing link → Adherence downgraded (broken flow = rules can't execute)
- Dead end → Context Efficiency downgraded (output nobody reads = waste)
- Observability gap → Adherence downgraded (can't retroactively verify flow executed correctly)

---

## Step 1.7: Assumption Stress Test

> "Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing." — Anthropic Engineering

For each component (hook, rule, instruction, ceremony step), ask:

```
What does this component assume the AI cannot do on its own? Is that assumption still valid with the current model?
```

### Method

1. List all components
2. For each component, write out its **capability assumption** (assumes AI cannot do X, so this component exists to do X)
3. Validate against the current model's known capabilities: is this assumption still valid?
4. If not → this component is **stale scaffolding** and should be removed or downgraded

### Common Stale Scaffolding Patterns

| Pattern | Assumption | When It No Longer Holds |
|---------|------------|------------------------|
| Inject progress on every prompt | AI can't remember what it did in the previous step | Model context window is large enough and AI manages its own state |
| Force-read specific files | AI won't seek out needed information on its own | Model has tool use and reads proactively |
| Step decomposition reminders | AI can't break down tasks on its own | Model planning capability has improved |
| Sprint decomposition | AI can't maintain consistency across long tasks | Model long-context capability improved (Opus 4.6 removed sprints) |
| Force TDD workflow | AI doesn't write tests | Model brings its own coding best practices |

### When to Run

- After each model upgrade
- When eval finds Overhead score declining
- When a component's idle rate > 50%

### Output

```
ASSUMPTION TEST:
  ✓ {completion gate hook} — assumption: AI skips completion artifact and marks done directly. Holds: without hard gate, it does skip
  ✓ {permission deny list} — assumption: AI attempts dangerous operations. Holds: model will attempt rm -rf etc.
  ⚠ {tech stack detection hook} — assumption: AI doesn't scan dependency files. Partially holds: AI reads but won't necessarily scan proactively
  ✗ {component X} — assumption: [write assumption]. Does not hold: [explain why model can now do this itself]. Recommend removing
```

---

## Step 2: Evaluate Six Dimensions

### Dimension 1: Endurance

> How long can AI work effectively under this system before starting to degrade?

Evaluate the design's impact on session longevity:

| Criteria | Question | Score |
|----------|----------|-------|
| **Context Budget** | How much context do system instructions occupy? How much is left for actual work? | always-loaded bytes / model context window. <5% = A, 5-10% = B, 10-20% = C, >20% = F |
| **Active Ceiling** | Does the system keep active context below the quality degradation threshold during long tasks? | has compaction/archiving that prevents context from filling past ~40% = A, grows linearly but slowly = C, no mechanism = F |
| **Growth Bound** | Does injected context grow without bound as work progresses? | has upper bound mechanism (cache/archive/compact) = A, linear growth but slow = C, unbounded growth = F |
| **Recovery Design** | After context reset, how much state can the system recover? | full reset + structured handoff = A, compact preserves partial = B, no mechanism = F |
| **Recovery Chain** | Does each step of the recovery mechanism have a trigger chain? | each step has explicit trigger = A, relies on AI remembering = C, broken chain = F |
| **Continuity Design** | When crossing sessions, can work continue seamlessly? | has handoff + state file = A, inferred from git history = C, complete break = F |
| **Degradation Curve** | Is quality degradation gradual or a cliff? | has tiered mechanism (lite/full) for gradual degradation = A, crashes before handled = F |

**Calculation**: Context Budget is the most critical metric.

Model context windows (input):
- Claude Opus: ~200K tokens
- Claude Sonnet: ~200K tokens  
- GPT-4: ~128K tokens
- Cursor (effective): ~20-40K tokens per completion

Formula: `context_budget_score = always_loaded_tokens / effective_context_window`

**Active Ceiling note**: Empirically, output quality degrades when active context exceeds ~40% of window size regardless of fixed overhead. A system with 5% always-loaded overhead but no compaction will still hit the Dumb Zone by mid-session on complex tasks. Evaluate whether the system has mechanisms (archiving completed work, compacting completed feature groups, structured handoff + new session) that prevent accumulation from reaching this ceiling.

### Dimension 2: Context Efficiency

> At any moment, is what's in context what the AI currently needs — and what it doesn't already know?

| Criteria | Question | Score |
|----------|----------|-------|
| **Redundancy** | How much of the injected information does the AI already know? | 0% redundancy = A, <20% = B, 20-50% = C, >50% = F |
| **Map vs Manual** | Is the instruction file an index (pointing to deeper docs) or an encyclopedia (explaining everything itself)? | ~100 line index pointing to details = A, mixed = C, giant single file = F |
| **Signal-to-Noise** | How much of always-loaded content is actually relevant to the current task? | >80% relevant = A, 50-80% = C, <50% = F |
| **Timeliness** | Is information injected when needed, or front-loaded? | most loaded on demand = A, mixed = C, all always-loaded = F |
| **Freshness** | Is the instruction file a living feedback loop (updated when agents fail), or a static document written once and forgotten? | updated after each agent failure class = A, updated occasionally by humans = C, never updated = F |
| **Layering** | Are static rules, dynamic state, and reference docs separated into layers? | 3+ clearly separated layers = A, 2 layers = C, all mixed together = F |
| **Deduplication** | Is the same concept **defined** in multiple places? | no duplicate definitions = A, minor = C, severe = F |

**Redundancy is the most important criterion**. All others ask "what was put in"; this one asks "should it be in there at all."

**How to assess Redundancy — Ownership Tracing**:

For each dynamic injection (hook output) and each auto-loaded instruction, trace the full chain:

```
Who creates this information? → Who modifies it? → Who needs it? → Who delivers it?
```

Classification rules:
1. **Creator = Consumer** → the deliverer in between is redundant. Example: AI modifies SPEC → hook injects SPEC summary → AI reads summary. AI is both creator and consumer; hook is redundant deliverer
2. **Already in context** → redundant injection. Example: CLAUDE.md says "read SPEC" → hook also says "read SPEC"
3. **Creation chain inference** → if A's existence necessarily implies B, informing of B's existence is redundant. Example: project-init creates SPEC and DECISION simultaneously → having SPEC implies having DECISION → telling AI DECISION exists is redundant
4. **AI cannot know on its own** → has value. Example: tech stack detection (AI doesn't scan package.json)

Redundancy = (type 1 + type 2 + type 3) / total injection volume

**How to assess Map vs Manual**:
The three causes of death for giant instruction files (quoting Anthropic Engineering):
1. Crowds out context — less room for actual work
2. Unmaintainable — the bigger it gets, the harder to change; the harder to change, the more stale
3. Cannot be mechanically verified — plain text instructions can't be linted

Map characteristics: main file ≤ 150 lines, uses progressive disclosure pointing to rules/, docs/, SPEC.
Manual characteristics: main file > 300 lines, contains complete flows, examples, templates.

**How to assess Signal-to-Noise**: 
Read all always-loaded content. For each paragraph, ask: "In a task that only changes one CSS class, is this paragraph useful?" Count useful vs total. If most content is only relevant to complex multi-agent scenarios but loads for every task → low signal.

**How to assess Layering**:
- Layer 1 (always): Core behavior rules that apply to every interaction
- Layer 2 (conditional): Task-specific context loaded when detected (e.g., has SPEC → load SPEC rules)
- Layer 3 (on-demand): Reference docs the AI reads only when it needs them
- If all three exist and are clearly separated → A. If everything is in one file → F.

**How to assess Deduplication**:
Distinguish between **definition** and **reference**:
- **Definition**: teaches the concept (what to do, how to do it, rules). Example: coding-style.md's "Error Handling" section with 4 bullet points
- **Reference**: mentions the concept in passing or points to the definition. Example: code-review.md severity example "missing error handling causes crash"

Only count as duplicate if the same concept has **2+ definitions**. References are fine — they're how you avoid duplication.

Grep alone will over-count (matches keywords in both definitions and references). Manual classification needed.

### Dimension 3: Overhead

> How much does this system consume on its own? Is it worth the value it provides?

| Criteria | Question | Score |
|----------|----------|-------|
| **Fixed Tax** | Fixed token cost per API call | <2000 tokens = A, 2000-5000 = B, 5000-10000 = C, >10000 = F |
| **Variable Tax** | How many tokens does the system append after each human input? | <50 tokens = A, 50-200 = C, >200 = F |
| **Ceremony** | How many "process steps" (rather than "writing code") are needed to complete a task? | <3 steps = A, 3-5 = C, >5 = F |
| **Boot Sequence** | How many files must be read at session start before work can begin? | 0-1 file = A, 2-3 = B, 4+ = C, must run commands too = F |
| **Idle Components** | What fraction of components run but produce no output most of the time? | <20% = A, 20-50% = C, >50% = F |
| **Trigger Placement** | Is each reminder/check placed at the trigger point closest to where the action occurs? | all precise = A, mostly precise = B, some misplaced = C, severely misplaced = F |

**How to assess Trigger Placement**:
Map each reminder to the optimal trigger point:
- {task tracking file} change-related → PostToolUse Edit {task file} (not every prompt)
- Code quality → PostToolUse Edit/Write code (not session start)
- Session-level → UserPromptSubmit first time only (not every time)
- Before leaving → Stop hook or PreCompact

If a reminder is placed at UserPromptSubmit (every prompt) but the action only occurs when editing {task file} → misplaced = waste.

**How to assess Ceremony**:
Trace the path from "task done" to "officially done":
- Just commit → 0 ceremony
- Commit + update task file → 1 step
- Commit + update task + write log + run verify + mark complete → 4 steps
- Commit + log + mark + spawn evaluator + spawn reviewer + archive → 6 steps

More ceremony = more consistent quality, but also more overhead. Score based on whether the ceremony is proportional to the task complexity (one-size-fits-all ceremony = bad).

### Dimension 4: Adherence

> Can the system's rules actually be followed? Is there an enforcement mechanism?

| Criteria | Question | Score |
|----------|----------|-------|
| **Enforceability** | Are rules mechanically enforced (lint/hook block) or soft suggestions? | critical rules have hard gate + error with fix instructions = A, hard gate but no fix instructions = B, all soft = F |
| **Error Remediation** | When a hook/linter blocks the AI, does the error message tell it exactly how to fix the problem? | all block messages include specific fix instructions = A, some do = B, blocks with no guidance = F |
| **Phase Separation** | Does the system enforce separation between planning (research/design) and execution (writing code)? | explicit plan-then-execute workflow with human checkpoint = A, AI decides when to stop planning = C, no separation = F |
| **Observability** | Can you tell after the fact whether rules were followed? | has artifact/log that can be verified = A, can only look at code = C, cannot verify = F |
| **Consistency** | Are there contradictions between rules? | 0 contradictions = A, some but priority can be determined = C, some that cannot be resolved = F |
| **Proportionality** | Do important rules have strong enforcement while unimportant ones are lighter? | clearly tiered = A, all treated equally = C, important ones actually not enforced = F |
| **Escape Hatch** | When rules conflict with reality, is there a reasonable exit mechanism? | has explicit override/PAUSE conditions = A, can only violate = C, no way out = F |
| **Cross-Reference** | Do the rules in instruction files match what hooks actually check? | fully consistent = A, minor phantom references = C, severely inconsistent = F |

**How to assess Cross-Reference**:
1. Extract all statements from instruction files (CLAUDE.md/rules) that reference hook behavior
2. For each: does the hook actually do this? (phantom reference = says it does but doesn't)
3. Extract all hook outputs: does the instruction file tell AI how to respond? (unhandled output = does it but doesn't say so)
4. Do settings.json deny/ask lists match CLAUDE.md's prohibited behavior list?

**How to assess Enforceability — Tool Chain Mapping**:

For each rule in the system, trace its enforcement tool chain:

| Enforcement Tool | Expected Compliance Rate | Example |
|-----------------|------------------------|---------|
| permission deny list | ~100% | rm -rf, force push, deploy |
| permission ask list | ~100% | git push, package install |
| hook exit 2 (block) | ~100% | completing task without log artifact, hardcoded secret |
| hook warn + fix instruction | ~80-90% | modifying code without test, diff too large |
| hook warn (no fix instruction) | ~60-70% | archive reminder, expired decision |
| spawned Reviewer post-check | ~90% | reinventing the wheel, test coverage |
| rules/instruction file soft text | ~40-70% | systems thinking, trade-off reasoning |
| no enforcement at all | ~20-40% | relies entirely on AI's own judgment |

**Method**:
1. List all rules in rules/CLAUDE.md (every `-` bullet or checklist item)
2. For each rule, mark its enforcement tool (deny/block/warn/reviewer/soft/none)
3. Calculate weighted compliance rate: `Σ(expected compliance rate per rule) / total rule count`
4. Find **high-value rules with low enforcement** (important rules that only have soft enforcement)

**Scoring**:
- Weighted compliance rate > 80% = A (critical rules all have hard enforcement)
- 60-80% = B (most have enforcement)
- 40-60% = C (mixed)
- < 40% = F (mostly soft)

**Important**: more hard rules is not automatically better. The goal is **important rules have hard enforcement, unimportant ones can be soft**. A system with only 10 hard rules that all cover critical operations = A. 100 hard rules but AI is blocked from doing any work = F (over-enforcement).

**How to assess Error Remediation**:
A block message that only says "violation detected" forces the AI to guess the fix, often looping. A block message that says "violation detected: do X instead" is a self-correcting system. Check every `exit 2` path in your hooks: does each one print a specific actionable instruction? Example of good remediation: `🚫 Task marked complete without Log entry. Add: - HH:MM {tid} done. What you did. How you verified.` Example of poor remediation: `🚫 BLOCK: validation failed`.

**How to assess Phase Separation**:
Does the system have a distinct planning phase where a plan is produced and optionally reviewed before any code is written? Signs of good phase separation: explicit plan files (SPEC.md, feature lists), human review checkpoint before execution begins, AI cannot write code until a plan is approved. Signs of absent phase separation: AI switches between planning and coding within the same turn, no artifact captures the plan, no human checkpoint before execution. Quote from Boris Tane (Cloudflare): "Never let an agent write code before you've reviewed and approved a written plan. That separation of planning and execution is the most important thing I do."

### Dimension 5: Robustness

> Can the system itself break? What happens when it does?

| Criteria | Question | Score |
|----------|----------|-------|
| **Graceful Degradation** | If one component breaks, do others still work? | components independent, failed ones skipped = A, has dependencies but can fallback = C, cascading failure = F |
| **Error Isolation** | Does one hook erroring affect the entire session? | errors caught, session continues = A, interrupts flow = F |
| **State Consistency** | How many sources of truth are there? Can they conflict? | single source of truth = A, multiple but synchronized = C, multiple and can drift = F |
| **Empty State** | Does the system work normally when there's nothing (new repo, no SPEC, no DECISION)? | fully normal = A, partial functionality = C, errors or hangs = F |
| **Mid-Session Change** | If a human changes code/requirements/config mid-session, can the system adapt? | has detection + adjustment mechanism = A, relies on AI judgment = C, will conflict = F |
| **Repo as Record** | Is all knowledge the AI needs in version-controlled files? | all in repo/config = A, some external (Slack/Docs) = C, critical knowledge not in repo = F |
| **Entropy Defense** | Is there a mechanism preventing AI from copying bad patterns AND periodically cleaning up AI-generated low-quality code? | has lint + structural tests + GC agent for AI slop = A, has lint only = B, none = F |
| **Dependency Liveness** | Are all referenced components still alive? | 0 dead references = A, some but non-critical = C, critical dependency dead = F |

**How to assess Repo as Record**:
"Things not in the repository don't exist for the agent" — Anthropic Engineering
1. List all knowledge AI needs to make decisions (architecture, conventions, plans, decision records)
2. For each: is it in version-controlled files in the repo? Or only in Slack/Google Docs/someone's memory?
3. Does global config (~/.claude/) count as repo? → If there's a versioned harness repo with install/symlink mechanism = yes. If written by hand with no backup = no.

**How to assess Entropy Defense**:
Two distinct problems — both need addressing:

**Problem A: AI copies bad patterns** — AI replicates whatever it sees in the repo, including existing bugs and anti-patterns.
1. Is there lint/structural test protecting invariants? (not just style, but structure: "every API must have an error response")
2. Is there a background task scanning for drift? ({session hook} detects repeated modifications, stale decisions)
3. Do lint/hook error messages embed fix instructions so AI can self-correct?

**Problem B: AI-generated slop accumulates** — LLM-generated code tends to reinvent existing functionality, over-engineer simple things, and produce low readability. This accumulates differently from human-written technical debt.
1. Is there a periodic "garbage collection" agent or task that scans for duplicate implementations, dead code, and over-engineered patterns?
2. Does cleanup throughput scale proportionally with generation throughput? (If generating 100 LOC/day, cleanup should also run daily, not quarterly)
3. Are cleanup runs tracked (what was found, what was removed)?

Systems that only address Problem A score B. Systems addressing both score A.

**How to assess Dependency Liveness**:
1. hooks: grep source/bash calls to other scripts → check they exist and are non-empty
2. instruction files: grep referenced docs/hooks/paths → check they exist
3. settings.json: registered hook paths → check they exist and are non-empty
4. docs/: on-demand files referenced by instruction files → check they exist

**How to assess Graceful Degradation**:
Draw the dependency graph of all components. For each:
- If it fails, what else breaks?
- Is there a `|| true`, `2>/dev/null`, or fallback?
- Does it block (exit 2) or warn (echo)?

Single point of failure (one component failure kills the session) = automatic F.

### Dimension 6: Collaboration Quality

> Is the interaction design between human and AI good?

| Criteria | Question | Score |
|----------|----------|-------|
| **Right Info at Right Time** | Does the system give information to the AI only when it needs it? | has state-aware injection = A, fixed injection = C, doesn't inject, lets AI search = F |
| **Noise Level** | How much system output is unnecessary for the human/AI to see? | only outputs when there's actionable info = A, has cache to prevent repetition = B, outputs every time = F |
| **Autonomy Gradient** | When should it act automatically vs ask the human? | has clear tiers (deny/ask/allow) = A, ambiguous = C, all automatic or all manual = F |
| **Transparency** | Can the human see what the AI is doing? | has progress tracking + event log = A, has but scattered = C, black box = F |
| **Multi-Agent Design** | Can multiple AI agents coordinate? | has coordination protocol + conflict resolution = A, simple division of labor = C, not considered = F |
| **Agent Readability** | Is the codebase readable to AI? Does the stack use tech AI knows well? | uses "boring" tech (stable APIs, good training coverage) + worktree isolation = A, mixed = C, uses frameworks AI doesn't know well = F |
| **Failure Analysis** | When something goes wrong, does the system guide "what context/tool/constraint is missing" vs "try again"? | has explicit failure diagnosis flow = A, relies on AI judgment = C, no guidance = F |
| **Config Alignment** | Are permission settings and behavior instructions consistent? | settings deny/ask fully matches CLAUDE.md prohibited behaviors = A, has differences but no contradiction = B, contradictions exist = F |

**How to assess Agent Readability**:
"Prefer boring technology" — frameworks with stable APIs and good training coverage are easier for AI to use correctly.
1. Are the main frameworks commonly seen in AI training data? (React/Express/FastAPI = boring ✓, custom framework = risky)
2. Are there wrappers around opaque upstreams? (sometimes reimplementing a subset beats wrapping opaque upstream behavior)
3. Can the application start isolated instances per git worktree? (AI can spin up its own test environment)

**How to assess Failure Analysis**:
"Human time is the scarcest resource. When things go wrong, the answer is not to try harder, but what context/tool/constraint is missing"
1. Does the system have a PAUSE mechanism (when uncertain → stop and ask)?
2. When a hook blocks, does it tell the AI specifically what's missing + how to fix it?
3. When there are repeated failures, is there an escalation mechanism (rather than keep retrying)?

**How to assess Config Alignment**:
1. List {permission config}'s deny list → does {instruction file}'s prohibited behavior have corresponding entries?
2. List {instruction file}'s prohibited behaviors → does {permission config} support them with deny or ask?
3. If instruction file says "don't push" but permission config doesn't block it → soft rule with no hard backing
4. If permission config denies an operation but instruction file doesn't mention it → AI doesn't understand why it's blocked

---

## Step 2.5: Move Impact Check

> When eval recommends moving content in Step 4, this check must be run first.

Three questions before moving:
1. **Who references it?** — grep content keywords across all files, find all references
2. **Does the target have a trigger?** — if target is docs/ (on-demand), when does AI know to read it? No trigger = effectively doesn't exist
3. **Was the source updated?** — after moving, have the places that referenced this content (CLAUDE.md, hooks) been updated?

Failure pattern: moving content to docs/ to optimize Token/Space, but no trigger mechanism for AI to know when to read it → content effectively doesn't exist.

---

## Step 3: Score and Generate Report

### Scoring Rules

For each dimension, average the criteria scores:
- A (Excellent) = 4, B (Good) = 3, C (Adequate) = 2, D (Poor) = 1, F (Failing) = 0, N/A = skip

Dimension score: average of criteria → map back to letter grade
- ≥3.5 = A, ≥2.5 = B, ≥1.5 = C, ≥0.5 = D, <0.5 = F

**Overall** = weighted average:
- Robustness ×2 (if system breaks, nothing else matters)
- Endurance ×1.5 (determines maximum useful work)
- Adherence ×1.5 (determines quality of work)
- Context Efficiency ×1, Overhead ×1, Collaboration ×1

### Output Format

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

### Per-Dimension Detail

For each dimension, output:
1. Criteria table with scores and evidence
2. Strongest point (what the design does well)
3. Weakest point (biggest design flaw)
4. One specific, actionable fix

### Comparative Notes

If evaluating a system with hooks + state management (e.g., harness):
- Compare to baseline: "same project with just a CLAUDE.md and no hooks"
- Is the added complexity worth the added control?

If evaluating a minimal system (e.g., just .cursorrules):
- Note what's missing that would be valuable
- Note what's gained by simplicity (lower overhead, less to break)

---

## Step 4: Recommendations

Max 5, prioritized by:
1. Robustness fix (system is broken)
2. Endurance fix (AI can't sustain work)  
3. Overhead reduction (wasting money/context)
4. Adherence improvement (quality at risk)
5. Context/Collaboration polish (nice to have)

Each recommendation:
- What to change (specific)
- Why (which criteria it improves)
- Trade-off (what you give up)

---

## Reference: System Archetypes

Common AI development system patterns and their typical scores:

| Archetype | Example | Endurance | Context | Overhead | Adherence | Robustness | Collab |
|-----------|---------|-----------|---------|----------|-----------|------------|--------|
| **Bare** | No config at all | A | F | A | F | A | F |
| **Single-file** | One CLAUDE.md | B | C | B | D | A | D |
| **Rules-based** | .cursorrules + rules/ | B | B | B | D | A | C |
| **Hooks-enhanced** | CLAUDE.md + hooks | C | B | C | B | C | B |
| **Full harness** | SPEC + hooks + multi-agent | D | B | D | A | C | A |
| **Over-engineered** | Everything + kitchen sink | F | D | F | B | D | B |

The sweet spot depends on project complexity:
- Solo weekend project → Single-file or Rules-based
- Team project, 1-3 months → Hooks-enhanced
- Complex multi-agent, long-running → Full harness
- If your overhead score is worse than your adherence score → you're over-engineered
