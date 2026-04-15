---
name: team
description: N coordinated agents on shared task list using OpenCode native teams
argument-hint: "[N:agent-type] [ralph] <task description>"
aliases: []
level: 4
---

Spawn N coordinated agents working on a shared task list using OpenCode's native team tools. Replaces the legacy `/swarm` skill (SQLite-based) with built-in team management, inter-agent messaging, and task dependencies -- no external dependencies required.

```
/team N:agent-type "task description"
/team "task description"
/team ralph "task description"
```

### Parameters
- **N** - Number of teammate agents (1-20). Optional; defaults to auto-sizing based on task decomposition.
- **agent-type** - Agent type to spawn for the `team-exec` stage (e.g., executor, debugger, designer, codex, gemini). Optional; defaults to stage-aware routing. Use `codex` to spawn Codex CLI workers or `gemini` for Gemini CLI workers (requires respective CLIs installed). See Stage Agent Routing below.
- **task** - High-level task to decompose and distribute among teammates
- **ralph** - Optional modifier. When present, wraps the team pipeline in Ralph's persistence loop (retry on failure, architect verification before completion). See Team + Ralph Composition below.

### Examples
```bash
/team 5:executor "fix all TypeScript errors across the project"
/team 3:debugger "fix build errors in src/"
/team 4:designer "implement responsive layouts for all page components"
/team "refactor the auth module with security review"
/team ralph "build a complete REST API for user management"
# With Codex CLI workers (requires: npm install -g @openai/codex)
/team 2:codex "review architecture and suggest improvements"
# With Gemini CLI workers (requires: npm install -g @google/gemini-cli)
/team 2:gemini "redesign the UI components"
```

## Architecture
```
User: "/team 3:executor fix all TypeScript errors"
              |
              v
      [TEAM ORCHESTRATOR (Lead)]
              |
              +-- TeamCreate("fix-ts-errors")
              |       -> lead becomes team-lead@fix-ts-errors
              |
              +-- Analyze & decompose task into subtasks
              |       -> explore/architect produces subtask list
              |
              +-- TaskCreate x N (one per subtask)
              |       -> tasks #1, #2, #3 with dependencies
              |
              +-- TaskUpdate x N (pre-assign owners)
              |       -> task #1 owner=worker-1, etc.
              |
              +-- Task(team_name="fix-ts-errors", name="worker-1") x 3
              |       -> spawns teammates into the team
              |
              +-- Monitor loop
              |       <- SendMessage from teammates (auto-delivered)
              |       -> TaskList polling for progress
              |       -> SendMessage to unblock/coordinate
              |
              +-- Completion
                      -> SendMessage(shutdown_request) to each teammate
                      <- SendMessage(shutdown_response, approve: true)
                      -> TeamDelete("fix-ts-errors")
                      -> rm .opencode/state/team-state.json
```

**Storage layout (managed by OpenCode):**
```
~/.opencode/
  teams/fix-ts-errors/
    config.json          # Team metadata + members array
  tasks/fix-ts-errors/
    .lock                # File lock for concurrent access
    1.json               # Subtask #1
    2.json               # Subtask #2 (may be internal)
    3.json               # Subtask #3
    ...
```

## Staged Pipeline (Canonical Team Runtime)
Team execution follows a staged pipeline:

`team-plan -> team-prd -> team-exec -> team-verify -> team-fix (loop)`

### Stage Agent Routing
Each pipeline stage uses **specialized agents** -- not just executors. The lead selects agents based on the stage and task characteristics.

| Stage | Required Agents | Optional Agents | Selection Criteria |
|-------|----------------|-----------------|-------------------|
| **team-plan** | `explore` (quick), `planner` (ultrabrain) | `analyst` (ultrabrain), `architect` (ultrabrain) | Use `analyst` for unclear requirements. Use `architect` for systems with complex boundaries. |
| **team-prd** | `analyst` (ultrabrain) | `critic` (ultrabrain) | Use `critic` to challenge scope. |
| **team-exec** | `executor` (default) | `executor` (ultrabrain), `debugger` (default), `designer` (default), `writer` (quick), `test-engineer` (default) | Match agent to subtask type. Use `executor` (category=ultrabrain) for complex autonomous work, `designer` for UI, `debugger` for compilation issues, `writer` for docs, `test-engineer` for test creation. |
| **team-verify** | `verifier` (default) | `test-engineer` (default), `security-reviewer` (default), `code-reviewer` (ultrabrain) | Always run `verifier`. Add `security-reviewer` for auth/crypto changes. Add `code-reviewer` for >20 files or architectural changes. `code-reviewer` also covers style/formatting checks. |
| **team-fix** | `executor` (default) | `debugger` (default), `executor` (ultrabrain) | Use `debugger` for type/build errors and regression isolation. Use `executor` (category=ultrabrain) for complex multi-file fixes. |

**Routing rules:**

1. **The lead picks agents per stage, not the user.** The user's `N:agent-type` parameter only overrides the `team-exec` stage worker type. All other stages use stage-appropriate specialists.
2. **Specialist agents complement executor agents.** Route analysis/review to architect/critic agents and UI work to designer agents. Tmux CLI workers are one-shot and don't participate in team communication.
3. **Cost mode affects category tier.** In downgrade: `ultrabrain` agents to default, default to `quick` where quality permits. `team-verify` always uses at least default category.
4. **Risk level escalates review.** Security-sensitive or >20 file changes must include `security-reviewer` + `code-reviewer` (ultrabrain category) in `team-verify`.

### Stage Entry/Exit Criteria
- **team-plan**
  - Entry: Team invocation is parsed and orchestration starts.
  - Agents: `explore` scans codebase, `planner` creates task graph, optionally `analyst`/`architect` for complex tasks.
  - Exit: decomposition is complete and a runnable task graph is prepared.
- **team-prd**
  - Entry: scope is ambiguous or acceptance criteria are missing.
  - Agents: `analyst` extracts requirements, optionally `critic`.
  - Exit: acceptance criteria and boundaries are explicit.
- **team-exec**
  - Entry: `TeamCreate`, `TaskCreate`, assignment, and worker spawn are complete.
  - Agents: workers spawned as the appropriate specialist type per subtask (see routing table).
  - Exit: execution tasks reach terminal state for the current pass.
- **team-verify**
  - Entry: execution pass finishes.
  - Agents: `verifier` + task-appropriate reviewers (see routing table).
  - Exit (pass): verification gates pass with no required follow-up.
  - Exit (fail): fix tasks are generated and control moves to `team-fix`.
- **team-fix**
  - Entry: verification found defects/regressions/incomplete criteria.
  - Agents: `executor`/`debugger` depending on defect type.
  - Exit: fixes are complete and flow returns to `team-exec` then `team-verify`.

### Verify/Fix Loop and Stop Conditions
Continue `team-exec -> team-verify -> team-fix` until:
1. verification passes and no required fix tasks remain, or
2. work reaches an explicit terminal blocked/failed outcome with evidence.

`team-fix` is bounded by max attempts. If fix attempts exceed the configured limit, transition to terminal `failed` (no infinite loop).

### Stage Handoff Convention
When transitioning between stages, important context — decisions made, alternatives rejected, risks identified — lives only in the lead's conversation history. If the lead's context compacts or agents restart, this knowledge is lost.

**Each completing stage MUST produce a handoff document before transitioning.**

The lead writes handoffs to `.opencode/handoffs/<stage-name>.md`.

#### Handoff Format

```markdown
## Handoff: <current-stage> → <next-stage>
- **Decided**: [key decisions made in this stage]
- **Rejected**: [alternatives considered and why they were rejected]
- **Risks**: [identified risks for the next stage]
- **Files**: [key files created or modified]
- **Remaining**: [items left for the next stage to handle]
```

#### Handoff Rules

1. **Lead reads previous handoff BEFORE spawning next stage's agents.** The handoff content is included in the next stage's agent spawn prompts, ensuring agents start with full context.
2. **Handoffs accumulate.** The verify stage can read all prior handoffs (plan → prd → exec) for full decision history.
3. **On team cancellation, handoffs survive** in `.opencode/handoffs/` for session resume. They are not deleted by `TeamDelete`.
4. **Handoffs are lightweight.** 10-20 lines max. They capture decisions and rationale, not full specifications (those live in deliverable files like DESIGN.md).

#### Example

```markdown
## Handoff: team-plan → team-exec
- **Decided**: Microservice architecture with 3 services (auth, api, worker). PostgreSQL for persistence. JWT for auth tokens.
- **Rejected**: Monolith (scaling concerns), MongoDB (team expertise is SQL), session cookies (API-first design).
- **Risks**: Worker service needs Redis for job queue — not yet provisioned. Auth service has no rate limiting in initial design.
- **Files**: DESIGN.md, TEST_STRATEGY.md
- **Remaining**: Database migration scripts, CI/CD pipeline config, Redis provisioning.
```

### Resume and Cancel Semantics
- **Resume:** restart from the last non-terminal stage using staged state + live task status. Read `.opencode/handoffs/` to recover stage transition context.
- **Cancel:** `/cancel` requests teammate shutdown, waits for responses (best effort), marks phase `cancelled` with `active=false`, captures cancellation metadata, then deletes team resources and clears/preserves Team state per policy. Handoff files in `.opencode/handoffs/` are preserved for potential resume.
- Terminal states are `complete`, `failed`, and `cancelled`.

### Phase 1: Parse Input
- Extract **N** (agent count), validate 1-20
- Extract **agent-type**, validate it maps to a known OMO subagent
- Extract **task** description

### Phase 2: Analyze & Decompose
Use `explore` or `architect` agent to analyze the codebase and break the task into N subtasks:

- Each subtask should be **file-scoped** or **module-scoped** to avoid conflicts
- Subtasks must be independent or have clear dependency ordering
- Each subtask needs a concise `subject` and detailed `description`
- Identify dependencies between subtasks (e.g., "shared types must be fixed before consumers")

### Phase 3: Create Team
Call `TeamCreate` with a slug derived from the task:

```json
{
  "team_name": "fix-ts-errors",
  "description": "Fix all TypeScript errors across the project"
}
```

**Response:**
```json
{
  "team_name": "fix-ts-errors",
  "team_file_path": "~/.opencode/teams/fix-ts-errors/config.json",
  "lead_agent_id": "team-lead@fix-ts-errors"
}
```

The current session becomes the team lead (`team-lead@fix-ts-errors`).

Write OMO state using the `state_write` MCP tool for proper session-scoped persistence:

```
state_write(mode="team", active=true, current_phase="team-plan", state={
  "team_name": "fix-ts-errors",
  "agent_count": 3,
  "agent_types": "executor",
  "task": "fix all TypeScript errors",
  "fix_loop_count": 0,
  "max_fix_loops": 3,
  "linked_ralph": false,
  "stage_history": "team-plan"
})
```

> **Note:** The MCP `state_write` tool transports all values as strings. Consumers must coerce `agent_count`, `fix_loop_count`, `max_fix_loops` to numbers and `linked_ralph` to boolean when reading state.

**State schema fields:**

| Field | Type | Description |
|-------|------|-------------|
| `active` | boolean | Whether team mode is active |
| `current_phase` | string | Current pipeline stage: `team-plan`, `team-prd`, `team-exec`, `team-verify`, `team-fix` |
| `team_name` | string | Slug name for the team |
| `agent_count` | number | Number of worker agents |
| `agent_types` | string | Comma-separated agent types used in team-exec |
| `task` | string | Original task description |
| `fix_loop_count` | number | Current fix iteration count |
| `max_fix_loops` | number | Maximum fix iterations before failing (default: 3) |
| `linked_ralph` | boolean | Whether team is linked to a ralph persistence loop |
| `stage_history` | string | Comma-separated list of stage transitions with timestamps |

**Update state on every stage transition:**

```
state_write(mode="team", current_phase="team-exec", state={
  "stage_history": "team-plan:2026-02-07T12:00:00Z,team-prd:2026-02-07T12:01:00Z,team-exec:2026-02-07T12:02:00Z"
})
```

**Read state for resume detection:**

```
state_read(mode="team")
```

If `active=true` and `current_phase` is non-terminal, resume from the last incomplete stage instead of creating a new team.

### Phase 4: Create Tasks
Call `TaskCreate` for each subtask. Set dependencies with `TaskUpdate` using `addBlockedBy`.

```json
// TaskCreate for subtask 1
{
  "subject": "Fix type errors in src/auth/",
  "description": "Fix all TypeScript errors in src/auth/login.ts, src/auth/session.ts, and src/auth/types.ts. Run tsc --noEmit to verify.",
  "activeForm": "Fixing auth type errors"
}
```

For tasks with dependencies, use `TaskUpdate` after creation:

```json
// Task #3 depends on task #1 (shared types must be fixed first)
{
  "taskId": "3",
  "addBlockedBy": ["1"]
}
```

**Pre-assign owners from the lead** to avoid race conditions (there is no atomic claiming):

```json
// Assign task #1 to worker-1
{
  "taskId": "1",
  "owner": "worker-1"
}
```

### Phase 5: Spawn Teammates
Spawn N teammates using `Task` with `team_name` and `name` parameters. Each teammate gets the team worker preamble (see below) plus their specific assignment.

```json
{
  "subagent_type": "executor",
  "team_name": "fix-ts-errors",
  "name": "worker-1",
  "prompt": "<worker-preamble + assigned tasks>"
}
```

**Response:**
```json
{
  "agent_id": "worker-1@fix-ts-errors",
  "name": "worker-1",
  "team_name": "fix-ts-errors"
}
```

**IMPORTANT:** Spawn all teammates in parallel (they are background agents). Do NOT wait for one to finish before spawning the next.

### Phase 6: Monitor
The lead orchestrator monitors progress through two channels:

1. **Inbound messages** -- Teammates send `SendMessage` to `team-lead` when they complete tasks or need help. These arrive automatically as new conversation turns (no polling needed).

2. **TaskList polling** -- Periodically call `TaskList` to check overall progress:
   ```
   #1 [completed] Fix type errors in src/auth/ (worker-1)
   #3 [in_progress] Fix type errors in src/api/ (worker-2)
   #5 [pending] Fix type errors in src/utils/ (worker-3)
   ```

#### Task Watchdog Policy

Monitor for stuck or failed teammates:

- **Max in-progress age**: If a task stays `in_progress` for more than 5 minutes without messages, send a status check
- **Suspected dead worker**: No messages + stuck task for 10+ minutes → reassign task to another worker
- **Reassign threshold**: If a worker fails 2+ tasks, stop assigning new tasks to it

### Phase 6.5: Stage Transitions (State Persistence)
On every stage transition, update OMO state:

```
// Entering team-exec after planning
state_write(mode="team", current_phase="team-exec", state={
  "stage_history": "team-plan:T1,team-prd:T2,team-exec:T3"
})

// Entering team-verify after execution
state_write(mode="team", current_phase="team-verify")

// Entering team-fix after verify failure
state_write(mode="team", current_phase="team-fix", state={
  "fix_loop_count": 1
})
```

### Phase 7: Completion
When all real tasks (non-internal) are completed or failed:

1. **Verify results** -- Check that all subtasks are marked `completed` via `TaskList`
2. **Shutdown teammates** -- Send `shutdown_request` to each active teammate
3. **Await responses** -- Each teammate responds with `shutdown_response(approve: true)` and terminates
4. **Delete team** -- Call `TeamDelete` to clean up
5. **Clean OMO state** -- Remove `.opencode/state/team-state.json`
6. **Report summary** -- Present results to the user

When spawning teammates, include this preamble in the prompt to establish the work protocol:

```
You are a TEAM WORKER in team "{team_name}". Your name is "{worker_name}".
You report to the team lead ("team-lead").
You are not the leader and must not perform leader orchestration actions.

== WORK PROTOCOL ==

1. CLAIM: Call TaskList to see your assigned tasks (owner = "{worker_name}").
   Pick the first task with status "pending" that is assigned to you.
   Call TaskUpdate to set status "in_progress":
   {"taskId": "ID", "status": "in_progress", "owner": "{worker_name}"}

2. WORK: Execute the task using your tools (Read, Write, Edit, Bash).
   Do NOT spawn sub-agents. Do NOT delegate. Work directly.

3. COMPLETE: When done, mark the task completed:
   {"taskId": "ID", "status": "completed"}

4. REPORT: Notify the lead via SendMessage:
   {"type": "message", "recipient": "team-lead", "content": "Completed task #ID: <summary>", "summary": "Task #ID complete"}

5. NEXT: Check TaskList for more assigned tasks. If you have more pending tasks, go to step 1.
   If no more tasks are assigned to you, notify the lead:
   {"type": "message", "recipient": "team-lead", "content": "All assigned tasks complete. Standing by.", "summary": "All tasks done, standing by"}

6. SHUTDOWN: When you receive a shutdown_request, respond with:
   {"type": "shutdown_response", "request_id": "<from the request>", "approve": true}

== BLOCKED TASKS ==
If a task has blockedBy dependencies, skip it until those tasks are completed.
Check TaskList periodically to see if blockers have been resolved.

== ERRORS ==
If you cannot complete a task, report the failure to the lead:
{"type": "message", "recipient": "team-lead", "content": "FAILED task #ID: <reason>", "summary": "Task #ID failed"}
Do NOT mark the task as completed. Leave it in_progress so the lead can reassign.

== RULES ==
- NEVER spawn sub-agents or use the Task tool
- NEVER run tmux pane/session orchestration commands
- NEVER run team spawning/orchestration skills or commands (for example $team, $ultrawork, $autopilot, $ralph, omo team ...)
- ALWAYS use absolute file paths
- ALWAYS report progress via SendMessage to "team-lead"
- Use SendMessage with type "message" only -- never "broadcast"
```

## Team + Ralph Composition

When the user invokes `/team ralph`, says "team ralph", or combines both keywords, team mode wraps itself in Ralph's persistence loop. This provides:

- **Team orchestration** -- multi-agent staged pipeline with specialized agents per stage
- **Ralph persistence** -- retry on failure, architect verification before completion, iteration tracking

### Activation
Team+Ralph activates when:
1. User invokes `/team ralph "task"`
2. Keyword detector finds both `team` and `ralph` in the prompt
3. Hook detects `MAGIC KEYWORD: RALPH` alongside team context

### State Linkage
Both modes write their own state files with cross-references:

```
// Team state (via state_write)
state_write(mode="team", active=true, current_phase="team-plan", state={
  "team_name": "build-rest-api",
  "linked_ralph": true,
  "task": "build a complete REST API"
})

// Ralph state (via state_write)
state_write(mode="ralph", active=true, iteration=1, max_iterations=10, current_phase="execution", state={
  "linked_team": true,
  "team_name": "build-rest-api"
})
```

### Execution Flow
1. Ralph outer loop starts (iteration 1)
2. Team pipeline runs: `team-plan -> team-prd -> team-exec -> team-verify`
3. If `team-verify` passes: Ralph runs architect verification (STANDARD tier minimum)
4. If architect approves: both modes complete, run `/cancel`
5. If `team-verify` fails OR architect rejects: team enters `team-fix`, then loops back to `team-exec -> team-verify`
6. If fix loop exceeds `max_fix_loops`: Ralph increments iteration and retries the full pipeline
7. If Ralph exceeds `max_iterations`: terminal `failed` state

### Cancellation
Cancel either mode cancels both:
- **Cancel Ralph (linked):** Cancel Team first (graceful shutdown), then clear Ralph state
- **Cancel Team (linked):** Clear Team, mark Ralph iteration cancelled, stop loop

## Idempotent Recovery
If the lead crashes mid-run, the team skill should detect existing state and resume:

1. Check `~/.opencode/teams/` for teams matching the task slug
2. If found, read `config.json` to discover active members
3. Resume monitor mode instead of creating a duplicate team
4. Call `TaskList` to determine current progress
5. Continue from the monitoring phase

## Configuration
Optional settings via `oh-my-opencode.json` (or `oh-my-openagent.json`):

```jsonc
{
  "team": {
    "maxAgents": 20,
    "defaultAgentType": "executor",
    "monitorIntervalMs": 30000,
    "shutdownTimeoutMs": 15000
  }
}
```

> **Note:** Team members do not have a hardcoded model default. Each teammate is a separate session that inherits the user's configured model. Since teammates can spawn their own subagents, the session model acts as the orchestration layer while subagents can use any category tier.

## State Cleanup
On successful completion:

1. `TeamDelete` handles all OpenCode state:
   - Removes `~/.opencode/teams/{team_name}/` (config)
   - Removes `~/.opencode/tasks/{team_name}/` (all task files + lock)
2. OMO state cleanup via MCP tools:
   ```
   state_clear(mode="team")
   ```
   If linked to Ralph:
   ```
   state_clear(mode="ralph")
   ```
3. Or run `/cancel` which handles all cleanup automatically.

**IMPORTANT:** Call `TeamDelete` only AFTER all teammates have been shut down.

## Git Worktree Integration
MCP workers can operate in isolated git worktrees to prevent file conflicts between concurrent workers.

### How It Works
1. **Worktree creation**: Before spawning a worker, call `createWorkerWorktree(teamName, workerName, repoRoot)` to create an isolated worktree at `.opencode/worktrees/{team}/{worker}` with branch `omo-team/{teamName}/{workerName}`.

2. **Worker isolation**: Pass the worktree path as the `workingDirectory` in the worker's config. The worker operates exclusively in its own worktree.

3. **Merge coordination**: After a worker completes its tasks, use `checkMergeConflicts()` to verify the branch can be cleanly merged, then `mergeWorkerBranch()` to merge with `--no-ff` for clear history.

4. **Team cleanup**: On team shutdown, call `cleanupTeamWorktrees(teamName, repoRoot)` to remove all worktrees and their branches.

## Gotchas

1. **Internal tasks pollute TaskList** -- When a teammate is spawned, the system auto-creates an internal task with `metadata._internal: true`. Filter them when counting real task progress.

2. **No atomic claiming** -- There is no transactional guarantee on `TaskUpdate`. The lead should pre-assign owners before spawning teammates.

3. **Task IDs are strings** -- IDs are auto-incrementing strings ("1", "2", "3"), not integers.

4. **TeamDelete requires empty team** -- All teammates must be shut down before calling `TeamDelete`.

5. **Messages are auto-delivered** -- Teammate messages arrive to the lead as new conversation turns. No polling needed.

6. **shutdown_response needs request_id** -- The teammate must extract and pass back the `request_id`.

7. **Team name must be a valid slug** -- Use lowercase letters, numbers, and hyphens.

8. **Broadcast is expensive** -- Each broadcast sends a separate message to every teammate. Use DM by default.

9. **CLI workers are one-shot, not persistent** -- They run as autonomous one-shot jobs and don't participate in team communication.
