---
description: "Organize GitHub issues — cluster by domain, create master epics, detect duplicates, map dependencies, find orphaned issues."
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
6. Combinable: `--find-orphans domain:data` → Orphans in data domain only
