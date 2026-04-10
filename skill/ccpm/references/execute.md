# Execute — Start Building with Parallel Agents

This phase covers analyzing GitHub issues for parallel work streams and launching agents to execute them.

---

## Issue Analysis

**Trigger**: User wants to understand how to parallelize work on an issue before starting.

### Preflight
- Find the local task file: check `.claude/epics/*/<N>.md` first, then search for `github:.*issues/<N>` in frontmatter.
- If not found: "❌ No local task for issue #<N>. Run a sync first."

### Process

Get issue details: `gh issue view <N> --json title,body,labels`

Read the local task file fully. Identify independent work streams by asking:
- Which files will be created/modified?
- Which changes can happen simultaneously without conflict?
- What are the dependencies between changes?

**Common stream patterns:**
- Database Layer: schema, migrations, models
- Service Layer: business logic, data access
- API Layer: endpoints, validation, middleware
- UI Layer: components, pages, styles
- Test Layer: unit tests, integration tests

Create `.claude/epics/<epic_name>/<N>-analysis.md`:

```markdown
---
issue: <N>
title: <title>
analyzed: <run: date -u +"%Y-%m-%dT%H:%M:%SZ">
estimated_hours: <total>
parallelization_factor: <1.0-5.0>
---

# Parallel Work Analysis: Issue #<N>

## Overview

## Parallel Streams

### Stream A: <Name>
**Scope**: 
**Files**: 
**Can Start**: immediately
**Estimated Hours**: 
**Dependencies**: none

### Stream B: <Name>
**Scope**: 
**Files**: 
**Can Start**: after Stream A
**Dependencies**: Stream A

## Coordination Points
### Shared Files
### Sequential Requirements

## Conflict Risk Assessment

## Parallelization Strategy

## Expected Timeline
- With parallel execution: <max_stream_hours>h wall time
- Without: <sum_all_hours>h
- Efficiency gain: <pct>%
```

**Before writing**: Tell the user: "Accept the file write — we'll review it with revdiff right after."

**After writing**: Launch revdiff for inline review of the analysis. Read `references/revdiff-review.md` and follow the review loop with `.claude/epics/<epic_name>/<N>-analysis.md`. Process all annotations, update the analysis, and re-launch until the user quits without annotations.

**After approval**: "✅ Analysis approved for issue #<N> — N parallel streams identified. Ready to start? Say: start issue <N>"

---

## Starting an Issue

**Trigger**: User wants to begin work on a specific GitHub issue.

### Preflight
1. Verify issue exists and is open: `gh issue view <N> --json state,title,labels,body`
2. Find local task file (as above).
3. Check for analysis file: `.claude/epics/*/<N>-analysis.md` — if missing, run analysis first (or do both in sequence: analyze then start).
4. Verify epic worktree exists: `git worktree list | grep "epic-<name>"` — if not: "❌ No worktree. Sync the epic first."

### Process

**Step 1 — Read the analysis**, identify which streams can start immediately vs. which have dependencies.

**Step 2 — Generate context brief:**

Read the following sources and compile into `.claude/epics/<epic>/<N>-context-brief.md`:

- **PRD**: Read the PRD linked in epic frontmatter (`prd:` field). Extract: problem statement, relevant user stories, functional requirements for this task, constraints, out-of-scope items.
- **Epic architecture**: Read `.claude/epics/<epic>/epic.md`. Extract: architecture decisions, technical approach sections relevant to this task.
- **Project conventions**: Read `CLAUDE.md` from project root if present. Extract: coding patterns, naming conventions, directory structure rules, any project-specific instructions.
- **Cross-task context**: Read sibling task files in `.claude/epics/<epic>/`. For each sibling, note: what it produces, what it consumes, shared interfaces or files, coordination points with this task.
- **Analysis context**: Include key points from `<N>-analysis.md` — stream structure, coordination points, conflict risks.

```markdown
---
issue: <N>
compiled: <run: date -u +"%Y-%m-%dT%H:%M:%SZ">
sources:
  prd: .claude/prds/<name>.md
  epic: .claude/epics/<epic>/epic.md
  task: .claude/epics/<epic>/<N>.md
  analysis: .claude/epics/<epic>/<N>-analysis.md
---

# Context Brief: Issue #<N>

## Requirements Context
(Relevant PRD sections: problem statement, user stories, functional requirements, constraints)

## Architecture Decisions
(From epic: architecture decisions, technical approach relevant to this task)

## Project Conventions
(From CLAUDE.md and codebase: coding patterns, naming conventions, directory structure)

## Cross-Task Context
(Sibling tasks: what they produce/consume, shared interfaces, coordination points)

## Key Constraints
(Out-of-scope items, non-functional requirements, things explicitly NOT to do)
```

**Step 3 — Gate 1: Context brief review:**

Launch a Codex review of the context brief before any agents start. The review verifies that the brief is complete and consistent with the PRD and epic.

```yaml
Task:
  subagent_type: "codex:codex-rescue"
  description: "Gate 1: context brief review #<N>"
  prompt: |
    Review the context brief for Issue #<N> at: .claude/epics/<epic>/<N>-context-brief.md

    Cross-reference against:
    - PRD at: <prd_path>
    - Epic at: .claude/epics/<epic>/epic.md
    - Task spec at: .claude/epics/<epic>/<N>.md

    Check:
    1. Are all relevant PRD requirements for this task captured?
    2. Are the epic's architecture decisions accurately reflected?
    3. Are project conventions from CLAUDE.md included (if CLAUDE.md exists)?
    4. Is cross-task context complete — does the brief mention what sibling tasks produce/consume?
    5. Are there any contradictions between the brief and the source documents?

    Write your review to: .claude/epics/<epic>/reviews/<N>-context-review.md
    Use this frontmatter:
    ---
    gate: context-review
    issue: <N>
    reviewer: codex
    effort: <your chosen effort level>
    verdict: pass | fail
    reviewed: <current datetime>
    ---

    If pass: state what was verified.
    If fail: list specific gaps or contradictions that must be fixed before agents launch.

    Do not modify any files other than the review file.
    --wait --effort <chosen based on brief size>
```

```bash
mkdir -p .claude/epics/<epic>/reviews
```

**If Gate 1 fails**: Read the review findings. Fix the context brief to address the gaps. Re-run Gate 1. If the orchestrator believes the findings are incorrect, follow the Escalation Protocol (see below).

**Step 4 — Create progress tracking:**
```bash
mkdir -p .claude/epics/<epic>/updates/<N>
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

Create `.claude/epics/<epic>/updates/<N>/stream-<X>.md` for each stream:
```markdown
---
issue: <N>
stream: <stream_name>
started: <datetime>
status: in_progress
---
## Scope
## Progress
- Starting implementation
```

**Step 5 — Launch parallel agents** for each stream that can start immediately:

```yaml
Task:
  description: "Issue #<N> Stream <X>"
  subagent_type: "general-purpose"
  prompt: |
    You are working on Issue #<N> in the epic worktree at: ../epic-<name>/
    
    Your stream: <stream_name>
    Your scope — files to modify: <file_patterns>
    Work to complete: <stream_description>
    
    CRITICAL — Read these files before writing any code:
    1. Context brief (full project context): .claude/epics/<epic>/<N>-context-brief.md
    2. Full task spec: .claude/epics/<epic>/<N>.md
    3. Stream analysis: .claude/epics/<epic>/<N>-analysis.md
    4. Epic architecture decisions: .claude/epics/<epic>/epic.md — read the Architecture Decisions section
    5. PRD (requirements & constraints): <prd_path>
    6. Project conventions: CLAUDE.md (if present in project root)
    
    Implementation rules:
    1. Follow the architecture decisions from the epic — do not invent alternative approaches
    2. Follow project conventions from CLAUDE.md — naming, patterns, structure
    3. Work ONLY in your assigned files
    4. Commit frequently: "Issue #<N>: <specific change>"
    5. Update progress in: .claude/epics/<epic>/updates/<N>/stream-<X>.md
    6. If you need to touch files outside your scope, note it in your progress file and wait
    7. Never use --force on git operations
    
    Complete your stream's work and mark status: completed when done.
```

Streams with unmet dependencies are queued — launch them as their dependencies complete.

**Step 6 — Gate 2: Post-stream review (per stream):**

When a stream agent completes (marks status: completed), run a Codex review of its changes before accepting the stream as done.

```bash
# Capture the stream's diff (from epic branch base to current)
git -C ../epic-<name> diff epic/<name>~<commits>..HEAD -- <stream_file_patterns> > /tmp/stream-<X>-diff.txt
```

```yaml
Task:
  subagent_type: "codex:codex-rescue"
  description: "Gate 2: stream <X> review #<N>"
  prompt: |
    Review the code changes from Stream <X> of Issue #<N>.

    Read:
    - The diff at: /tmp/stream-<X>-diff.txt (or run: git -C ../epic-<name> diff <base>..HEAD -- <stream_files>)
    - Task spec at: .claude/epics/<epic>/<N>.md
    - Context brief at: .claude/epics/<epic>/<N>-context-brief.md
    - Epic architecture at: .claude/epics/<epic>/epic.md

    Check:
    1. Does the implementation satisfy the task's acceptance criteria relevant to this stream?
    2. Does it follow the architecture decisions from the epic?
    3. Does it follow project conventions (naming, patterns, structure)?
    4. Are there any issues with code quality, missing error handling at boundaries, or obvious bugs?
    5. Does it stay within the stream's assigned file scope?

    Write your review to: .claude/epics/<epic>/reviews/<N>-stream-<X>-review.md
    Use this frontmatter:
    ---
    gate: stream-review
    issue: <N>
    stream: <X>
    reviewer: codex
    effort: <your chosen effort level>
    verdict: pass | fail
    reviewed: <current datetime>
    ---

    If pass: summarize what was implemented and confirm alignment.
    If fail: list specific issues that must be fixed, with file paths and line references.

    Do not modify any files other than the review file.
    --wait --effort <chosen based on diff size and task complexity>
```

**If Gate 2 fails**: Present the review findings. Re-launch the stream agent with the specific fixes required. After fixes, re-run Gate 2. If contested, follow the Escalation Protocol.

**Step 7 — Gate 3: Post-issue integration review:**

After ALL streams for an issue have passed Gate 2, run a final integration review of the combined changes.

```yaml
Task:
  subagent_type: "codex:codex-rescue"
  description: "Gate 3: integration review #<N>"
  prompt: |
    Review the combined implementation of Issue #<N> across all streams.

    Read:
    - Combined diff: run git -C ../epic-<name> diff main..HEAD (or the relevant range for this issue's commits)
    - All stream review files in: .claude/epics/<epic>/reviews/<N>-stream-*-review.md
    - Task spec at: .claude/epics/<epic>/<N>.md
    - Context brief at: .claude/epics/<epic>/<N>-context-brief.md
    - Epic architecture at: .claude/epics/<epic>/epic.md

    Check:
    1. Do the streams integrate coherently? No conflicting patterns, duplicate logic, or inconsistent interfaces.
    2. Are shared types, configs, or interfaces consistent across streams?
    3. Does the combined result satisfy ALL acceptance criteria from the task spec?
    4. Are there any integration gaps — things that fall between streams that nobody implemented?

    Write your review to: .claude/epics/<epic>/reviews/<N>-integration-review.md
    Use this frontmatter:
    ---
    gate: integration-review
    issue: <N>
    reviewer: codex
    effort: <your chosen effort level>
    verdict: pass | fail
    reviewed: <current datetime>
    ---

    If pass: confirm the issue is fully implemented and integrated.
    If fail: list specific integration issues with file paths and what needs to change.

    Do not modify any files other than the review file.
    --wait --effort <chosen based on combined diff size>
```

**If Gate 3 fails**: Present findings. Fix integration issues (may require re-launching specific stream agents). Re-run Gate 3. If contested, follow the Escalation Protocol.

**Step 8 — Assign on GitHub:**
```bash
gh issue edit <N> --add-assignee @me --add-label "in-progress"
```

**Step 9 — Create execution status file** at `.claude/epics/<epic>/updates/<N>/execution.md`:
```markdown
## Active Streams
- Stream A: <name> — Started <time>
- Stream B: <name> — Started <time>

## Queued
- Stream C: <name> — Waiting on Stream A

## Completed
(none yet)

## Reviews
- Gate 1 (context brief): pending | pass | fail
- Gate 2 (streams): <stream>: pending | pass | fail
- Gate 3 (integration): pending | pass | fail
```

**Output:**
```
✅ Started work on issue #<N>

Launched N agents:
  Stream A: <name> ✓ Started
  Stream B: <name> ✓ Started
  Stream C: <name> ⏸ Waiting (depends on A)

Review gates active:
  Gate 1 (context brief): ✅ Passed
  Gate 2 (per-stream): will run on stream completion
  Gate 3 (integration): will run after all streams pass

Monitor: check progress in .claude/epics/<epic>/updates/<N>/
Sync updates: "sync issue <N>"
```

---

## Starting a Full Epic

**Trigger**: User wants to launch parallel agents across all ready issues in an epic at once.

### Preflight
- Verify `.claude/epics/<name>/epic.md` exists and has a `github:` field (i.e., it's been synced).
- Check for uncommitted changes: `git status --porcelain` — block if dirty.
- Verify epic branch exists: `git branch -a | grep "epic/<name>"`

### Process

**Step 1 — Read all task files** in `.claude/epics/<name>/`. Parse frontmatter for `status`, `depends_on`, `parallel`.

**Step 2 — Categorize tasks:**
- Ready: status=open, no unmet depends_on
- Blocked: has unmet depends_on
- In Progress: already has an execution file
- Complete: status=closed

**Step 3 — Analyze any ready tasks** that don't have an analysis file yet (run issue analysis inline).

**Step 4 — Launch agents** for all ready tasks following the same per-issue agent launch pattern above.

**Step 5 — Create/update** `.claude/epics/<name>/execution-status.md` with all active agents and queued issues.

**Step 6 — As agents complete**, check if blocked issues are now unblocked. Before launching newly-unblocked tasks, run Gate 4.

**Step 7 — Gate 4: Wave gate review (per dependency transition):**

Before launching a task whose dependencies just completed, verify that the completed work actually provides what the downstream task expects.

```yaml
Task:
  subagent_type: "codex:codex-rescue"
  description: "Gate 4: wave gate #<blocked_N> deps [<completed_N>]"
  prompt: |
    Verify that completed work satisfies the dependency contract for a downstream task.

    Completed task(s):
    - Task #<completed_N>: read spec at .claude/epics/<epic>/<completed_N>.md
    - Integration review at: .claude/epics/<epic>/reviews/<completed_N>-integration-review.md
    - See the actual code changes: git -C ../epic-<name> log --oneline | grep "Issue #<completed_N>"

    Downstream task waiting to launch:
    - Task #<blocked_N>: read spec at .claude/epics/<epic>/<blocked_N>.md
    - Its depends_on includes: [<completed_N>]

    Also read:
    - Epic architecture at: .claude/epics/<epic>/epic.md
    - Context brief (if already generated): .claude/epics/<epic>/<blocked_N>-context-brief.md

    Check:
    1. Does the completed task's output provide the interfaces, data structures, or infrastructure that the downstream task expects?
    2. Are the completed task's APIs, types, or schemas compatible with what the downstream task will consume?
    3. Are there any gaps — things the downstream task assumes exist but were not implemented?
    4. Are there any deviations from the epic's architecture that would cause problems for the downstream task?

    Write your review to: .claude/epics/<epic>/reviews/<blocked_N>-wave-gate-<completed_N>-review.md
    Use this frontmatter:
    ---
    gate: wave-gate
    issue: <blocked_N>
    dependency: <completed_N>
    reviewer: codex
    effort: <your chosen effort level>
    verdict: pass | fail
    reviewed: <current datetime>
    ---

    If pass: confirm the dependency contract is satisfied and the downstream task can proceed.
    If fail: list what's missing or incompatible, with specific details.

    Do not modify any files other than the review file.
    --wait --effort <chosen based on dependency complexity>
```

**If Gate 4 fails**: The completed task may need additional work before the dependent task can start. Present findings. Fix the gap (re-launch the completed task's agent for targeted fixes). Re-run Gate 4. If contested, follow the Escalation Protocol.

After Gate 4 passes, launch the unblocked task following the full per-issue pattern (Steps 2–9 above).

---

## Escalation Protocol

When a Codex review gate returns a `fail` verdict and the orchestrator (or implementing agent) believes the findings are incorrect or the divergence is justified:

1. **Assess**: Read the review findings carefully. Determine if the implementing agent's approach has valid technical reasons for diverging from the spec or architecture.

2. **Contest** (if applicable): If the divergence is justified, prepare a counterargument with specific reasoning — why the implementation is correct despite the review finding.

3. **Present to user**: Show both perspectives clearly:
   ```
   ⚠️ Review gate disagreement on Issue #<N> (<gate type>)
   
   Reviewer finding:
     <specific issue from the review>
   
   Counterargument:
     <why the implementation may be correct>
   
   Options:
     1. Accept — override the review, proceed as-is
     2. Reject — fix the implementation to match the review
     3. Partial — fix specific items: <list which>
   ```

4. **Record decision**: Update the review artifact file:
   - Change verdict from `fail` to `escalated`
   - Append a `## User Decision` section with the user's choice and reasoning

5. **Act on decision**:
   - Accept: proceed to next step, no changes needed
   - Reject: re-launch the implementing agent with specific fix instructions, then re-run the gate
   - Partial: re-launch with targeted fixes only, then re-run the gate

---

## Agent Coordination Rules

When multiple agents work in the same worktree simultaneously:

- Each agent works only on files in its assigned stream scope.
- Agents commit frequently with `Issue #<N>: <description>` format.
- Before modifying a shared file, check `git status <file>` — if another agent has it modified, wait and pull first.
- Agents sync via commits: `git pull --rebase origin epic/<name>` before starting new file work.
- Conflicts are never auto-resolved — agents report them and pause.
- No `--force` flags ever.

Shared files that commonly need coordination (types, config, package.json) should be handled by one designated stream; others pull after that commit.
