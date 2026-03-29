---
description: "Issue cleanup. Detect stale issues, auto-find completed, merge duplicates. Always confirms before closing."
allowed-tools: [Read, Bash, Glob, Grep]
---

# /issue-close — Issue Cleanup

Find and close issues that are stale, completed, or duplicated.
**Never auto-closes** — always shows a list and asks for approval.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output must be in that language.**

## Input

```
/issue-close --stale             — Find stale issues (90+ days inactive)
/issue-close --done              — Find issues completed via merged PRs
/issue-close --merge #133 #219   — Merge duplicate issues
/issue-close #94 #95             — Close specific issues with reason
```

## Principles

1. **Never auto-close** — show list, get approval, then close
2. **Always leave a comment** — why it was closed
3. **Preserve content on merge** — transfer unique content to the surviving issue

## Modes

### --stale (Inactive Issues)

```bash
gh issue list --state open --json number,title,updatedAt,labels
```

**Stale criteria:**
- 90+ days without any update (comments, label changes, body edits)
- Exception: `epic` label → excluded (master epics live through sub-issues)
- Exception: `type:idea` / `type:research` → 180 days (ideas can age)

```
  🕸️ Stale Issues (90+ days inactive)

    #94  In-app notifications (145 days) — domain:comm
    #95  OCR image storage (140 days) — domain:perf
    #111 PostGIS location data (130 days) — domain:location

  Options:
    1. Close all (reason: "Inactive 90+ days. Reopen if needed.")
    2. Review one by one (keep / close / move to Icebox)
    3. Cancel
```

**One-by-one review:**
```
  #94 In-app notifications (145 days, domain:comm)
    Body: "In-app notification display feature" (152 chars)

    1. Close — no longer needed
    2. Icebox — revisit later
    3. Keep — leave open
    4. Refine — run /issue-refine then keep
```

**On close:**
```bash
gh issue comment {number} --body "🧹 Closed: inactive for 90+ days. Reopen if needed."
gh issue close {number}
```

### --done (Completed via PR)

Find issues referenced in merged PRs but still open.

```bash
gh pr list --state merged --limit 50 --json number,title,body \
  --jq '.[].body' | grep -oP '(?i)(close[sd]?|fix(es|ed)?|resolve[sd]?)\s+#\d+'
```

```
  ✅ Completed via PR but still open:

    #275 Region chip UX → PR #278 (merged 3/28)
    #261 Service select-all → PR #270 (merged 3/27)

  → Close these? (Y / select)
```

### --merge (Duplicate Merge)

```
/issue-close --merge #133 #219
```

Compare both issues, merge into one:

```
  🔄 Issue Merge

  #133 User behavior analytics (1,200 chars)
  #219 Service activity logging (2,500 chars)

  Overlap: both about "data collection infrastructure"
    #133 unique: user journey tracking, behavior patterns
    #219 unique: API call logging, search tracking, full logging

  → Proposal: merge #133 into #219 (more detailed, more recent)
  → Proceed? (Y / reverse / cancel)
```

**On Y:**
1. Add #133's unique content to #219 body:
```markdown
## Merged from #133
- User journey tracking: ...
- Behavior pattern analysis: ...
```

2. Comment + close #133:
```bash
gh issue comment 133 --body "🔄 Merged into #219. Unique content transferred."
gh issue close 133
```

3. Comment on #219:
```bash
gh issue comment 219 --body "🔄 Merged content from #133."
```

### Direct Close (#numbers)

```
/issue-close #94 #95
```

```
  Closing:
    #94 In-app notifications
    #95 OCR image storage

  Reason? (Enter for default: "Closed — no longer planned")
  → user types reason or presses Enter

  → Proceed? (Y)
```

## Result Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Cleanup Result

  Closed: 3 (#94, #95, #133)
  Moved to Icebox: 1 (#111)
  Kept: 2 (#103, #146)
  Merged: #133 → #219

  Open issues: 93 (was 96)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## $ARGUMENTS

1. `--stale` → find stale issues (90+ days)
2. `--done` → find PR-completed issues
3. `--merge #A #B` → merge duplicates
4. `#number [#number ...]` → close with reason
5. Combinable: `--stale domain:comm`
