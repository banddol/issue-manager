# issue-manager

Full-lifecycle GitHub issue management for Claude Code.

Create, refine, organize, monitor, and clean up issues — all with consistent quality and service-oriented context.

## Commands

| Command | Lifecycle | Purpose |
|---------|:---------:|---------|
| `/issue-setup` | Setup | Auto-detect project structure, configure labels/fields/templates |
| `/issue` | Create | Create issues with service context, UX flows, and technical design |
| `/issue-refine` | Improve | Strengthen existing issues, quality audit with scoring |
| `/issue-organize` | Structure | Domain clustering, master epics, duplicate detection, dependency mapping |
| `/issue-status` | Monitor | Progress dashboard, stale alerts, weekly trends |
| `/issue-close` | Cleanup | Close stale issues, detect completed, merge duplicates |

```
Lifecycle:

  /issue-setup → /issue → /issue-refine → /issue-organize → /issue-status → /issue-close
     Setup       Create     Improve         Structure         Monitor         Cleanup
```

## Install

```bash
git clone https://github.com/banddol/issue-manager.git ~/.claude/skills/issue-manager
```

## Quick Start

```bash
/issue-setup    # Interactive wizard — auto-detects everything, you just approve
```

The wizard:
1. Detects your repo and language
2. **Learns your project's rules** by scanning all docs (CLAUDE.md, README.md, CONTRIBUTING.md, docs/*.md, etc.) — extracts guidelines, terminology, user personas, conventions
3. Connects GitHub Projects and auto-maps fields
4. Analyzes project structure to suggest domain labels
5. Generates `issue-manager.config.json`

No docs? A baseline of 4 universal issue quality rules is always applied. Optionally answer 3 quick questions to add project context.

Most steps only need **Y** to proceed.

### Updating context

When your project docs evolve, re-sync:

```bash
/issue-setup --learn    # Re-scans docs, shows what changed, updates config
```

## Language Support

Auto-detects your project's language from documentation content. Supports: en, ko, ja, zh, es, fr, de, it, pt.

## Usage

```bash
# Create a new issue
/issue Users can't find notification settings, too many alerts

# Refine existing issues
/issue-refine #94 #103

# Quality audit
/issue-refine --audit

# Backlog structure
/issue-organize
/issue-organize --find-orphans
/issue-organize --create-master domain:auth

# Status dashboard
/issue-status
/issue-status --weekly
/issue-status --alerts

# Cleanup
/issue-close --stale
/issue-close --done
/issue-close --merge #133 #219
```

## Configuration

After `/issue-setup`, your project will have an `issue-manager.config.json` with:

| Section | Purpose |
|---------|---------|
| `project` | GitHub owner, Projects ID |
| `labels` | Type/scope/domain definitions |
| `projectFields` | Projects field IDs (Phase, Priority, Track) |
| `template` | Issue body sections, quality checklist |
| `references` | Files to read for context |
| `projectContext` | Auto-learned: guidelines, terminology, users (baseline + discovered) |
| `language` | Output language (auto-detected) |
| `scoring` | Quality thresholds for audits |

See `issue-manager.config.example.json` for a full example.

## Label Strategy

3-axis label system — static attributes only:

```
type:*    — What (feature, bug, infra, task, design, idea, research)
scope:*   — Where (frontend, backend, both)
domain:*  — Which area (your custom domains)
```

Dynamic attributes (Phase, Priority, Track, Status) are managed in GitHub Projects, not labels.

## License

MIT
