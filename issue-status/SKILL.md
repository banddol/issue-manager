---
description: "Issue status dashboard. Domain progress, stale alerts, weekly changes, risk signals at a glance."
allowed-tools: [Read, Bash, Glob, Grep]
---

# /issue-status — Status Dashboard

Analyze and report on the current state of all issues.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output must be in that language.**

## Input

```
/issue-status                    — Full status report
/issue-status domain:matching    — Domain detail
/issue-status --weekly           — Weekly changes
/issue-status --alerts           — Risk signals only
```

## Report Sections

### 1. Overview

```bash
gh issue list --state open --json number --jq 'length'
gh issue list --state closed --json number --jq 'length'
gh issue list --state all --json number,createdAt,closedAt
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Issue Status ({date})

  Total: {open} open / {closed} done
  This week: +{created} created, {closed_week} completed
  Completion rate: {percent}%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2. Domain Distribution

```bash
for domain in $(domains from config); do
  gh issue list --state open --label "domain:$domain" --json number --jq 'length'
done
```

```
  By domain:
    domain:matching  ████████░░  17
    domain:perf      ██████░░░░  13
    domain:data      ██████░░░░  12
    ...
```

### 3. Alerts

Auto-detect and warn:

```bash
# Stale issues (30+ days without update)
gh issue list --state open --json number,title,updatedAt

# P0 still in Todo (from Projects)
# In Progress without linked PR
```

```
  ⚠️ Alerts:
    🔴 P0 still Todo: #238 ...
    🟡 30+ days stale: #94 (45d), #95 (40d)
    🟡 In Progress, no PR: #166 ...
    💡 Domains without master epic: data, comm, admin
```

### 4. Weekly Changes (--weekly)

```bash
gh issue list --state all --json number,title,createdAt,closedAt,labels \
  --jq '.[] | select(.createdAt > "{7_days_ago}" or .closedAt > "{7_days_ago}")'
```

```
  📅 This week ({date_range})

  ✅ Completed:
    #275 ...
    #261 ...

  🆕 New:
    #274 ...
    #276 ...

  📈 Trend: {closed} completed > {created} new (backlog shrinking)
```

### 5. Domain Detail (domain:X)

```
/issue-status domain:matching

  📊 domain:matching (17 issues)
  Master Epic: #236 Hyper-Fit Pipeline

  By phase:
    Phase 3:  ████░░  #166, #170, #171, #177
    Phase 4:  ██████  #163, #218, #236, #239

  Recent activity:
    #166: last updated 3 days ago
    #236: last updated 5 days ago

  Next priority suggestion:
    → #166 (Phase 3 + P1 + no dependencies → ready to start)
```

## $ARGUMENTS

1. No args → full status report
2. `domain:X` → domain detail
3. `--weekly` → weekly change report
4. `--alerts` → risk signals only
5. Combinable: `--weekly domain:matching`
