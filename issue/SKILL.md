---
description: "Create GitHub issues with service-oriented quality. Describe a problem or idea, get a structured issue with context, UX flow, and technical design."
allowed-tools: [Read, Bash, Glob, Grep, WebSearch, Agent]
---

# /issue — Issue Creator

Create new GitHub issues with consistent, service-oriented quality.

## Language

Read `config.language` from `issue-manager.config.json`.
**All output (issue title, body, comments) must be in that language.**
If no config exists, suggest running `/issue-setup` first.

## Input

```
/issue [problem, idea, or feature — natural language in any language]
```

## Execution

### 1. Read Config + Context

- Read `issue-manager.config.json` for labels, fields, template, language
- Read files listed in `config.references[]` for project context
- Check for duplicates: `gh issue list --label "domain:{relevant}" --state open`

### 2. Build Issue Body

Use `config.template.sections[]`. For each section:
- `required: true` → always include
- `required: false` → include if relevant

**Section guide:**

**Background (WHY)**
- Problem: what's broken or missing?
- Affected users: who, what role, what context? (concrete personas)
- Impact: what if we don't? what value if we do?

**Brand/DNA Check** (if configured in `config.template.sections[name=brandCheck]`)
- Run through the configured checklist items

**User Scenario**
- Step-by-step UX flow with ASCII mockups or numbered steps
- Include: happy path, empty state, error state, success feedback

**Features**
- Checklist with priority: 🔴 Must / 🟡 Should / 🟢 Could

**Technical Design** (if applicable)
- Components, API endpoints, DB changes, dependencies (`#number`)

**Design Guidelines** (if applicable)
- Writing tone, information density, accessibility notes

**Related Issues**
- Issue numbers with relationship description

### 3. Determine Labels (3-axis)

From `config.labels`:
- **type**: match issue nature → `config.labels.types[]`
- **scope**: determine → `config.labels.scopes[]`
- **domain**: match → `config.labels.domains[]`

### 4. Quality Check

Run `config.template.qualityChecklist[]` before creating. Fix failures.

### 5. Create Issue

```bash
gh issue create --title "{prefix}: {title}" --body "{body}" --label "type:X,scope:X,domain:X"
```

### 6. Register to Projects

If `config.project.projectNumber` exists:
```bash
gh project item-add {projectNumber} --owner {owner} --url {issue_url}
```
Then set Phase + Priority via `config.projectFields`.

### 7. Report

Show: issue number + URL, labels, Phase/Priority, related issues.

## Multiple Issues

If input contains multiple distinct items, create each separately.

## $ARGUMENTS

1. **Natural language text** → create new issue
2. **Multiple items** (numbered/comma-separated) → create each separately
