# Conventions — File Formats, Paths & Rules

Read this before doing any file operations across all phases.

---

## Directory Structure

```
.ccpm/
├── prds/
│   └── <feature-name>.md          # Product requirement documents
└── epics/
    ├── <feature-name>/
    │   ├── epic.md                # Technical epic
    │   ├── <N>.md                 # Task files (named by GitHub issue number after sync)
    │   ├── <N>-analysis.md        # Parallel work stream analysis
    │   ├── <N>-context-brief.md   # Compiled context for subagent consumption
    │   ├── github-mapping.md      # Issue number → URL mapping
    │   ├── execution-status.md    # Active agents tracker
    │   ├── reviews/
    │   │   ├── <N>-context-review.md       # Gate 1: context brief review
    │   │   ├── <N>-stream-<X>-review.md    # Gate 2: per-stream review
    │   │   ├── <N>-integration-review.md   # Gate 3: cross-stream review
    │   │   └── <N>-wave-gate-<dep_N>-review.md  # Gate 4: wave gate review
    │   └── updates/
    │       └── <issue_N>/
    │           ├── stream-A.md    # Per-agent progress
    │           ├── progress.md    # Overall issue progress
    │           └── execution.md  # Execution state
    └── archived/
        └── <feature-name>/        # Completed epics
```

Note: `.ccpm/` lives at the project root (sibling to `.claude/`, `src/`, etc.). Artifacts are git-trackable and meant to be committed with the code. The data root was moved out of `.claude/` because Claude Code hard-protects `.claude/` against Write/Edit even under `--dangerously-skip-permissions`.

---

## Frontmatter Schemas

### PRD (.ccpm/prds/<name>.md)
```yaml
---
name: <feature-name>        # kebab-case, matches filename
description: <one-liner>    # used in lists and summaries
status: backlog | active | completed
created: <ISO 8601>         # date -u +"%Y-%m-%dT%H:%M:%SZ"
---
```

### Epic (.ccpm/epics/<name>/epic.md)
```yaml
---
name: <feature-name>
status: backlog | in-progress | completed
created: <ISO 8601>
updated: <ISO 8601>
progress: 0%                # recalculated when tasks close
prd: .ccpm/prds/<name>.md
github: https://github.com/<owner>/<repo>/issues/<N>  # set on sync
---
```

### Task (.ccpm/epics/<name>/<N>.md)
```yaml
---
name: <Task Title>
status: open | in-progress | closed
created: <ISO 8601>
updated: <ISO 8601>
github: https://github.com/<owner>/<repo>/issues/<N>  # set on sync
depends_on: []              # issue numbers this must wait for
parallel: true              # can run concurrently with non-conflicting tasks
conflicts_with: []          # issue numbers that touch the same files
---
```

### Progress (.ccpm/epics/<name>/updates/<N>/progress.md)
```yaml
---
issue: <N>
started: <ISO 8601>
last_sync: <ISO 8601>
completion: 0%
---
```

### Context Brief (.ccpm/epics/<name>/<N>-context-brief.md)
```yaml
---
issue: <N>
compiled: <ISO 8601>
sources:
  prd: .ccpm/prds/<name>.md
  epic: .ccpm/epics/<name>/epic.md
  task: .ccpm/epics/<name>/<N>.md
  analysis: .ccpm/epics/<name>/<N>-analysis.md
---
```

### Review Artifact (.ccpm/epics/<name>/reviews/<N>-*-review.md)
```yaml
---
gate: context-review | stream-review | integration-review | wave-gate
issue: <N>
stream: <X>              # only for stream-review gate
dependency: <dep_N>      # only for wave-gate
reviewer: codex
effort: <level used>     # adaptive per review
verdict: pass | fail | escalated
reviewed: <ISO 8601>
---
```

---

## Datetime Rule

Always get real current datetime from the system — never use placeholder text:
```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

---

## Frontmatter Update Pattern

When updating a single frontmatter field in an existing file:
```bash
sed -i.bak "/^<field>:/c\\<field>: <value>" <file>
rm <file>.bak
```

When stripping frontmatter to get body content for GitHub:
```bash
sed '1,/^---$/d; 1,/^---$/d' <file> > /tmp/body.md
```

---

## GitHub Operations

### Repository Safety Check (run before any write operation)
```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]]; then
  echo "❌ Cannot write to the CCPM template repository."
  echo "Update remote: git remote set-url origin https://github.com/YOUR/REPO.git"
  exit 1
fi
REPO=$(echo "$remote_url" | sed 's|.*github.com[:/]||' | sed 's|\.git$||')
```

### Authentication
Don't pre-check authentication. Run the `gh` command and handle failure:
```bash
gh <command> || echo "❌ GitHub CLI failed. Run: gh auth login"
```

### Getting Issue Numbers
```bash
# From a task file's github field:
grep 'github:' <file> | grep -oE '[0-9]+$'
```

---

## Git / Worktree Conventions

- One branch per epic: `epic/<name>`
- Worktrees live at `../epic-<name>/` (sibling to project root)
- Always start branches from an up-to-date main:
  ```bash
  git checkout main && git pull origin main
  git worktree add ../epic-<name> -b epic/<name>
  ```
- Commit format inside epics: `Issue #<N>: <description>`
- Never use `--force` in any git operation

---

## Naming Conventions

- Feature names: kebab-case, lowercase, letters/numbers/hyphens, starts with a letter
- Task files before sync: `001.md`, `002.md`, ... (sequential)
- Task files after sync: renamed to GitHub issue number (e.g., `1234.md`)
- Labels applied on sync: `epic`, `epic:<name>`, `feature` (for epics); `task`, `epic:<name>` (for tasks)

---

## Epic Progress Calculation

```bash
total=$(ls .ccpm/epics/<name>/[0-9]*.md 2>/dev/null | wc -l)
closed=$(grep -l '^status: closed' .ccpm/epics/<name>/[0-9]*.md 2>/dev/null | wc -l)
progress=$((closed * 100 / total))
```

Update epic frontmatter when any task closes.
