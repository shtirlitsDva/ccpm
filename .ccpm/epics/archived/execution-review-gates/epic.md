---
name: execution-review-gates
status: completed
created: 2026-04-10T19:15:53Z
updated: 2026-04-10T21:07:13Z
progress: 100%
prd: .ccpm/prds/execution-review-gates.md
github: https://github.com/shtirlitsDva/ccpm/issues/1
---

# Epic: execution-review-gates

<overview>
Insert four Codex-powered review gates into the CCPM Execute phase (`references/execute.md`) and update `references/conventions.md` with new file schemas. The gates add blocking quality checks at four points in the execution pipeline:

1. **Gate 1 — Context Brief Review**: Before launching subagents, compile all upstream context (PRD, epic architecture, CLAUDE.md, cross-task deps) into a `<N>-context-brief.md`, then have Codex verify completeness.
2. **Gate 2 — Post-Stream Review**: After each parallel stream completes, Codex reviews the git diff against task spec and architecture.
3. **Gate 3 — Post-Issue Integration Review**: After all streams for an issue pass Gate 2, Codex reviews the combined diff for cross-stream coherence.
4. **Gate 4 — Wave Gate Review**: Before launching dependent tasks, Codex verifies completed work satisfies the dependency contract.

All gates are blocking with an escalation protocol: if the reviewer and implementer disagree, the user decides.
</overview>

<architecture-decisions>
**AD-1: Gates live in execute.md, not a separate file**
The gates are steps in the execution workflow, not a separate system. They insert into the existing "Starting an Issue" and "Starting a Full Epic" sections of `execute.md`. No new phase files.

**AD-2: Codex invoked via codex:codex-rescue subagent**
All reviews use the `/codex-agent` skill pattern — spawning `codex:codex-rescue` Task calls. Reviews are read-only (Codex does not modify code). Effort level is adaptive per review, not hardcoded.

**AD-3: Context brief is a compiled file, not inline prompt expansion**
Instead of bloating the subagent prompt with all context, we compile a `<N>-context-brief.md` file that subagents read. This keeps prompts manageable while ensuring context is available and reviewable.

**AD-4: Review artifacts persist in a reviews/ subdirectory**
All review outputs go to `.ccpm/epics/<epic>/reviews/`. Each review produces a file with consistent frontmatter (gate type, verdict, timestamp). This provides traceability without cluttering the main epic directory.

**AD-5: Escalation is a protocol, not a tool**
When Codex review fails, the orchestrating Claude presents both perspectives (reviewer findings + implementer reasoning) to the user. The user decides. Decision is recorded in the review artifact. No special tooling needed — it's a workflow pattern documented in execute.md.

**AD-6: Enhanced subagent prompt references context brief**
The existing thin subagent prompt in execute.md gets expanded to include instructions to read the context brief, PRD, and epic architecture sections. The prompt template itself stays in execute.md.
</architecture-decisions>

<technical-approach>

<files-modified>
- `references/execute.md` — Main target. Insert context brief generation, all four gate steps, escalation protocol, and enhanced subagent prompt.
- `references/conventions.md` — Add directory structure entries for `reviews/` and `<N>-context-brief.md`. Add frontmatter schemas for context brief and review artifact files.
</files-modified>

<execution-flow>
The modified execution flow for "Starting an Issue" becomes:

```
Step 1 — Read analysis (existing)
Step 2 — Generate context brief (NEW)
  → Compile PRD + epic arch + CLAUDE.md + cross-task context into <N>-context-brief.md
Step 3 — Gate 1: Context brief review (NEW)
  → Codex reviews brief for completeness
  → Block until pass or user override
Step 4 — Create progress tracking (existing, renumbered)
Step 5 — Launch parallel agents with enhanced prompt (MODIFIED)
  → Prompt now includes: read context brief, read PRD, read epic arch
Step 6 — Gate 2: Post-stream review (NEW)
  → When each stream completes, Codex reviews its diff
  → Failed → fix or escalate
Step 7 — Gate 3: Post-issue integration review (NEW)
  → After all streams pass Gate 2, Codex reviews combined diff
  → Failed → fix or escalate
Step 8 — Assign on GitHub (existing, renumbered)
```

For "Starting a Full Epic", add:

```
Gate 4: Wave gate review (NEW)
  → Before launching blocked tasks whose deps just completed
  → Codex verifies completed work provides what downstream expects
  → Failed → fix or escalate
```
</execution-flow>

<context-brief-structure>
```markdown
---
issue: <N>
compiled: <ISO 8601>
sources:
  prd: .ccpm/prds/<name>.md
  epic: .ccpm/epics/<epic>/epic.md
  task: .ccpm/epics/<epic>/<N>.md
  analysis: .ccpm/epics/<epic>/<N>-analysis.md
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
</context-brief-structure>

<review-artifact-structure>
```markdown
---
gate: context-review | stream-review | integration-review | wave-gate
issue: <N>
stream: <X> (for stream reviews)
reviewer: codex
effort: <level used>
verdict: pass | fail | escalated
reviewed: <ISO 8601>
---

# Review: <gate type> for Issue #<N>

## What Was Reviewed
## Findings
## Verdict
## Reasoning
```
</review-artifact-structure>

<escalation-protocol>
1. Codex review returns `fail` with findings
2. Orchestrator reads findings and assesses whether implementing agent's work has valid reasons for divergence
3. If contestable: orchestrator presents both sides to user
4. User decides: `accept` (override, proceed), `reject` (fix required), `partial` (fix specific items)
5. Decision recorded in review artifact: verdict updated to `escalated`, user decision appended
</escalation-protocol>

</technical-approach>

<implementation-strategy>
Two tasks, both modifying skill reference files:

1. **Update execute.md** — Insert context brief generation, all four gates, escalation protocol, and enhanced subagent prompt. This is the bulk of the work.
2. **Update conventions.md** — Add new file types to directory structure, add frontmatter schemas for context brief and review artifact files.

Task 1 must be done before Task 2, since conventions.md schemas should match what execute.md actually produces. Both tasks modify files in `references/` within the CCPM skill directory.
</implementation-strategy>

<task-breakdown-preview>
1. Update execute.md with context brief generation, four review gates, escalation protocol, and enhanced subagent prompt (~L)
2. Update conventions.md with new directory entries and frontmatter schemas (~S)
</task-breakdown-preview>

<dependencies>
- Codex plugin (openai/codex-plugin-cc) must be installed
- `/codex-agent` skill must be available
- `codex:codex-rescue` subagent type accessible
</dependencies>

<success-criteria-technical>
- `execute.md` contains all four gate steps with Codex invocation patterns
- Context brief template compiles PRD + epic + conventions + cross-task context
- Subagent prompt template references context brief
- Escalation protocol documented with user decision recording
- `conventions.md` directory structure includes `reviews/` and `<N>-context-brief.md`
- `conventions.md` has frontmatter schemas for both new file types
- All changes are insertions/modifications to existing files, no new phase files created
</success-criteria-technical>

<estimated-effort>
- Task 1 (execute.md): L — ~3-4 hours, significant content insertion
- Task 2 (conventions.md): S — ~30 min, schema additions
- Total: ~4 hours

<tasks-created>
- [ ] 001.md - Update execute.md with context briefs, four review gates, and enhanced subagent prompt (parallel: true)
- [ ] 002.md - Update conventions.md with review artifact and context brief schemas (parallel: false)

Total tasks: 2
Parallel tasks: 1
Sequential tasks: 1
Estimated total effort: 4 hours
</tasks-created>
</estimated-effort>
