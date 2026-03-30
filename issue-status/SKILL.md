---
description: "Issue status dashboard. Projects-aware analysis with Phase/Priority/Track breakdown, domain progress, stale alerts, weekly changes, and actionable next steps."
allowed-tools: [Read, Bash, Glob, Grep]
---

# /issue-status — Status Dashboard

Analyze and report on the current state of all issues using **both** GitHub Issues and GitHub Projects data.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output must be in that language.**

## Config

Read `issue-manager.config.json` to get:
- `project.owner` — GitHub user/org
- `project.projectNumber` — Projects board number
- `labels.domains[]` — domain list for iteration
- `projectFields` — Phase/Priority/Track field IDs and option IDs

## Input

```
/issue-status                    — Full status report
/issue-status domain:matching    — Domain detail
/issue-status --weekly           — Weekly changes
/issue-status --alerts           — Risk signals only
/issue-status --priority         — Priority-first view (P0 → P1 → P2)
```

## Data Collection

### Step 1: Fetch Projects data (primary source)

This is the **most important step**. Always fetch project items first — they contain Phase, Priority, Track, and Status fields that `gh issue list` cannot provide.

```bash
# Fetch ALL project items with custom field values
# --limit should be high enough to cover all items (check config totalCount)
gh project item-list {projectNumber} --owner {owner} --format json --limit 500
```

This returns each item with: `title`, `status`, `phase`, `priority`, `track`, `labels[]`, `content.number`, `content.url`.

Parse the JSON to build a lookup map: `issueNumber → { status, phase, priority, track }`.

### Step 2: Fetch issue metadata (supplementary)

```bash
# Open issues with dates and labels
gh issue list --state open --json number,title,updatedAt,createdAt,labels --limit 500

# Closed issues count
gh issue list --state closed --json number --jq 'length'

# This week's activity (created or closed in last 7 days)
gh issue list --state all --json number,title,createdAt,closedAt,labels \
  --jq '[.[] | select(.createdAt > "{7_days_ago}" or .closedAt > "{7_days_ago}")]'
```

### Step 3: Cross-reference

Join the two datasets by issue number. Every open issue should have both:
- Issue metadata (title, labels, dates) from `gh issue list`
- Project fields (phase, priority, track, status) from `gh project item-list`

Flag any open issues that are **not in the project** — these are orphaned issues.

---

## Report Sections

### 1. Overview

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Issue Status ({date})

  Total: {open} open / {closed} done ({percent}%)
  This week: +{created} created, {closed_week} completed
  Backlog trend: {trend_direction} ({net_change})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2. Priority Breakdown (from Projects)

Group open issues by Priority field. This is the **most actionable** section.

```
  By Priority:
    🔴 P0 런칭차단    2 issues
       #238 VWorld 중개업소 인증            Phase 3 선행  In Progress
       #166 고객 요청 2-Track              3A: 거래 코어  Done

    🟡 P1 핵심       15 issues
       #285 워크스페이스 연결 통합           3A: 거래 코어  Todo
       #284 통합 채팅 시스템               3A: 거래 코어  Todo
       ...

    🟢 P2 보강        8 issues
       ...

    ⚪ 미배정          5 issues
       #278 활동 지역 Badge 드래그 ...
       ...
```

### 3. Phase Breakdown (from Projects)

Group open issues by Phase field. Shows where the backlog concentrates.

```
  By Phase:
    v1.0 출시       ██░░░░░░░░   3   (P0: 1, P1: 2)
    Phase 3 선행    ████░░░░░░   5   (P0: 1, P1: 3, P2: 1)
    3A: 거래 코어   ████████░░  12   (P1: 8, P2: 4)
    3B: 플랫폼 운영 ██░░░░░░░░   3   (P1: 2, P2: 1)
    Phase 4         ████░░░░░░   4   (P1: 3, P2: 1)
    Phase 5         █░░░░░░░░░   1   (P2: 1)
    Cross-phase     ██░░░░░░░░   2
    Icebox          ░░░░░░░░░░   0
    Backlog         ░░░░░░░░░░   0
```

### 4. Track Distribution (from Projects)

```
  By Track:
    전문가-고객 연결  ████████░░  12
    제품 고도화       ██████░░░░   8
    운영 안정성       ████░░░░░░   5
    수익화           ██░░░░░░░░   2
    배포             █░░░░░░░░░   1
    어드민           █░░░░░░░░░   1
    리서치           █░░░░░░░░░   1
    미래             ░░░░░░░░░░   0
```

### 5. Domain Distribution (from Labels)

```bash
for domain in $(domains from config); do
  gh issue list --state open --label "domain:$domain" --json number --jq 'length'
done
```

```
  By Domain:
    domain:matching  ████████░░  17
    domain:perf      ██████░░░░  13
    ...
```

Note: One issue can have multiple domains, so totals may exceed open count.

### 6. Status Flow (from Projects)

```
  Pipeline:
    Todo          ████████████████  22
    In Progress   ████              5
    Done          ██████████████    17  (this week)
```

### 7. Alerts

Auto-detect from the cross-referenced data:

```
  ⚠️ Alerts:
    🔴 P0 still Todo:
       #{n} {title} — Phase: {phase}, assigned {days} ago
    🟡 In Progress > 7 days without recent commit:
       #{n} {title} — last updated {days}d ago
    🟡 30+ days stale (no update):
       #{n} {title} ({days}d)
    🟡 Open issues not in project (orphaned):
       #{n} {title}
    💡 Phases with no P0/P1:
       Phase 5, Cross-phase (no urgency — expected)
    💡 Tracks with 0 issues:
       미래 (empty — consider archiving or adding ideas)
```

Priority-based alerts (from Projects data):
- **🔴 Critical**: P0 issues in `Todo` status (should be `In Progress`)
- **🔴 Critical**: P0 issues with no assignee
- **🟡 Warning**: `In Progress` issues with no update in 7+ days
- **🟡 Warning**: Open issues not added to the project board
- **🟡 Warning**: Issues with no Priority assigned
- **💡 Info**: Phases/Tracks with zero issues

### 8. Weekly Changes (--weekly)

```bash
gh issue list --state all --json number,title,createdAt,closedAt,labels \
  --jq '[.[] | select(.createdAt > "{7_days_ago}" or .closedAt > "{7_days_ago}")]'
```

Cross-reference with project items to include Phase/Priority in the report.

```
  📅 This week ({date_range})

  ✅ Completed ({count}):
    #{n} {title}  [Phase: {phase}, Priority: {priority}]
    ...

  🆕 New ({count}):
    #{n} {title}  [Phase: {phase}, Priority: {priority}]
    ...

  📈 Trend: {closed} completed vs {created} new
     → Backlog {shrinking|growing|stable} ({net_change})
```

### 9. Domain Detail (domain:X)

When a specific domain is requested, provide deep analysis:

```bash
gh issue list --state open --label "domain:{domain}" --json number,title,updatedAt,labels
# Cross-reference each with project items for phase/priority/track
```

```
  📊 domain:{domain} ({count} issues)
  Master Epic: #{n} {title} (if exists — look for 'epic' label in this domain)

  By Phase (from Projects):
    Phase 3 선행:  ████░░  #{n}, #{n}  [P1, P1]
    3A: 거래 코어: ██████  #{n}, #{n}, #{n}  [P0, P1, P2]
    Phase 4:       ██░░░░  #{n}  [P1]

  By Priority:
    P0: #{n} {title}
    P1: #{n} {title}, #{n} {title}
    P2: #{n} {title}

  Recent activity:
    #{n}: last updated {days} ago
    #{n}: last updated {days} ago

  🎯 Suggested next:
    → #{n} {title}
      Reason: {phase} + {priority} + Status: Todo + no blocking dependencies
```

### 10. Priority View (--priority)

When `--priority` flag is used, reorganize the entire report around priority:

```
  🔴 P0 런칭차단 ({count} issues)
  ─────────────────────────────
  Status    | #    | Title                    | Phase         | Track
  Todo      | #238 | VWorld 중개업소 인증      | Phase 3 선행   | 전문가-고객 연결
  Progress  | #166 | 고객 요청 2-Track         | 3A: 거래 코어  | 제품 고도화

  🟡 P1 핵심 ({count} issues)
  ─────────────────────────────
  ...

  🟢 P2 보강 ({count} issues)
  ─────────────────────────────
  ...
```

---

## Output Rules

1. **Projects data is primary** — Phase/Priority/Track from `gh project item-list` are the authoritative source for prioritization and planning
2. **Labels are secondary** — domain/type/scope from `gh issue list` provide categorization
3. **Always show Priority breakdown** in full reports — this is the most actionable information
4. **Bar charts**: Use block characters (█░) for visual distribution, max 10 blocks
5. **Sorting**: Within each section, sort by Priority (P0 first), then by Phase (earliest first)
6. **Counts must match**: Verify that totals are consistent across sections

## $ARGUMENTS

1. No args → full status report (sections 1-7)
2. `domain:X` → domain detail (section 9)
3. `--weekly` → weekly change report (section 8)
4. `--alerts` → risk signals only (section 7)
5. `--priority` → priority-first view (section 10)
6. Combinable: `--weekly domain:matching`, `--alerts --priority`
