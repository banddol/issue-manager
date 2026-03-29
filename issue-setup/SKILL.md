---
description: "Interactive setup wizard for issue-manager. Auto-detects project structure, learns project context, proposes optimal config. Most steps just need Y."
allowed-tools: [Read, Write, Bash, Glob, Grep, Agent]
---

# /issue-setup — Setup Wizard

Analyze the project and generate `issue-manager.config.json` through an interactive wizard.
Auto-detect everything possible, propose optimal settings, user just approves.

## Core Pattern: "Propose → Approve"

Every step:
1. **Auto-analyze** to determine the best value
2. **Show proposal** with a default `(Y)`
3. User presses **Y/Enter to proceed**, or types a modification

## Language Handling

**Auto-detection logic:**
1. Read `CLAUDE.md` or `README.md`
2. Count characters by Unicode block:
   - CJK Unified (Chinese) → "zh"
   - Hangul (Korean) → "ko"
   - Hiragana/Katakana (Japanese) → "ja"
   - Latin with diacritics patterns → detect "es", "fr", "de", "it", "pt"
   - Default → "en"
3. Save to config, no user selection needed

**All subsequent output** must be in the detected language.

---

## Execution Flow

### Step 1: Project Detection (automatic, no user input)

```bash
gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'
```

Auto-detect:
- **owner/name**: from git remote
- **language**: from CLAUDE.md / README.md content analysis
- **existing config**: check if `issue-manager.config.json` exists

If config exists:
```
  Existing config found.
  → Keep current settings? (Y / reset)
```

Output:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 issue-manager setup

[1/6] Project Detection
  ✅ {owner}/{name}
  ✅ Language: {detected}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### Step 2: Project Context Learning (automatic + confirm)

**This is the key step that makes issue-manager project-specific, not generic.**

Every project gets a **baseline** of universal issue quality rules.
Then, project docs are scanned to **discover and layer on** project-specific rules.

#### Baseline (always applied)

These are good for ANY project, regardless of type:

```json
{
  "guidelines": [
    {
      "category": "Issue Quality",
      "rules": [
        "Clearly state the problem and who is affected",
        "Include concrete user scenario or steps to reproduce",
        "Define acceptance criteria (how do we know it's done?)",
        "Check for duplicate issues before creating"
      ]
    }
  ],
  "users": [
    { "name": "User", "context": "General user of the product" }
  ]
}
```

#### Discovery (layered on top of baseline)

Scan all documentation files:

```bash
for f in CLAUDE.md README.md DESIGN.md ARCHITECTURE.md CONTRIBUTING.md \
         docs/**/*.md .github/CONTRIBUTING.md; do
  [ -f "$f" ] && echo "$f"
done
```

Read each file and extract rules, guidelines, constraints, terminology,
and conventions. Group into natural categories based on what's found.

**Extraction approach:**

1. Read all docs
2. Identify sections with rules, guidelines, or constraints
   (checklists, do/don't patterns, terminology tables, persona descriptions,
   naming conventions, design tokens, tone guides, etc.)
3. Group into categories named after what they are
4. Extract glossary/terminology if found
5. **Merge with baseline** — discovered rules ADD to baseline, not replace

**Different projects produce different results:**

```
# Brand new project (no docs):
  └── Issue Quality (4 rules) ← baseline only

# CLI developer tool (README only):
  ├── Issue Quality (4 rules) ← baseline
  └── Code Standards (3 rules) ← discovered from README

# B2C service (rich CLAUDE.md):
  ├── Issue Quality (4 rules) ← baseline
  ├── Brand Values (4 rules) ← discovered
  ├── Legal Framing (3 rules) ← discovered
  ├── Target Audience (3 personas) ← discovered
  ├── Writing Tone (4 rules) ← discovered
  └── Design Constraints (4 rules) ← discovered

# Open source library:
  ├── Issue Quality (4 rules) ← baseline
  ├── API Design (5 conventions) ← discovered
  └── Contribution Guidelines (3 rules) ← discovered
```

**Show results to user:**

```
[2/6] Project Context Learning

  Case A — docs found:
    📖 Scanned: CLAUDE.md (850 lines), docs/glossary.md (120 lines)

    ✅ Baseline: Issue Quality (4 rules)
    ✅ Discovered: {n} guidelines across {n} categories
      ├── {Category A} ({n} rules)
      │   "{first rule preview...}"
      ├── {Category B} ({n} rules)
      │   "{first rule preview...}"
      └── Terminology ({n} terms)

    → Proceed? (Y / edit)

  Case B — no docs found:
    📖 No project documentation found.
    ✅ Baseline: Issue Quality (4 rules) applied

    Enhance with 3 quick questions? (recommended)
      Q1: What does your project do? (one sentence)
      Q2: Who uses it?
      Q3: Any rules for writing issues? (terms to avoid, conventions, etc.)

    Or: Y to continue with baseline only.
          Add docs later → /issue-setup --learn

  Case C — docs found but nothing extractable:
    📖 Scanned: README.md (50 lines) — no guidelines found
    ✅ Baseline: Issue Quality (4 rules) applied

    → Proceed with baseline? (Y / add custom rules)
```

**Saved in config as `projectContext`:**
```json
{
  "projectContext": {
    "description": "One-sentence project description",
    "guidelines": [
      {
        "category": "Issue Quality",
        "source": "baseline",
        "rules": [
          "Clearly state the problem and who is affected",
          "Include concrete user scenario or steps",
          "Define acceptance criteria",
          "Check for duplicates"
        ]
      },
      {
        "category": "Discovered Category",
        "source": "docs",
        "rules": ["rule from docs"],
        "prohibitedTerms": ["if any found"],
        "replacements": {"term": "alternative"}
      }
    ],
    "terminology": [
      {"term": "Domain Term", "definition": "What it means"}
    ],
    "users": [
      {"name": "User Type", "context": "relevant context"}
    ]
  }
}
```

Each guideline category has a `source` field ("baseline" or "docs") so
`/issue-setup --learn` knows which to preserve and which to refresh.

---

### Step 3: GitHub Projects (selection needed)

```bash
gh project list --owner {owner} --format json
```

**Auto-proposal logic:**
- **1 project** → auto-select, just confirm
- **Multiple** → recommend the one with most items `(recommended)`
- **0 projects** → "Proceed without Projects" auto-selected

```
[3/6] GitHub Projects
  Found:
    1. My Roadmap (161 items) ← recommended
    2. Old Board (3 items)

  → Proceed with 1? (Y / number)
```

After selection, **auto-map fields:**

```bash
gh project field-list {number} --owner {owner} --format json
```

Mapping rules (match field name, case-insensitive):
- Contains "phase/stage/단계/フェーズ/阶段/fase/etapa" → phase
- Contains "priority/우선/優先/优先/prioridad/priorité/priorität" → priority
- Contains "track/트랙/トラック/轨道/pista/voie" → track
- Unmatched SingleSelect fields → skip (can add later in config)

```
  ✅ Phase → "Phase" (10 options)
  ✅ Priority → "Priority" (3 options)
  ✅ Track → "Track" (8 options)
  → Correct? (Y / edit)
```

---

### Step 4: Domain Labels (auto-proposed)

```bash
gh label list --json name,description --jq '.[] | select(.name | startswith("domain:"))'
```

**Case A: Existing domain labels found** → use as-is
```
  ✅ 10 existing domain labels found
  → Keep? (Y / add / remove)
```

**Case B: No domain labels** → analyze and propose

Three sources combined:
1. **Directory structure**: `src/app/*/`, `src/components/*/`, `src/lib/*/`
2. **Issue title clustering**: group by repeated keywords
3. **CLAUDE.md / README**: extract major feature areas

```
[4/6] Domain Labels
  🔍 Proposed domains (from project analysis):
    1. auth      — src/app/auth/ found + 4 related issues
    2. payment   — src/components/payment/ found + 3 issues
    3. dashboard — src/app/dashboard/ (8 sub-routes)
    4. api       — src/lib/api/ found + 6 issues
    5. ui        — src/components/ui/ (30+ components)

  → Create these? (Y / add / edit / remove)
```

On Y: create labels with auto-assigned colors from palette.

---

### Step 5: Issue Template (auto-recommended by project size)

**Auto-recommendation logic:**

```bash
issue_count=$(gh issue list --state open --json number --jq 'length')
has_context=$(test -n "$projectContext_principles" && echo "yes" || echo "no")
```

| Condition | Recommendation | Reason |
|-----------|---------------|--------|
| projectContext has guidelines | **Full (7 sections)** | Project has rules to enforce |
| 10+ issues, no projectContext | **Standard (4 sections)** | Typical dev project |
| <10 issues | **Minimal (3 sections)** | Fast iteration first |

```
[5/6] Issue Template
  → Recommended: {type} ({n} sections) — {reason}

  → Proceed? (Y / full / standard / minimal)
```

**If Full selected and projectContext has guidelines,**
auto-generate checklists from Step 2:

```
  Guidelines checklist (from project context):
    ✅ "{rule from category A}"
    ✅ "{rule from category B}"
    ...

  Quality checklist (auto-generated):
    ✅ "No prohibited terms?" (if prohibitedTerms found in context)
    ✅ "Appropriate for target users?" (if users found in context)
    ✅ "{design rule}?" (if design rules found in context)

  → Proceed? (Y / edit)
```

The checklists are built from whatever was discovered in Step 2.
If nothing was found, standard generic checklists are used.

---

### Step 6: Summary + Generate

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Configuration Summary

  Project:      {owner}/{name}
  Language:     {language}
  Context:      {n} guidelines across {n} categories
  Projects:     {name} (#{number})
  Fields:       Phase({n}) + Priority({n}) + Track({n})
  Domains:      {list}
  Template:     {type} ({n} sections)
  References:   {list}

  → Generate issue-manager.config.json? (Y)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

After Y:
1. Write `issue-manager.config.json` (including `projectContext`)
2. Create domain labels in GitHub (if new)
3. Show completion message

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Setup complete!

  Project context: {n} guidelines, {n} terms, {n} user types
  (or: "No project docs found — using generic mode")

  Available commands:
    /issue          — create issues (with your project's rules applied)
    /issue-refine   — strengthen existing issues
    /issue-organize — structure your backlog
    /issue-status   — progress dashboard
    /issue-close    — cleanup stale/done issues

  Tip: after updating project docs, run /issue-setup --learn to re-sync
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## How projectContext is used by other commands

### /issue (create)
- **Guidelines check**: uses `projectContext.guidelines[].rules` as checklist in issue body
- **Quality check**: validates against any `prohibitedTerms` found in guidelines
- **Terminology**: uses correct project terms from `projectContext.terminology`
- **User awareness**: considers `projectContext.users` when writing scenarios
- **All rules**: applies whatever rules were discovered — the format adapts to the project

### /issue-refine (improve)
- **Scoring**: extra points for guideline compliance
- **Validation**: flags prohibited terms, wrong terminology in existing content
- **Added content**: follows project tone and conventions

### /issue-organize (structure)
- **Epic descriptions**: follow project conventions and terminology

---

## User Involvement Summary

| Step | Auto | User |
|------|:----:|:----:|
| Project detection | ✅ fully auto | none |
| Language | ✅ auto-detect | none |
| **Context learning** | **✅ extract from docs** | **Y (confirm)** |
| Projects selection | ✅ recommend best | **Y or number** |
| Field mapping | ✅ name-based | Y (confirm) |
| Domain labels | ✅ structure+issues+docs | Y (confirm) |
| Template + checklists | ✅ context-based recommend | Y (confirm) |
| Generate | config assembly | **Y** |

**Typical flow: 1 selection (Projects) + 6× Enter**

---

## Color Palette (auto-assigned to domain labels)

```
1D76DB, 5319E7, 0E8A16, FBCA04, 006B75,
B60205, F9D0C4, FF9500, D4C5F9, C2E0C6,
E4E669, BFD4F2, D93F0B, 0075CA, 7057FF
```

## $ARGUMENTS

- No args → full setup wizard
- `--learn` → re-scan all docs and update `projectContext` only (other settings unchanged)
- `--reset` → delete existing config, start fresh
- `--check` → validate current config only

### --learn (Context Re-sync)

When project documentation evolves, run `/issue-setup --learn` to refresh:

```
/issue-setup --learn

  📖 Re-scanning project documentation...
     Found: CLAUDE.md (920 lines), README.md (200 lines), docs/glossary.md (130 lines)

  Changes detected:
    + 1 new guideline in "Code Standards": "..."
    + 2 new terms in Terminology
    ~ Updated user context: "..."
    - Removed: (none)

  → Update projectContext? (Y / diff / cancel)
```

This only updates `projectContext` in the config file. All other settings (projects, labels, template) remain unchanged.

**When to use:**
- After editing project documentation with new guidelines
- After adding a glossary or convention document
- After a major product direction change
