---
description: "Refine existing GitHub issues with service context. Analyze quality, add missing sections, and suggest improvements — without changing original content."
allowed-tools: [Read, Bash, Glob, Grep, Agent]
---

# /issue-refine — Issue Refiner

Analyze and strengthen existing issue content. Adds missing service context
while preserving the original author's intent.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output and added content must be in that language.**

## Setup

Read `issue-manager.config.json` from the project root for template sections,
scoring criteria, and project references.

## Input

```
/issue-refine #94 #103 #112         — Refine specific issues
/issue-refine domain:comm            — Audit all issues in a domain
/issue-refine --audit                — Quality audit of all open issues (report only)
/issue-refine --audit domain:data    — Audit specific domain only
/issue-refine                        — Audit all, then suggest which to refine
```

## Core Principle

**Never delete or alter existing content.** Only add missing sections.
Service improvement opinions go in a separate comment, not in the body.

## Execution Flow

### Step 1: Read Config + References

```bash
cat issue-manager.config.json
```

Read all `config.references[]` files for project context.

### Step 2: Score Each Issue

Read the issue body and score against `config.template.sections[]`:

| Section | 0 pts | 1 pt | 2 pts |
|---------|-------|------|-------|
| Background (WHY) | Missing | One-liner | Problem + users + impact |
| User Scenario | Missing | Text only | ASCII UI or step-by-step flow |
| Brand/DNA Check | Missing | Partial | Full checklist |
| Feature Checklist | Missing | List only | Prioritized (🔴🟡🟢) |
| Technical Design | Missing | Components only | Components + API + DB + deps |
| Design Guidelines | Missing | Partial | Tone + density + accessibility |
| Related Issues | Missing | Numbers only | With relationship description |

**Max score**: 2 × (number of configured sections)
**Thresholds**: from `config.scoring.thresholds`

### Step 3: Plan Refinement

For each issue below the "good" threshold, plan what to add:
- Which sections are missing or incomplete?
- What content should be added based on the issue's topic and project context?

**Show the plan to the user before executing:**

```
=== Refinement Plan ===

#94 (3/14 → target 11/14):
  + Add: ## Background — who uses notifications, why in-app matters
  + Add: ## User Scenario — notification types, display locations, read tracking
  + Add: ## Design Guidelines — tone for different user types
  + Comment: Service improvement suggestion about contextual notifications

#103 (5/14 → target 12/14):
  + Add: ## User Scenario — channel × type matrix
  + Add: ## Brand Check — age-inclusive alarm settings

Proceed? [Y/N]
```

### Step 4: Refine (after approval)

For each issue:

1. Read current body: `gh issue view <number> --json body`
2. Insert missing sections at appropriate positions:
   - Background → top of body
   - User Scenario → after existing feature description
   - Technical Design → bottom
3. Update body: `gh issue edit <number> --body "updated body"`
4. Add service improvement comment (separate from body):
   ```bash
   gh issue comment <number> --body "💡 Service improvement suggestion: ..."
   ```

### Step 5: Label Check

Verify each issue has all 3 label axes:
- `type:*` present?
- `scope:*` present?
- `domain:*` present?

Add missing labels. Report mismatched labels to user.

### Step 6: Report

```
=== Refinement Report ===

| Issue | Before | After | Added |
|-------|:------:|:-----:|-------|
| #94   | 3/14   | 11/14 | Background, UX flow, Design guide |
| #103  | 5/14   | 12/14 | User scenario, Brand check |

Labels fixed: #94 (+domain:comm), #103 (OK)
Comments added: #94 (contextual notification suggestion)
```

## --audit Mode (read-only)

Score all issues without modifying anything:

```
=== Issue Quality Audit ===

🔴 Needs major work (below threshold):
  #94  In-app notifications (3/14)
  #95  OCR image upload (2/14)

🟡 Needs some work (below "good"):
  #103 Notification settings (7/14)
  #112 File upload optimization (8/14)

🟢 Good (at or above "good"):
  #166 Customer request 2-Track (13/14)
  #174 Service config management (12/14)

Average: 8.2/14
Issues needing refinement: 15
```

## Batch Safety

When refining 5+ issues, confirm in batches:
"Refining these 5 issues. Proceed?"

## $ARGUMENTS

1. `#number [#number ...]` → Refine specific issues
2. `domain:X` → Audit domain, suggest refinements
3. `--audit` → Read-only quality report
4. `--audit domain:X` → Domain-specific audit
5. No args → Full audit, then suggest top priorities
