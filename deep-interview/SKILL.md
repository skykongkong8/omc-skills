---
name: deep-interview
description: Socratic deep interview with mathematical ambiguity gating before autonomous execution
argument-hint: "[--quick|--standard|--deep] [--autoresearch] <idea or vague description>"
pipeline: [deep-interview, omo-plan, autopilot]
next-skill: omo-plan
next-skill-args: --consensus --direct
handoff: .opencode/specs/deep-interview-{slug}.md
level: 3
---

<Purpose>
Deep Interview implements Ouroboros-inspired Socratic questioning with mathematical ambiguity scoring. It replaces vague ideas with crystal-clear specifications by asking targeted questions that expose hidden assumptions, measuring clarity across weighted dimensions, and refusing to proceed until ambiguity drops below a configurable threshold (default: 20%). The output feeds into a 3-stage pipeline: **deep-interview → omo-plan (consensus refinement) → autopilot (execution)**, ensuring maximum clarity at every stage.
</Purpose>

<Use_When>
- User has a vague idea and wants thorough requirements gathering before execution
- User says "deep interview", "interview me", "ask me everything", "don't assume", "make sure you understand"
- User says "ouroboros", "socratic", "I have a vague idea", "not sure exactly what I want"
- User wants to avoid "that's not what I meant" outcomes from autonomous execution
- Task is complex enough that jumping to code would waste cycles on scope discovery
- User wants mathematically-validated clarity before committing to execution
</Use_When>

<Do_Not_Use_When>
- User has a detailed, specific request with file paths, function names, or acceptance criteria -- execute directly
- User wants to explore options or brainstorm -- use `omo-plan` skill instead
- User wants a quick fix or single change -- delegate to executor or ralph
- User says "just do it" or "skip the questions" -- respect their intent
- User already has a PRD or plan file -- use ralph or autopilot with that plan
</Do_Not_Use_When>

<Why_This_Exists>
AI can build anything. The hard part is knowing what to build. OMO's autopilot Phase 0 expands ideas into specs via analyst + architect, but this single-pass approach struggles with genuinely vague inputs. It asks "what do you want?" instead of "what are you assuming?" Deep Interview applies Socratic methodology to iteratively expose assumptions and mathematically gate readiness, ensuring the AI has genuine clarity before spending execution cycles.

Inspired by the [Ouroboros project](https://github.com/Q00/ouroboros) which demonstrated that specification quality is the primary bottleneck in AI-assisted development.
</Why_This_Exists>

<Execution_Policy>
- Ask ONE question at a time -- never batch multiple questions
- Target the WEAKEST clarity dimension with each question
- Make weakest-dimension targeting explicit every round: name the weakest dimension, state its score/gap, and explain why the next question is aimed there
- Gather codebase facts via `explore` agent BEFORE asking the user about them
- For brownfield confirmation questions, cite the repo evidence that triggered the question (file path, symbol, or pattern) instead of asking the user to rediscover it
- Score ambiguity after every answer -- display the score transparently
- Do not proceed to execution until ambiguity ≤ threshold (default 0.2)
- Allow early exit with a clear warning if ambiguity is still high
- Persist interview state for resume across session interruptions
- Challenge agents activate at specific round thresholds to shift perspective
</Execution_Policy>

<Autoresearch_Mode>
When arguments include `--autoresearch`, Deep Interview becomes the zero-learning-curve setup lane for `autoresearch`.

- If no usable mission brief is present yet, start by asking: **"What should autoresearch improve or prove for this repo?"**
- After the mission is clear, collect an evaluator command. If the user leaves it blank, infer one only when repo evidence is strong; otherwise keep interviewing until an evaluator is explicit enough to launch safely.
- Keep the usual one-question-per-round rule, but treat **mission clarity** and **evaluator clarity** as hard readiness gates in addition to the normal ambiguity threshold.
- Once ready, do **not** bridge into `omo-plan`, `autopilot`, `ralph`, or `team`. Instead run:
  - `omo autoresearch --mission "<mission>" --eval "<evaluator>" [--keep-policy <policy>] [--slug <slug>]`
- This direct handoff is expected to detach into the real autoresearch runtime tmux session. After a successful handoff, announce the launched session and end the interview lane.
</Autoresearch_Mode>

<Steps>

## Phase 1: Initialize

1. **Parse the user's idea** from `{{ARGUMENTS}}`
2. **Detect brownfield vs greenfield**:
   - Run `Task(subagent_type="explore", category="quick")`: check if cwd has existing source code, package files, or git history
   - If source files exist AND the user's idea references modifying/extending something: **brownfield**
   - Otherwise: **greenfield**
3. **For brownfield**: Run `Task(subagent_type="explore", category="quick")` to map relevant codebase areas, store as `codebase_context`
4. **Initialize state** via `state_write(mode="deep-interview")`:

```json
{
  "active": true,
  "current_phase": "deep-interview",
  "state": {
    "interview_id": "<uuid>",
    "type": "greenfield|brownfield",
    "initial_idea": "<user input>",
    "rounds": [],
    "current_ambiguity": 1.0,
    "threshold": 0.2,
    "codebase_context": null,
    "challenge_modes_used": [],
    "ontology_snapshots": []
  }
}
```

5. **Announce the interview** to the user:

> Starting deep interview. I'll ask targeted questions to understand your idea thoroughly before building anything. After each answer, I'll show your clarity score. We'll proceed to execution once ambiguity drops below 20%.
>
> **Your idea:** "{initial_idea}"
> **Project type:** {greenfield|brownfield}
> **Current ambiguity:** 100% (we haven't started yet)

## Phase 2: Interview Loop

Repeat until `ambiguity ≤ threshold` OR user exits early:

### Step 2a: Generate Next Question

Build the question generation prompt with:
- The user's original idea
- All prior Q&A rounds (conversation history)
- Current clarity scores per dimension (which is weakest?)
- Challenge agent mode (if activated -- see Phase 3)
- Brownfield codebase context (if applicable)

**Question targeting strategy:**
- Identify the dimension with the LOWEST clarity score
- Generate a question that specifically improves that dimension
- State, in one sentence before the question, why this dimension is now the bottleneck to reducing ambiguity
- Questions should expose ASSUMPTIONS, not gather feature lists
- If the scope is still conceptually fuzzy (entities keep shifting, the user is naming symptoms, or the core noun is unstable), switch to an ontology-style question that asks what the thing fundamentally IS before returning to feature/detail questions

**Question styles by dimension:**
| Dimension | Question Style | Example |
|-----------|---------------|---------|
| Goal Clarity | "What exactly happens when...?" | "When you say 'manage tasks', what specific action does a user take first?" |
| Constraint Clarity | "What are the boundaries?" | "Should this work offline, or is internet connectivity assumed?" |
| Success Criteria | "How do we know it works?" | "If I showed you the finished product, what would make you say 'yes, that's it'?" |
| Context Clarity (brownfield) | "How does this fit?" | "I found JWT auth middleware in `src/auth/` (pattern: passport + JWT). Should this feature extend that path or intentionally diverge from it?" |
| Scope-fuzzy / ontology stress | "What IS the core thing here?" | "You have named Tasks, Projects, and Workspaces across the last rounds. Which one is the core entity, and which are supporting views or containers?" |

### Step 2b: Ask the Question

Use `AskUserQuestion` with the generated question. Present it clearly with the current ambiguity context:

```
Round {n} | Targeting: {weakest_dimension} | Why now: {one_sentence_targeting_rationale} | Ambiguity: {score}%

{question}
```

Options should include contextually relevant choices plus free-text.

### Step 2c: Score Ambiguity

After receiving the user's answer, score clarity across all dimensions.

**Scoring prompt** (use `category="ultrabrain"`, temperature 0.1 for consistency):

```
Given the following interview transcript for a {greenfield|brownfield} project, score clarity on each dimension from 0.0 to 1.0:

Original idea: {idea}

Transcript:
{all rounds Q&A}

Score each dimension:
1. Goal Clarity (0.0-1.0): Is the primary objective unambiguous? Can you state it in one sentence without qualifiers? Can you name the key entities (nouns) and their relationships (verbs) without ambiguity?
2. Constraint Clarity (0.0-1.0): Are the boundaries, limitations, and non-goals clear?
3. Success Criteria Clarity (0.0-1.0): Could you write a test that verifies success? Are acceptance criteria concrete?
{4. Context Clarity (0.0-1.0): [brownfield only] Do we understand the existing system well enough to modify it safely? Do the identified entities map cleanly to existing codebase structures?}

For each dimension provide:
- score: float (0.0-1.0)
- justification: one sentence explaining the score
- gap: what's still unclear (if score < 0.9)

Also identify:
- weakest_dimension: the single lowest-confidence dimension this round
- weakest_dimension_rationale: one sentence explaining why it is the highest-leverage target for the next question

5. Ontology Extraction: Identify all key entities (nouns) discussed in the transcript.

{If round > 1, inject: "Previous round's entities: {prior_entities_json from state.ontology_snapshots[-1]}. REUSE these entity names where the concept is the same. Only introduce new names for genuinely new concepts."}

For each entity provide:
- name: string (the entity name, e.g., "User", "Order", "PaymentMethod")
- type: string (e.g., "core domain", "supporting", "external system")
- fields: string[] (key attributes mentioned)
- relationships: string[] (e.g., "User has many Orders")

Respond as JSON. Include an additional "ontology" key containing the entities array alongside the dimension scores.
```

**Calculate ambiguity:**

Greenfield: `ambiguity = 1 - (goal × 0.40 + constraints × 0.30 + criteria × 0.30)`
Brownfield: `ambiguity = 1 - (goal × 0.35 + constraints × 0.25 + criteria × 0.25 + context × 0.15)`

**Calculate ontology stability:**

**Round 1 special case:** For the first round, skip stability comparison. All entities are "new". Set stability_ratio = N/A. If any round produces zero entities, set stability_ratio = N/A (avoids division by zero).

For rounds 2+, compare with the previous round's entity list:
- `stable_entities`: entities present in both rounds with the same name
- `changed_entities`: entities with different names but the same type AND >50% field overlap (treated as renamed, not new+removed)
- `new_entities`: entities in this round not matched by name or fuzzy-match to any previous entity
- `removed_entities`: entities in the previous round not matched to any current entity
- `stability_ratio`: (stable + changed) / total_entities (0.0 to 1.0, where 1.0 = fully converged)

This formula counts renamed entities (changed) toward stability.

**Show your work:** Before reporting stability numbers, briefly list which entities were matched (by name or fuzzy) and which are new/removed.

Store the ontology snapshot (entities + stability_ratio + matching_reasoning) in `state.ontology_snapshots[]`.

### Step 2d: Report Progress

After scoring, show the user their progress:

```
Round {n} complete.

| Dimension | Score | Weight | Weighted | Gap |
|-----------|-------|--------|----------|-----|
| Goal | {s} | {w} | {s*w} | {gap or "Clear"} |
| Constraints | {s} | {w} | {s*w} | {gap or "Clear"} |
| Success Criteria | {s} | {w} | {s*w} | {gap or "Clear"} |
| Context (brownfield) | {s} | {w} | {s*w} | {gap or "Clear"} |
| **Ambiguity** | | | **{score}%** | |

**Ontology:** {entity_count} entities | Stability: {stability_ratio} | New: {new} | Changed: {changed} | Stable: {stable}

**Next target:** {weakest_dimension} — {weakest_dimension_rationale}

{score <= threshold ? "Clarity threshold met! Ready to proceed." : "Focusing next question on: {weakest_dimension}"}
```

### Step 2e: Update State

Update interview state with the new round and scores via `state_write`.

### Step 2f: Check Soft Limits

- **Round 3+**: Allow early exit if user says "enough", "let's go", "build it"
- **Round 10**: Show soft warning: "We're at 10 rounds. Current ambiguity: {score}%. Continue or proceed with current clarity?"
- **Round 20**: Hard cap: "Maximum interview rounds reached. Proceeding with current clarity level ({score}%)."

## Phase 3: Challenge Agents

At specific round thresholds, shift the questioning perspective:

### Round 4+: Contrarian Mode
> You are now in CONTRARIAN mode. Your next question should challenge the user's core assumption. Ask "What if the opposite were true?" or "What if this constraint doesn't actually exist?" The goal is to test whether the user's framing is correct or just habitual.

### Round 6+: Simplifier Mode
> You are now in SIMPLIFIER mode. Your next question should probe whether complexity can be removed. Ask "What's the simplest version that would still be valuable?" or "Which of these constraints are actually necessary vs. assumed?" The goal is to find the minimal viable specification.

### Round 8+: Ontologist Mode (if ambiguity still > 0.3)
> You are now in ONTOLOGIST mode. The ambiguity is still high after 8 rounds. The tracked entities so far are: {current_entities_summary from latest ontology snapshot}. Ask "What IS this, really?" or "Looking at these entities, which one is the CORE concept and which are just supporting?" The goal is to find the essence by examining the ontology.

Challenge modes are used ONCE each, then return to normal Socratic questioning. Track which modes have been used in state.

## Phase 4: Crystallize Spec

When ambiguity ≤ threshold (or hard cap / early exit):

1. **Generate the specification** using `category="ultrabrain"` with the full interview transcript
2. **Write to file**: `.opencode/specs/deep-interview-{slug}.md`

Spec structure:

```markdown
# Deep Interview Spec: {title}

## Metadata
- Interview ID: {uuid}
- Rounds: {count}
- Final Ambiguity Score: {score}%
- Type: greenfield | brownfield
- Generated: {timestamp}
- Threshold: {threshold}
- Status: {PASSED | BELOW_THRESHOLD_EARLY_EXIT}

## Clarity Breakdown
| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Goal Clarity | {s} | {w} | {s*w} |
| Constraint Clarity | {s} | {w} | {s*w} |
| Success Criteria | {s} | {w} | {s*w} |
| Context Clarity | {s} | {w} | {s*w} |
| **Total Clarity** | | | **{total}** |
| **Ambiguity** | | | **{1-total}** |

## Goal
{crystal-clear goal statement derived from interview}

## Constraints
- {constraint 1}
- {constraint 2}
- ...

## Non-Goals
- {explicitly excluded scope 1}
- {explicitly excluded scope 2}

## Acceptance Criteria
- [ ] {testable criterion 1}
- [ ] {testable criterion 2}
- [ ] {testable criterion 3}
- ...

## Assumptions Exposed & Resolved
| Assumption | Challenge | Resolution |
|------------|-----------|------------|
| {assumption} | {how it was questioned} | {what was decided} |

## Technical Context
{brownfield: relevant codebase findings from explore agent}
{greenfield: technology choices and constraints}

## Ontology (Key Entities)
| Entity | Type | Fields | Relationships |
|--------|------|--------|---------------|
| {entity.name} | {entity.type} | {entity.fields} | {entity.relationships} |

## Ontology Convergence
| Round | Entity Count | New | Changed | Stable | Stability Ratio |
|-------|-------------|-----|---------|--------|----------------|
| 1 | {n} | {n} | - | - | - |
| 2 | {n} | {new} | {changed} | {stable} | {ratio}% |
| ... | ... | ... | ... | ... | ... |

## Interview Transcript
<details>
<summary>Full Q&A ({n} rounds)</summary>

### Round 1
**Q:** {question}
**A:** {answer}
**Ambiguity:** {score}% (Goal: {g}, Constraints: {c}, Criteria: {cr})

...
</details>
```

## Phase 5: Execution Bridge

**Autoresearch override:** if `--autoresearch` is active, skip the standard execution options below. The only valid bridge is the direct `omo autoresearch` handoff described above.

After the spec is written, present execution options via `AskUserQuestion`:

**Question:** "Your spec is ready (ambiguity: {score}%). How would you like to proceed?"

**Options:**

1. **Ralplan → Autopilot (Recommended)**
   - Description: "3-stage pipeline: consensus-refine this spec with Planner/Architect/Critic, then execute with full autopilot. Maximum quality."
   - Action: Invoke `Skill("omo-plan")` with `--consensus --direct` flags and the spec file path as context. When consensus completes, invoke `Skill("autopilot")` with the consensus plan. Autopilot skips Phase 0+1 and starts at Phase 2 (Execution).
   - Pipeline: `deep-interview spec → omo-plan --consensus --direct → autopilot execution`

2. **Execute with autopilot (skip ralplan)**
   - Description: "Full autonomous pipeline — planning, parallel implementation, QA, validation. Faster but without consensus refinement."
   - Action: Invoke `Skill("autopilot")` with the spec file path as context. The spec replaces autopilot's Phase 0 — autopilot starts at Phase 1 (Planning).

3. **Execute with ralph**
   - Description: "Persistence loop with architect verification — keeps working until all acceptance criteria pass"
   - Action: Invoke `Skill("ralph")` with the spec file path as the task definition.

4. **Execute with team**
   - Description: "N coordinated parallel agents — fastest execution for large specs"
   - Action: Invoke `Skill("team")` with the spec file path as the shared plan.

5. **Refine further**
   - Description: "Continue interviewing to improve clarity (current: {score}%)"
   - Action: Return to Phase 2 interview loop.

**IMPORTANT:** On execution selection, **MUST** invoke the chosen skill via `Skill()`. Do NOT implement directly. The deep-interview agent is a requirements agent, not an execution agent.

### The 3-Stage Pipeline (Recommended Path)

```
Stage 1: Deep Interview          Stage 2: Ralplan                Stage 3: Autopilot
┌─────────────────────┐    ┌───────────────────────────┐    ┌──────────────────────┐
│ Socratic Q&A        │    │ Planner creates plan      │    │ Phase 2: Execution   │
│ Ambiguity scoring   │───>│ Architect reviews         │───>│ Phase 3: QA cycling  │
│ Challenge agents    │    │ Critic validates          │    │ Phase 4: Validation  │
│ Spec crystallization│    │ Loop until consensus      │    │ Phase 5: Cleanup     │
│ Gate: ≤20% ambiguity│    │ ADR + RALPLAN-DR summary  │    │                      │
└─────────────────────┘    └───────────────────────────┘    └──────────────────────┘
Output: spec.md            Output: consensus-plan.md        Output: working code
```

</Steps>

<Tool_Usage>
- Use `AskUserQuestion` for each interview question — provides clickable UI with contextual options
- Use `Task(subagent_type="explore", category="quick")` for brownfield codebase exploration (run BEFORE asking user about codebase)
- Use `category="ultrabrain"` (temperature 0.1) for ambiguity scoring — consistency is critical
- Use `state_write` / `state_read` for interview state persistence
- Use `Write` tool to save the final spec to `.opencode/specs/`
- Use `Skill()` to bridge to execution modes — never implement directly
- Challenge agent modes are prompt injections, not separate agent spawns
</Tool_Usage>

<Examples>
<Good>
Targeting weakest dimension:
```
Scores: Goal=0.9, Constraints=0.4, Criteria=0.7
Next question targets Constraints (lowest at 0.4):
"You mentioned this should 'work on mobile'. Does that mean a native app,
a responsive web app, or a PWA? And are there specific devices or OS versions
you need to support?"
```
Why good: Identifies weakest dimension, asks a specific question to improve it.
</Good>

<Good>
Gathering codebase facts before asking:
```
[spawns Task(subagent_type="explore", category="quick"): "find authentication implementation"]
[receives: "Auth is in src/auth/ using JWT with passport.js"]

Question: "I found JWT authentication with passport.js in `src/auth/`.
For this new feature, should we extend the existing auth middleware or create
a separate authentication flow?"
```
Why good: Explored first, cited the repo evidence, then asked an informed question.
</Good>

<Good>
Contrarian mode activation:
```
Round 5 | Contrarian Mode | Ambiguity: 42%

You've said this needs to support 10,000 concurrent users. What if it only
needed to handle 100? Would the architecture change fundamentally, or is
the 10K number an assumption rather than a measured requirement?
```
Why good: Challenges a specific assumption that could simplify the solution.
</Good>

<Bad>
Batching multiple questions:
```
"What's the target audience? And what tech stack? And how should auth work?"
```
Why bad: Three questions at once — causes shallow answers.
</Bad>

<Bad>
Asking about codebase facts:
```
"What database does your project use?"
```
Why bad: Should have spawned explore agent to find this.
</Bad>
</Examples>

<Escalation_And_Stop_Conditions>
- **Hard cap at 20 rounds**: Proceed with whatever clarity exists
- **Soft warning at 10 rounds**: Offer to continue or proceed
- **Early exit (round 3+)**: Allow with warning if ambiguity > threshold
- **User says "stop", "cancel", "abort"**: Stop immediately, save state for resume
- **Ambiguity stalls** (same score +-0.05 for 3 rounds): Activate Ontologist mode to reframe
- **All dimensions at 0.9+**: Skip to spec generation even if not at round minimum
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Interview completed (ambiguity ≤ threshold OR user chose early exit)
- [ ] Ambiguity score displayed after every round
- [ ] Every round explicitly names the weakest dimension and why it is the next target
- [ ] Challenge agents activated at correct thresholds (round 4, 6, 8)
- [ ] Spec file written to `.opencode/specs/deep-interview-{slug}.md`
- [ ] Spec includes: goal, constraints, acceptance criteria, clarity breakdown, transcript
- [ ] Execution bridge presented via AskUserQuestion
- [ ] Selected execution mode invoked via Skill() (never direct implementation)
- [ ] State cleaned up after execution handoff
</Final_Checklist>

<Advanced>
## Configuration

Optional settings in `oh-my-opencode.json` (or `oh-my-openagent.json`):

```jsonc
{
  "deepInterview": {
    "ambiguityThreshold": 0.2,
    "maxRounds": 20,
    "softWarningRounds": 10,
    "minRoundsBeforeExit": 3,
    "enableChallengeAgents": true,
    "autoExecuteOnComplete": false,
    "defaultExecutionMode": "autopilot",
    "scoringCategory": "ultrabrain"
  }
}
```

## Resume
If interrupted, run `/deep-interview` again. The skill reads state from `state_read(mode="deep-interview")` and resumes from the last completed round.

## Brownfield vs Greenfield Weights
| Dimension | Greenfield | Brownfield |
|-----------|-----------|------------|
| Goal Clarity | 40% | 35% |
| Constraint Clarity | 30% | 25% |
| Success Criteria | 30% | 25% |
| Context Clarity | N/A | 15% |

## Challenge Agent Modes
| Mode | Activates | Purpose |
|------|-----------|---------|
| Contrarian | Round 4+ | Challenge assumptions |
| Simplifier | Round 6+ | Remove complexity |
| Ontologist | Round 8+ (if ambiguity > 0.3) | Find essence |

## Ambiguity Score Interpretation
| Score Range | Meaning | Action |
|-------------|---------|--------|
| 0.0 - 0.1 | Crystal clear | Proceed immediately |
| 0.1 - 0.2 | Clear enough | Proceed (default threshold) |
| 0.2 - 0.4 | Some gaps | Continue interviewing |
| 0.4 - 0.6 | Significant gaps | Focus on weakest dimensions |
| 0.6 - 0.8 | Very unclear | May need reframing (Ontologist) |
| 0.8 - 1.0 | Almost nothing known | Early stages, keep going |
</Advanced>

Task: {{ARGUMENTS}}
