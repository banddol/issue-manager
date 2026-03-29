---
name: issue-manager
description: |
  Full-lifecycle GitHub issue management. Create, refine, organize, monitor, and clean up
  issues with consistent quality and service-oriented context. Use sub-commands:
  /issue (create), /issue-setup (configure), /issue-refine (improve),
  /issue-organize (structure), /issue-status (dashboard), /issue-close (cleanup).
allowed-tools: [Read, Bash, Glob, Grep, WebSearch, Agent]
---

# /issue-manager — Help & Navigation

Show help, check setup status, and route to sub-commands.

## Input

```
/issue-manager              — Show help (default)
/issue-manager help         — Same as above
/issue-manager <command>    — Route to sub-command
```

## Execution

### 1. Read Config Status

```bash
# Check if config exists in current project
test -f issue-manager.config.json && echo "configured" || echo "not configured"
```

If config exists, read it to extract: `project.owner`, `project.name`, `language`, domain count, projectContext summary.

### 2. Display Help

**Case A: Config exists (configured project)**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  issue-manager — GitHub Issue Lifecycle

  Project:   {owner}/{name}
  Language:  {language}
  Domains:   {n} configured
  Context:   {n} guidelines, {n} terms
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Lifecycle:

    /issue-setup → /issue → /issue-refine → /issue-organize → /issue-status → /issue-close
       Setup       Create     Improve         Structure         Monitor         Cleanup

  Commands:

    /issue <description>         Create a new issue with service context
    /issue-refine #N             Strengthen existing issues, quality audit
    /issue-refine --audit        Score all open issues
    /issue-organize              Domain clustering, epics, dependency mapping
    /issue-organize --find-orphans
    /issue-status                Progress dashboard, stale alerts
    /issue-status --weekly       Weekly change report
    /issue-close --stale         Close stale issues
    /issue-close --done          Detect and close completed issues
    /issue-close --merge #A #B   Merge duplicate issues

  Config:

    /issue-setup --learn         Re-sync project context from docs
    /issue-setup --check         Validate current config
    /issue-setup --reset         Start fresh

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Case B: No config (first-time user)**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  issue-manager — GitHub Issue Lifecycle

  Not configured yet.

  Get started:

    /issue-setup       Interactive wizard — auto-detects everything, you just approve

  The wizard will:
    1. Detect your repo and language
    2. Learn your project's rules from docs
    3. Connect GitHub Projects and map fields
    4. Suggest domain labels from project structure
    5. Generate issue-manager.config.json

  Most steps just need Y to proceed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  After setup, these commands become available:

    /issue             Create issues with project context
    /issue-refine      Strengthen existing issues
    /issue-organize    Structure your backlog
    /issue-status      Progress dashboard
    /issue-close       Cleanup stale/done issues

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 3. Sub-command Routing

If the user provides a known sub-command keyword, route to the appropriate skill:

| Input | Routes to |
|-------|-----------|
| `/issue-manager setup` | `/issue-setup` |
| `/issue-manager create <text>` | `/issue <text>` |
| `/issue-manager refine <args>` | `/issue-refine <args>` |
| `/issue-manager organize` | `/issue-organize` |
| `/issue-manager status` | `/issue-status` |
| `/issue-manager close <args>` | `/issue-close <args>` |
| `/issue-manager help` | Show help (this screen) |

When routing, tell the user which command is being executed:
```
  → Routing to /issue-status...
```

Then invoke the target skill with the remaining arguments.

## $ARGUMENTS

1. No args or `help` → show help screen
2. `setup` → route to `/issue-setup`
3. `create <text>` → route to `/issue <text>`
4. `refine <args>` → route to `/issue-refine <args>`
5. `organize` → route to `/issue-organize`
6. `status` → route to `/issue-status`
7. `close <args>` → route to `/issue-close <args>`
