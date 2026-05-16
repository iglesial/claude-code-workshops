---
name: 'cc-permissions'
description: 'This skill should be used when configuring Claude Code permissions: writing allow/deny/ask rules in settings.json, choosing permission modes (default, acceptEdits, plan, auto, dontAsk, bypassPermissions), setting up managed/org-level policy, restricting Bash/PowerShell/Read/Edit/WebFetch/MCP/Agent tools, configuring working directories, or explaining how the permission system works.'
---

# Claude Code Permissions

## Purpose

Guide users in configuring Claude Code's permission system: rule syntax, permission modes, settings file hierarchy, managed org policies, and integration with sandboxing and hooks.

## Reference material

Load `references/permissions.md` for the complete permissions documentation, including all rule syntax, mode descriptions, path pattern details, and managed-settings reference.

## Workflow

### 1. Identify the goal

Common requests fall into these categories:

- **Allow a tool/command** — add to `permissions.allow`
- **Block a tool/command** — add to `permissions.deny`
- **Require confirmation** — add to `permissions.ask`
- **Set a permission mode** — set `defaultMode`
- **Org-wide policy** — use managed settings
- **Understand the system** — explain from the reference

### 2. Determine the correct settings file

| Scope | File |
|---|---|
| Organization (non-overridable) | Managed settings (MDM / `managed-settings.json`) |
| Project (shared with team) | `.claude/settings.json` |
| Project (personal only) | `.claude/settings.local.json` |
| User (all projects) | `~/.claude/settings.json` |

Precedence (high to low): managed → CLI args → local project → shared project → user.

### 3. Write the rule

Use the correct tool prefix and specifier syntax. Examples:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "WebFetch(domain:api.example.com)",
      "mcp__puppeteer"
    ],
    "deny": [
      "Bash(git push *)",
      "Read(**/.env)",
      "Edit(/secrets/**)"
    ],
    "ask": [
      "Bash(rm *)"
    ]
  }
}
```

Key rule syntax rules to remember:
- Deny rules always win — evaluated before allow and ask
- `Bash(cmd *)` enforces a word boundary (matches `cmd arg` but not `cmdtool`)
- `Bash(cmd*)` without a space does NOT enforce a word boundary
- A single `*` spans multiple arguments: `Bash(git *)` matches `git log --all`
- Compound commands (`&&`, `||`, `;`, `|`) — each subcommand is checked independently
- `Read(.env)` and `Read(**/.env)` are equivalent (gitignore semantics)
- Use `//` prefix for absolute filesystem paths: `Read(//etc/passwd)`
- Use `~/` for home-relative paths: `Read(~/.ssh/**)`
- Use `/` for project-root-relative: `Edit(/src/**)`

### 4. Set a permission mode (if needed)

Add `"defaultMode"` to the relevant settings file:

```json
{
  "defaultMode": "acceptEdits"
}
```

Refer to the reference for mode descriptions and when to use each one.

### 5. Validate and explain

After writing config, confirm the rule achieves the stated goal and flag any known limitations (e.g., Bash URL filtering fragility, symlink behavior).
