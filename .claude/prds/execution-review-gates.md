---
name: execution-review-gates
description: Add four Codex-powered review gates to the CCPM Execute phase to catch context loss and spec divergence in subagent output
status: backlog
created: 2026-04-10T18:59:43Z
---

# PRD: execution-review-gates

<executive-summary>
CCPM's Execute phase launches parallel subagents to implement GitHub issues, but those agents consistently produce subpar results because they operate in a context vacuum — receiving minimal prompts with file pointers instead of the full architectural and requirements context built up during planning. There are zero automated quality gates: a subagent finishes, marks itself done, and moves on unchecked.

This PRD defines four Codex-powered review gates inserted into the Execute phase that block progression until automated review passes, with user escalation when the implementing agent and reviewer disagree.
</executive-summary>

<problem-statement>
The CCPM workflow produces high-quality artifacts during Plan and Structure phases — PRDs capture requirements precisely, epics document architecture decisions, and task files define clear acceptance criteria. But when execution begins, subagents receive a thin prompt (~10 lines) pointing them at the task file and analysis. They never see:

- The PRD (why the feature exists, user stories, constraints, out-of-scope)
- The Epic's architecture decisions (the most critical section)
- Project conventions (CLAUDE.md, coding patterns, existing architecture)
- Cross-task context (how their output feeds into sibling tasks)
- Design decisions from planning conversations that didn't make it into files

The result: subagents make wrong architecture choices, miss requirements, ignore existing code patterns, and produce output that needs significant manual cleanup. The context built during planning is effectively thrown away at execution time.
</problem-statement>

<user-stories>
**US-1: Context Brief Compilation**
As a developer using CCPM, I want each task's full context (PRD, epic architecture, project conventions, cross-task dependencies) compiled into a single brief before agents start, so that subagents have everything they need to implement correctly.
- Acceptance: A `<N>-context-brief.md` file is generated before any agent launches
- Acceptance: The brief includes PRD excerpts, epic architecture decisions, project conventions, and cross-task context

**US-2: Context Brief Review**
As a developer, I want the compiled context brief reviewed by Codex for completeness before agents consume it, so that missing context is caught before it causes bad implementations.
- Acceptance: Codex reviews the brief and either approves or flags gaps
- Acceptance: Gaps block agent launch until resolved

**US-3: Post-Stream Review**
As a developer, I want each completed parallel stream reviewed by Codex against the task spec and architecture, so that divergence is caught at the smallest unit of work.
- Acceptance: After a stream completes, Codex reviews the git diff
- Acceptance: Review checks: spec adherence, architecture consistency, code pattern conformance
- Acceptance: Failed review blocks the stream from being marked complete

**US-4: Post-Issue Integration Review**
As a developer, I want all streams for an issue reviewed together after completion, so that cross-stream integration issues are caught before moving on.
- Acceptance: After all streams for an issue complete, Codex reviews the combined diff
- Acceptance: Review checks: cross-stream coherence, no conflicting patterns, integration correctness

**US-5: Wave Gate Review**
As a developer, I want Codex to verify that completed work actually provides what downstream dependent tasks expect, before those tasks launch.
- Acceptance: Before launching a blocked task whose dependencies just completed, Codex reviews whether the completed work satisfies the dependency contract
- Acceptance: Failed gate blocks downstream launch until resolved

**US-6: Escalation on Disagreement**
As a developer, I want to be the final arbiter when the implementing agent and the Codex reviewer disagree, so that false positives don't block progress and real issues don't slip through.
- Acceptance: If a review fails, the implementing agent can contest with reasoning
- Acceptance: Unresolved disagreements escalate to the user with both perspectives presented
- Acceptance: User decision is final and recorded in the review artifact
</user-stories>

<functional-requirements>
**FR-1: Context Brief Generator**
- Reads the PRD, epic, task file, analysis file, CLAUDE.md, and other task files in the same epic
- Compiles a single `<N>-context-brief.md` with sections: Requirements Context, Architecture Decisions, Project Conventions, Cross-Task Dependencies, Key Constraints
- Stored at `.claude/epics/<epic>/<N>-context-brief.md`

**FR-2: Pre-Execution Review Gate (Gate 1)**
- Launches Codex via `codex:codex-rescue` to review the context brief
- Codex checks: completeness against PRD, architecture alignment with epic, no contradictions
- Produces `.claude/epics/<epic>/reviews/<N>-context-review.md`
- Blocking: agents cannot launch until review passes

**FR-3: Post-Stream Review Gate (Gate 2)**
- Triggered when a subagent stream marks itself complete
- Captures the stream's git diff
- Launches Codex to review diff against: task requirements, architecture decisions, project conventions
- Produces `.claude/epics/<epic>/reviews/<N>-stream-<X>-review.md`
- Blocking: stream not marked complete until review passes
- Effort level chosen by the reviewing agent based on diff size and task complexity

**FR-4: Post-Issue Integration Review Gate (Gate 3)**
- Triggered when all streams for an issue are reviewed and passed
- Captures combined diff for the issue
- Launches Codex to review: cross-stream coherence, integration correctness, no conflicting patterns
- Produces `.claude/epics/<epic>/reviews/<N>-integration-review.md`
- Blocking: issue not closed until review passes

**FR-5: Wave Gate Review (Gate 4)**
- Triggered before launching tasks that depend on just-completed tasks
- Codex reads the completed task's output and the dependent task's requirements
- Checks: does the completed work provide what the dependent task expects?
- Produces `.claude/epics/<epic>/reviews/<N>-wave-gate-<dep_N>.md`
- Blocking: dependent task does not launch until review passes

**FR-6: Escalation Protocol**
- If Codex review fails, the main orchestrator presents the review findings
- Implementing agent (or orchestrator) can contest with specific reasoning
- If contested, both perspectives are presented to the user
- User decides: accept (override review), reject (fix required), or partial (fix specific items)
- Decision recorded in the review artifact file

**FR-7: Review Artifact Schema**
- All review files follow a consistent frontmatter schema
- Include: gate type, issue number, reviewer (codex), verdict (pass/fail/escalated), timestamp, effort level used
- Body includes: what was reviewed, findings, verdict reasoning

**FR-8: Enhanced Subagent Prompt**
- Subagent prompts expanded to include instruction to read the context brief
- Explicit instruction to read PRD and epic architecture sections
- Reference to project conventions
</functional-requirements>

<non-functional-requirements>
- **Cost efficiency**: No redundant reviews. A stream that passed review and hasn't changed is not re-reviewed.
- **Traceability**: Every review produces a persistent artifact file. No fire-and-forget reviews.
- **Non-destructive reviews**: Codex review agents do not modify code. They review and report only.
- **Adaptive effort**: Codex effort level (`--effort`) is chosen by the reviewing agent based on task complexity and diff size, not hardcoded.
- **Workflow continuity**: Existing CCPM file conventions, directory structure, and frontmatter schemas are preserved. Changes are insertions into the Execute phase, not rewrites.
</non-functional-requirements>

<success-criteria>
- All four gates are implemented and documented in `execute.md`
- Context briefs are generated before every agent launch
- Every stream, issue, and wave transition passes through a Codex review
- Disagreements escalate to user with clear presentation of both sides
- Review artifacts are persisted and traceable
- Subagent prompts include context brief and architecture references
</success-criteria>

<constraints-and-assumptions>
- Reviews use the `/codex-agent` skill pattern (spawning `codex:codex-rescue` subagents)
- Reviews can run `--background` or `--wait` depending on review size
- Codex is assumed to be installed and authenticated (pre-existing `/codex:setup`)
- The PRD, epic, and task files produced by upstream phases are assumed to be high quality (those phases are out of scope)
- Review gates add latency to the execution phase — this is an acceptable tradeoff for quality
</constraints-and-assumptions>

<out-of-scope>
- Changes to the Plan phase (PRD/Epic creation) — working well
- Changes to the Structure phase (task decomposition) — working well
- Changes to the Sync phase (GitHub push) — mechanical, no quality issue
- Changes to the Track phase (status scripts) — reporting, not execution
- Replacing revdiff — stays as manual review tool, orthogonal to automated gates
- Codex reviewing PRDs or epics during planning — out of scope for this iteration
</out-of-scope>

<dependencies>
- `/codex-agent` skill must be functional (codex-cli installed, authenticated)
- `codex:codex-rescue` subagent type must be available
- Existing Execute phase workflow in `references/execute.md`
- Existing file conventions in `references/conventions.md`
</dependencies>
