---
description: "Organize GitHub issues — cluster by domain, create master epics, detect duplicates, map dependencies, find orphaned issues, and triage Projects fields (Phase/Priority/Track/Status)."
allowed-tools: [Read, Bash, Glob, Grep, Agent]
---

# /issue-organize — Issue Organizer

Analyze issue structure, find gaps, detect duplicates, and create master epics
to give the backlog a clear hierarchy.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output (reports, epic bodies, comments) must be in that language.**

## Setup

Read `issue-manager.config.json` from the project root for domain labels and project fields.

## Input

```
/issue-organize                              — Full structure report
/issue-organize domain:location              — Cluster analysis for a domain
/issue-organize --find-orphans               — Issues not referenced by any epic
/issue-organize --find-duplicates            — Detect similar/duplicate issues
/issue-organize --create-master domain:comm  — Create a master epic for a domain
/issue-organize --triage                     — Triage Projects fields (Phase/Priority/Track/Status)
/issue-organize --triage --auto              — Triage with auto-suggestion (still confirms before applying)
```

## Modes

### Mode 1: Full Structure Report (default)

Scan all open issues and produce:

```
=== Issue Structure Report ===

📊 Domain Distribution:
  domain:matching  17  (Master: #236 Hyper-Fit)
  domain:perf      13  (Master: #204 Performance)
  domain:data      12  (Master: ❌ none)
  domain:comm      11  (Master: ❌ none)
  ...

🔗 Master Epic Status:
  ✅ Has master: matching(#236), perf(#204), ux(#197)
  ❌ No master: data, comm, admin, location

🏚️ Orphan issues (not in any epic): N
🔄 Duplicate suspects: N pairs
📌 Missing dependencies: N
```

### Mode 2: Domain Cluster Analysis

For a specific domain, analyze all its issues:

1. **Group** issues by sub-topic
2. **Map dependencies** — what order should they be done?
3. **Suggest master epic** — if none exists

```
=== domain:comm Cluster Analysis ===

Group A — Customer Communication Channels
  #141 Kakao channel integration
  #167 Video/voice consultation
  #139 Call recording → AI summary
  #98  Safe number API

Group B — Notifications
  #103 Notification settings UI
  #94  In-app announcements
  #240 Service-ready notifications

Dependency order:
  #98 → #141 → #167 → #139
  #94 ─── independent
  #103 → #240

💡 Suggested master epic: "Communication Infrastructure"
  → Create? [Y/N]
```

### Mode 3: Find Orphans (--find-orphans)

1. Collect all issues with `epic` label
2. Extract `#number` references from their bodies
3. Find open issues not referenced anywhere

```
=== Orphan Issues ===

domain:data (5 orphans):
  #133 User behavior analytics
  #155 Activity counter system
  #161 Click tracking
  #219 Service activity logging
  #131 Post-launch data collection

💡 Suggestion: Create domain:data master epic
```

### Mode 4: Find Duplicates (--find-duplicates)

Compare issues by:
- Title keyword overlap (>60%)
- Same domain + similar feature description
- Same DB tables or API endpoints mentioned

```
=== Duplicate Suspects ===

🔴 High (consider merging):
  #133 User behavior analytics ↔ #219 Service activity logging
    → Both about "data collection infrastructure". Could merge.

🟡 Medium (link as related):
  #149 Admin cron dashboard ↔ #245 External data sync control tower
    → Both "admin monitoring". Good candidates for master epic.

🟢 Low (just note):
  #111 PostGIS location data ↔ #256 Location-based auto-suggest
    → Tech enabler vs feature. Link as dependency only.
```

### Mode 5: Create Master Epic (--create-master)

Analyze the domain, then generate a master epic:

**Master Epic Template:**

```markdown
## Overview
[Domain goal and project alignment]

## Sub-issues

### Group A — [sub-topic]
- [ ] #number title
- [ ] #number title

### Group B — [sub-topic]
- [ ] #number title

## Execution Order (dependency-based)
#A → #B → #C (independent: #D, #E)

## Success Metrics
- [ ] [measurable goal]

## Related Masters
- #other-epic (overlap area)
```

**After creating:**
1. Add `epic` label
2. Add to Projects with Phase + Priority
3. Comment on each sub-issue: "Parent epic: #master-number"

**Always confirm before creating:**
```
=== Master Epic Draft ===

Title: "epic: Communication Infrastructure — channels + notifications"
Labels: type:feature, scope:both, domain:comm, epic
Sub-issues: #94, #98, #103, #139, #141, #167, #240

Create this epic? [Y/N]
```

### Mode 6: Triage Projects Fields (--triage)

Scan all project items, find issues with missing or inconsistent Projects fields,
suggest values, and apply after user confirmation.

**This mode uses `gh project item-list` and `gh project item-edit` to manage Projects fields.**

#### Step 1: Fetch and Analyze

```bash
# Read config for project info and field IDs
cat issue-manager.config.json

# Fetch all project items
gh project item-list {projectNumber} --owner {owner} --format json --limit 500
```

From the config, extract:
- `project.projectNumber` and `project.owner` — for gh project commands
- `project.projectId` — for item-edit
- `projectFields.phase.fieldId` and `.options` — Phase field ID and option IDs
- `projectFields.priority.fieldId` and `.options` — Priority field ID and option IDs
- `projectFields.track.fieldId` and `.options` — Track field ID and option IDs

#### Step 2: Detect Issues

Categorize problems into 4 types:

```
=== Projects Field Triage ===

🔴 Status Mismatch (issues where Status doesn't match reality):
   #{n} {title}
     Status: Todo → Should be: Done
     Reason: Issue is closed / has merged PR / implementation commit found
     Fix: gh project item-edit ...

🟡 Priority Missing ({count} issues):
   #{n} {title} — Phase: {phase}, Domain: {domain}
   #{n} {title} — Phase: -, Domain: {domain}
   ...

🟡 Phase Missing ({count} issues):
   #{n} {title} — Priority: {priority}, Domain: {domain}
   ...

🟡 Track Missing ({count} issues):
   #{n} {title} — Phase: {phase}, Priority: {priority}
   ...

🔵 Priority-Phase Mismatch (contradictions):
   #{n} {title}
     Priority: P0 런칭차단, Phase: Phase 5
     → Suggestion: Downgrade to P1 핵심 (Phase 5 = future, not launch-blocking)
```

#### Step 3: Suggest Values (--auto flag)

When `--auto` flag is used, analyze each issue's title, labels, domain, and body
to suggest appropriate field values:

**Suggestion Heuristics:**

Phase suggestion (based on labels and content):
- Has `Phase X` in title → that Phase
- Domain dependencies suggest ordering (e.g., location before matching)
- Sub-issue of an epic → inherit epic's Phase
- No signals → suggest `Backlog`

Priority suggestion:
- `type:bug` → at least `P1 핵심`
- Sub-issue of P0 epic → `P0 런칭차단`
- `type:idea` or Icebox Phase → `P2 보강`
- `type:feature` in current Phase → `P1 핵심`
- No signals → `P2 보강`

Track suggestion (based on domain label):
- `domain:matching` → `전문가-고객 연결`
- `domain:expert` → `전문가-고객 연결`
- `domain:perf` → `운영 안정성`
- `domain:admin` → `어드민`
- `domain:payment` → `수익화`
- `domain:ux` → `제품 고도화`
- `domain:data` → `리서치`
- `domain:location` → `제품 고도화`
- `domain:comm` → `전문가-고객 연결`
- `domain:legal` → `운영 안정성`
- Multiple domains / unclear → leave for user to decide

Status correction:
- Issue state is `closed` but project Status is `Todo`/`In Progress` → `Done`
- Issue has merged PR (check linked PRs) → `Done`
- Issue is referenced in recent commits with "fix:", "feat:", "close" → suggest `Done`

```
=== Triage Suggestions ===

🔴 Status Fix (auto-applicable):
  #{n} {title}
    Status: Todo → Done  (issue is closed)
    Apply? [Y/n]

🟡 Field Suggestions:
  #{n} {title}
    Phase: (none) → Phase 3 선행  (sub-issue of #236, Phase 4 epic — suggest earlier phase)
    Priority: (none) → P1 핵심  (type:feature in active phase)
    Track: (none) → 전문가-고객 연결  (domain:matching)
    Apply all? [Y/n/edit]

  #{n} {title}
    Phase: (none) → Backlog  (no phase signals found)
    Priority: (none) → P2 보강  (type:feature, no urgency signals)
    Track: (none) → 제품 고도화  (domain:ux)
    Apply all? [Y/n/edit]

🔵 Contradiction Fix:
  #{n} {title}
    Priority: P0 런칭차단 → P1 핵심  (Phase 5 is future, not launch-blocking)
    Apply? [Y/n]
```

#### Step 4: Apply Changes

After user confirms each suggestion (or batch confirms), apply using:

```bash
# Get the item ID for the issue in the project
# Item IDs are available from gh project item-list output

# Set Phase
gh project item-edit \
  --project-id {projectId} \
  --id {itemId} \
  --field-id {phase.fieldId} \
  --single-select-option-id {phase.options[selectedPhase]}

# Set Priority
gh project item-edit \
  --project-id {projectId} \
  --id {itemId} \
  --field-id {priority.fieldId} \
  --single-select-option-id {priority.options[selectedPriority]}

# Set Track
gh project item-edit \
  --project-id {projectId} \
  --id {itemId} \
  --field-id {track.fieldId} \
  --single-select-option-id {track.options[selectedTrack]}

# Set Status (Status field uses different ID format — from gh project field-list)
gh project item-edit \
  --project-id {projectId} \
  --id {itemId} \
  --field-id {status.fieldId} \
  --single-select-option-id {status.options[selectedStatus]}
```

**Important**: The `--id` parameter is the **project item ID** (e.g., `PVTI_xxx`), NOT the issue number.
Get item IDs from the `gh project item-list` output: each item has an `id` field.

#### Step 5: Summary

After applying, show a summary:

```
=== Triage Complete ===

  Applied:
    ✅ #{n} Status: Todo → Done
    ✅ #{n} Phase: → Phase 3 선행, Priority: → P1 핵심, Track: → 전문가-고객 연결
    ✅ #{n} Priority: P0 → P1 핵심

  Skipped (user declined):
    ⏭️ #{n} Track suggestion declined

  Remaining unresolved:
    ❓ #{n} — could not determine Phase (manual review needed)

  💡 Run /issue-status to verify the updated dashboard
```

#### Adding New Issues to Project

If `--triage` finds open issues that are **not in the project board at all**, add them first:

```bash
# Add issue to project
gh project item-add {projectNumber} --owner {owner} --url https://github.com/{owner}/{repo}/issues/{number}
```

Then proceed with field assignment as above.

## Organization Principles

1. **Never merge** — preserve individual issue context. Reference from master only.
2. **Only close true duplicates** — same thing said differently. Add `Superseded by #X` comment.
3. **Master epics are "table of contents"** — implementation details stay in sub-issues.
4. **Confirm before modifying** — any close, create, or label change needs user approval.
5. **Respect existing masters** — if a domain already has an epic, only add missing sub-issue links.

## Existing Master Epics

Before creating, check if the domain already has a master:
```bash
gh issue list --label "epic,domain:X" --state open --json number,title
```

If found, suggest adding orphan issues to the existing master instead of creating a new one.

## $ARGUMENTS

1. No args → Full structure report
2. `domain:X` → Domain cluster analysis + suggestions
3. `--find-orphans` → Orphan issue detection
4. `--find-duplicates` → Duplicate/similar issue detection
5. `--create-master domain:X` → Create master epic (analyze → draft → confirm → create)
6. `--triage` → Scan and fix missing/inconsistent Projects fields (Phase/Priority/Track/Status)
7. `--triage --auto` → Triage with auto-suggestion based on labels, domain, and content analysis
8. Combinable: `--find-orphans domain:data` → Orphans in data domain only
9. Combinable: `--triage domain:matching` → Triage only issues in matching domain
