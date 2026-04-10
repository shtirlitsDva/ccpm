# Revdiff Document Review

Launch revdiff to let the user review and annotate a document inline, then process their feedback. This replaces dumping text to the terminal and asking "how does this look?"

**Prerequisite**: the `revdiff` plugin must be installed (`/plugin install revdiff@shtirlitsDva-revdiff` or equivalent). `revdiff` binary must be on PATH.

---

## Review Loop

After writing a document (PRD, epic, task file, analysis), launch revdiff for inline review:

### Step 1 — Find the launcher

Locate the revdiff launcher script from the installed plugin:

**POSIX (macOS/Linux):**
```bash
LAUNCHER=$(find ~/.claude -name 'launch-revdiff.sh' -path '*/cache/*' 2>/dev/null | head -1)
if [ -z "$LAUNCHER" ]; then
  echo "error: revdiff plugin not installed — run: /plugin install revdiff@shtirlitsDva-revdiff"
  exit 1
fi
```

**Windows (PowerShell):**
```powershell
$launcher = Get-ChildItem -Path "$env:USERPROFILE\.claude" -Recurse -Filter 'launch-revdiff.ps1' -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match 'cache' } |
  Select-Object -First 1
if ($null -eq $launcher) {
  Write-Error "revdiff plugin not installed — run: /plugin install revdiff@shtirlitsDva-revdiff"
  exit 1
}
```

### Step 2 — Launch review

**POSIX:**
```bash
ANNOTATIONS=$(bash "$LAUNCHER" --only="<absolute-path-to-file>" --wrap)
```

**Windows:**
```powershell
$annotations = & $launcher.FullName --only="<absolute-path-to-file>" --wrap
```

The launcher opens revdiff in a terminal overlay (tmux/kitty/wezterm split pane), blocks until the user quits, and prints captured annotations to stdout.

### Step 3 — Process annotations

If the launcher produces output, the user made annotations. The format is:

```
## filename:line ( )
annotation text here

## filename:line ( )
another annotation
```

Each annotation references a specific line in the document. Read each one, apply the requested change to the file, and re-launch revdiff (go back to Step 2).

### Step 4 — Done

When the launcher produces **no output**, the user quit without annotations — the document is approved. Proceed to the next phase.

---

## Integration Pattern

In phase reference files (plan.md, structure.md, execute.md), add this step after writing each document:

```
**After writing**: Launch revdiff for inline review.
Read `references/revdiff-review.md` and follow the review loop with the document file.
Process all annotations, update the document, and re-launch until the user quits without annotations.
```

Do NOT ask "how does this look?" or dump the document to the terminal. The revdiff review IS the approval step.
