---
name: cc-sandbox
description: 'This skill should be used when configuring Claude Code sandboxing: enabling the sandbox, choosing sandbox modes (auto-allow vs regular permissions), configuring filesystem restrictions (allowWrite, denyRead, denyWrite, allowRead), configuring network domain allow/deny lists, excluding commands from the sandbox, setting up custom proxy, or explaining how sandboxing interacts with permissions and how to install prerequisites on Linux/WSL2.'
---

# Claude Code Sandboxing

## Purpose

Guide users in enabling and configuring Claude Code's OS-level sandbox, which restricts Bash commands to defined filesystem paths and network domains. The sandbox complements the permissions system and reduces permission-prompt fatigue for safe commands.

## Reference material

Load `references/sandboxing.md` for the complete sandboxing documentation, including all settings keys, path prefix semantics, sandbox modes, platform prerequisites, security limitations, and integration details.

## Workflow

### 1. Identify the goal

Common requests fall into these categories:

- **Enable sandboxing** — run `/sandbox` in Claude Code, or set `sandbox.enabled: true` in settings
- **Configure filesystem access** — `allowWrite`, `denyRead`, `denyWrite`, `allowRead`
- **Configure network access** — `allowedDomains`, `deniedDomains`
- **Exclude a tool from the sandbox** — `excludedCommands`
- **Install prerequisites** — bubblewrap + socat on Linux/WSL2
- **Understand how sandbox and permissions interact** — explain layered model

### 2. Determine the correct settings file

| Scope | File |
|---|---|
| Organization (non-overridable) | Managed settings |
| Project (shared) | `.claude/settings.json` |
| Project (personal) | `.claude/settings.local.json` |
| User (all projects) | `~/.claude/settings.json` |

`sandbox.filesystem` arrays (`allowWrite`, `denyRead`, etc.) **merge across all scopes** — they are not replaced by higher-priority scopes. Each scope adds to the combined list.

### 3. Write the configuration

Minimal example to enable sandbox with extra write paths and domain restrictions:

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"],
      "denyRead": ["~/"],
      "allowRead": ["."]
    },
    "network": {
      "allowedDomains": ["registry.npmjs.org", "github.com"],
      "deniedDomains": ["example-blocked.com"]
    }
  }
}
```

Key path prefix rules:
- `/path` — absolute from filesystem root (`/tmp/build` stays `/tmp/build`)
- `~/path` — relative to home directory
- `./path` or bare `path` — relative to project root (in project settings) or `~/.claude` (in user settings)

Note: sandbox path prefixes differ from permission rule paths — here `/tmp/build` IS absolute, unlike in `Read`/`Edit` permission rules where `/path` means project-root-relative.

### 4. Handle common tool incompatibilities

- **docker** — incompatible with sandbox; add `"docker *"` to `excludedCommands`
- **watchman / jest** — use `jest --no-watchman`
- **Windows binaries in WSL2** — add to `excludedCommands` (sandbox blocks `/mnt/c/` access)
- **Sandbox escape hatch** — Claude may retry failed commands with `dangerouslyDisableSandbox`; disable with `"allowUnsandboxedCommands": false`

### 5. Platform prerequisites

- **macOS** — works out of the box (Seatbelt)
- **Linux / WSL2** — install `bubblewrap` and `socat` first; on Ubuntu 24.04+ also add an AppArmor profile for `bwrap`
- **WSL1** — not supported; upgrade to WSL2
- **Windows (native)** — not yet supported; use WSL2

Refer to the reference for exact install commands and the AppArmor profile snippet.
