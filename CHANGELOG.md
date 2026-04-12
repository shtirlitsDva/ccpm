# CCPM Changelog

## [2026-04-11] - Runtime Artifact Root Rename (Breaking)

### 💥 Breaking Change
- **Runtime data root moved from `.claude/` to `.ccpm/`**
  - All PRDs, epics, task files, analyses, context briefs, review artifacts, progress files, and archived epics now live under `.ccpm/` at the project root (sibling to `.claude/`).
  - **Why**: Claude Code hard-protects `.claude/` against Write/Edit operations even under `--dangerously-skip-permissions`. Only `.claude/commands/`, `.claude/agents/`, and `.claude/skills/` are carved out. Every other write under `.claude/` triggers an approval prompt per edit, making bypass mode effectively unusable for CCPM's artifact-heavy workflows (review gates, parallel worker updates, frontmatter updates).
  - References: anthropics/claude-code#37029, anthropics/claude-code#36192, anthropics/claude-code#36168.

### 🔄 Migration
- **Automatic on next `init.sh` run.** The initializer detects legacy `.claude/{prds,epics}/` trees and:
  - Moves them to `.ccpm/{prds,epics}/` using `git mv` (preserving history) when git-tracked, or `cp -rn` + cleanup when untracked.
  - Rewrites embedded `.claude/{prds,epics}/` paths inside all migrated `*.md` files (frontmatter `prd:`, `epic:`, `task:`, `analysis:` fields plus any prose/shell snippets).
  - Writes `.ccpm/.migration-complete` sentinel so re-runs are no-ops.
- No user action required beyond running the existing init script.

### 🧹 Cleanup
- Removed dead legacy `mkdir` lines in `init.sh` for `.claude/rules`, `.claude/agents`, and `.claude/scripts/pm` (v1 leftovers from before CCPM was a skill — no code referenced them).
- Removed the unreferenced `context/` entry from the directory-tree diagram in `conventions.md`. CCPM never managed that subtree; users who maintain a separate `.claude/context/` system are unaffected.
- Removed `.claude/rules` existence check from `validate.sh`.

### 📝 Documentation
- Updated all skill reference docs (`conventions.md`, `plan.md`, `structure.md`, `sync.md`, `execute.md`, `track.md`) to reference `.ccpm/`.
- Updated all 14 tracking scripts to read from `.ccpm/`.
- Updated `README.md` with new directory tree and rationale note.
- Preserved: `.claude/skills/ccpm` install path in `README.md` (Claude Code's own skill-install directory, unrelated to runtime artifacts) and `~/.claude` plugin-cache references in `revdiff-review.md`.

---

## [2025-01-24] - Major Cleanup & Issue Resolution Release

### 🎯 Overview
Resolved 10 of 12 open GitHub issues, modernized command syntax, improved documentation, and enhanced system accuracy. This release focuses on stability, usability, and addressing community feedback.

### ✨ Added
- **Local Mode Support** ([#201](https://github.com/automazeio/ccpm/issues/201))
  - Created `LOCAL_MODE.md` with comprehensive offline workflow guide
  - All core commands (prd-new, prd-parse, epic-decompose) work without GitHub
  - Clear distinction between local-only vs GitHub-dependent commands

- **Automatic GitHub Label Creation** ([#544](https://github.com/automazeio/ccpm/issues/544))
  - Enhanced `init.sh` to automatically create `epic` and `task` labels
  - Proper colors: `epic` (green #0E8A16), `task` (blue #1D76DB)  
  - Eliminates manual label setup during project initialization

- **Context Creation Accuracy Safeguards** ([#48](https://github.com/automazeio/ccpm/issues/48))
  - Added mandatory self-verification checkpoints in context commands
  - Implemented evidence-based analysis requirements
  - Added uncertainty flagging with `⚠️ Assumption - requires verification`
  - Enhanced both `/context:create` and `/context:update` with accuracy validation

### 🔄 Changed
- **Modernized Command Syntax** ([#531](https://github.com/automazeio/ccpm/issues/531))
  - Updated 14 PM command files to use concise `!bash` execution pattern
  - Simplified `allowed-tools` frontmatter declarations
  - Reduced token usage and improved Claude Code compatibility

- **Comprehensive README Overhaul** ([#323](https://github.com/automazeio/ccpm/issues/323))
  - Clarified PRD vs Epic terminology and definitions
  - Streamlined workflow explanations and removed redundant sections
  - Fixed installation instructions and troubleshooting guidance
  - Improved overall structure and navigation

### 📋 Research & Community Engagement
- **Multi-Tracker Support Analysis** ([#200](https://github.com/automazeio/ccpm/issues/200))
  - Researched CLI availability for Linear, Trello, Azure DevOps, Jira
  - Identified Linear as best first alternative to GitHub Issues
  - Provided detailed implementation roadmap for future development

- **GitLab Support Research** ([#588](https://github.com/automazeio/ccpm/issues/588))  
  - Confirmed strong `glab` CLI support for GitLab integration
  - Invited community contributor to submit existing GitLab implementation as PR
  - Updated project roadmap to include GitLab as priority platform

### 🐛 Clarified Platform Limitations
- **Windows Shell Compatibility** ([#609](https://github.com/automazeio/ccpm/issues/609))
  - Documented as Claude Code platform limitation (requires POSIX shell)
  - Provided workarounds and alternative solutions

- **Codex CLI Integration** ([#585](https://github.com/automazeio/ccpm/issues/585))
  - Explained future multi-AI provider support in new CLI architecture

- **Parallel Worker Agent Behavior** ([#530](https://github.com/automazeio/ccpm/issues/530))
  - Clarified agent role as coordinator, not direct coder
  - Provided implementation guidance and workarounds

### 🔒 Security
- **Privacy Documentation Fix** ([#630](https://github.com/automazeio/ccpm/issues/630))
  - Verified resolution via PR #631 (remove real repository references)

### 💡 Proposed Features
- **Bug Handling Workflow** ([#654](https://github.com/automazeio/ccpm/issues/654))
  - Designed `/pm:attach-bug` command for automated bug tracking
  - Proposed lightweight sub-issue integration with existing infrastructure
  - Community feedback requested on implementation approach

### 📊 Issues Resolved
**Closed**: 10 issues  
**Active Proposals**: 1 issue (#654)  
**Remaining Open**: 1 issue (#653)

#### Closed Issues:
- [#630](https://github.com/automazeio/ccpm/issues/630) - Privacy: Remove real repo references ✅  
- [#609](https://github.com/automazeio/ccpm/issues/609) - Windows shell error (platform limitation) ✅
- [#585](https://github.com/automazeio/ccpm/issues/585) - Codex CLI compatibility (architecture update) ✅  
- [#571](https://github.com/automazeio/ccpm/issues/571) - Figma MCP support (platform feature) ✅
- [#531](https://github.com/automazeio/ccpm/issues/531) - Use !bash in custom slash commands ✅
- [#323](https://github.com/automazeio/ccpm/issues/323) - Improve README.md ✅
- [#201](https://github.com/automazeio/ccpm/issues/201) - Local-only mode support ✅
- [#200](https://github.com/automazeio/ccpm/issues/200) - Multi-tracker support research ✅  
- [#588](https://github.com/automazeio/ccpm/issues/588) - GitLab support research ✅
- [#48](https://github.com/automazeio/ccpm/issues/48) - Context creation inaccuracies ✅
- [#530](https://github.com/automazeio/ccpm/issues/530) - Parallel worker coding operations ✅
- [#544](https://github.com/automazeio/ccpm/issues/544) - Auto-create labels during init ✅
- [#947](https://github.com/automazeio/ccpm/issues/947) - Project roadmap update ✅

### 🛠️ Technical Details
- **Files Modified**: 16 core files + documentation
- **New Files**: `LOCAL_MODE.md`, `CONTEXT_ACCURACY.md`  
- **Commands Updated**: All 14 PM slash commands modernized
- **Backward Compatibility**: Fully maintained
- **Dependencies**: No new external dependencies added

### 🏗️ Project Health
- **Issue Resolution Rate**: 83% (10/12 issues closed)
- **Documentation Coverage**: Significantly improved
- **Community Engagement**: Active contributor invitation and feedback solicitation
- **Code Quality**: Enhanced accuracy safeguards and validation

### 🚀 Next Steps
1. Community feedback on bug handling proposal (#654)
2. GitLab integration PR review and merge
3. Linear platform integration (pending demand)
4. Enhanced testing and validation workflows

---

*This release represents a major stability and usability milestone for CCPM, addressing the majority of outstanding community issues while establishing a foundation for future multi-platform support.*