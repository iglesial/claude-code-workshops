# Claude Code Permissions — Complete Reference

Source: https://code.claude.com/docs/en/permissions

## Permission system overview

Claude Code uses a tiered permission system:

| Tool type | Example | Approval required | "Yes, don't ask again" behavior |
|---|---|---|---|
| Read-only | File reads, Grep | No | N/A |
| Bash commands | Shell execution | Yes | Permanently per project directory and command |
| File modification | Edit/write files | Yes | Until session end |

Rules are evaluated in order: **deny → ask → allow**. The first matching rule wins, so deny rules always take precedence.

Permission rules are enforced by Claude Code, not by the model. CLAUDE.md instructions shape what Claude tries to do but do not change what Claude Code allows.

## Permission modes

Set with `"defaultMode"` in settings files.

| Mode | Description |
|---|---|
| `default` | Prompts for permission on first use of each tool |
| `acceptEdits` | Auto-accepts file edits and common filesystem commands (`mkdir`, `touch`, `mv`, `cp`, etc.) for paths in the working directory or `additionalDirectories` |
| `plan` | Claude reads files and runs read-only shell commands to explore but does not edit source files |
| `auto` | Auto-approves tool calls with background safety checks; currently a research preview |
| `dontAsk` | Auto-denies tools unless pre-approved via `/permissions` or `permissions.allow` rules |
| `bypassPermissions` | Skips all permission prompts. Root and home directory removals still prompt as a circuit breaker. **Only use in isolated environments (containers/VMs).** |

To prevent `bypassPermissions` or `auto` mode from being used, set `permissions.disableBypassPermissionsMode` or `permissions.disableAutoMode` to `"disable"` in any settings file (most useful in managed settings).

## Permission rule syntax

Rules follow the format `Tool` or `Tool(specifier)`.

### Match all uses of a tool

`Bash`, `WebFetch`, `Read` — matches all uses. `Bash(*)` is equivalent to `Bash`.

### Bash rules

- `Bash(npm run build)` — exact match
- `Bash(npm run test *)` — commands starting with `npm run test`
- `Bash(npm *)` — any command starting with `npm ` (note space — word boundary)
- `Bash(npm*)` — also matches `npmx` (no word boundary)
- `Bash(* install)` — any command ending with ` install`
- `Bash(git * main)` — matches `git checkout main`, `git push origin main`, etc.
- `Bash(ls:*)` — equivalent to `Bash(ls *)` (`:*` suffix = trailing wildcard)

A single `*` matches any sequence including spaces, so one wildcard can span multiple arguments.

**Word boundary**: Space before `*` enforces a word boundary. `Bash(ls *)` matches `ls -la` but not `lsof`. `Bash(ls*)` matches both.

**Compound commands**: `&&`, `||`, `;`, `|`, `|&`, `&`, and newlines split into subcommands. Each is checked independently.

**Process wrappers stripped before matching**: `timeout`, `time`, `nice`, `nohup`, `stdbuf`, bare `xargs`. So `Bash(npm test *)` also matches `timeout 30 npm test`. `npx`, `docker exec`, `devbox run`, `direnv exec`, `mise exec` are NOT stripped — write explicit rules for them.

**Exec wrappers that always prompt** (cannot be auto-approved by prefix): `watch`, `setsid`, `ionice`, `flock`, `find -exec`, `find -delete`.

**Read-only commands** (run without prompts in all modes): `ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only forms of `git`. Not configurable; to require a prompt, add an `ask` or `deny` rule.

**URL filtering warning**: Bash URL-filtering rules are fragile (option order, protocol variants, redirects, variables). Instead: deny `curl`/`wget` in Bash and use `WebFetch(domain:...)` allow rules, or implement a PreToolUse hook.

Example config:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Bash(git * main)",
      "Bash(* --version)",
      "Bash(* --help *)"
    ],
    "deny": [
      "Bash(git push *)"
    ]
  }
}
```

### PowerShell rules

Same shape as Bash rules. Wildcards with `*` at any position. Common aliases are canonicalized (e.g., `PowerShell(Get-ChildItem *)` also matches `gci`, `ls`, `dir`). Case-insensitive. Pipeline `|`, statement separator `;`, and (PS7+) `&&`/`||` split compound commands.

```json
{
  "permissions": {
    "allow": ["PowerShell(Get-ChildItem *)", "PowerShell(git commit *)"],
    "deny": ["PowerShell(Remove-Item *)"]
  }
}
```

### Read and Edit rules

`Edit` rules apply to all built-in file-editing tools. `Read` rules apply to all built-in file-reading tools (Grep, Glob, etc.).

**Path pattern types** (gitignore spec):

| Pattern | Meaning | Example | Matches |
|---|---|---|---|
| `//path` | Absolute from filesystem root | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | From home directory | `Read(~/Documents/*.pdf)` | `~/Documents/*.pdf` |
| `/path` | Relative to project root | `Edit(/src/**/*.ts)` | `<project root>/src/**/*.ts` |
| `path` or `./path` | Relative to current directory | `Read(*.env)` | `<cwd>/*.env` |

**Warning**: `/Users/alice/file` is NOT an absolute path — it's project-root-relative. Use `//Users/alice/file` for absolute paths.

**Windows**: paths normalized to POSIX form. `C:\Users\alice` → `/c/Users/alice`. Use `//c/**/.env` for drive-specific, `//**/.env` for all drives.

**Gitignore semantics**: `*` matches single directory; `**` matches recursively. Bare filenames like `Read(.env)` and `Read(**/.env)` are equivalent (match at any depth).

**Symlinks**: allow rules require both symlink path and target to match; deny rules apply when either matches.

Examples:
- `Edit(/docs/**)` — edits in `<project>/docs/`
- `Read(~/.zshrc)` — home directory `.zshrc`
- `Edit(//tmp/scratch.txt)` — absolute path
- `Read(src/**)` — from `<cwd>/src/`

### WebFetch rules

- `WebFetch(domain:example.com)` — fetch requests to example.com

### MCP rules

- `mcp__puppeteer` — all tools from the `puppeteer` MCP server
- `mcp__puppeteer__*` — same (wildcard equivalent)
- `mcp__puppeteer__puppeteer_navigate` — specific tool from that server

### Agent (subagent) rules

- `Agent(Explore)` — matches the Explore subagent
- `Agent(Plan)` — matches the Plan subagent
- `Agent(my-custom-agent)` — custom subagent by name

To disable the Explore agent:

```json
{
  "permissions": {
    "deny": ["Agent(Explore)"]
  }
}
```

## Settings files and precedence

Precedence (high → low):
1. **Managed settings** — cannot be overridden by any other level
2. **Command line arguments** — temporary session overrides
3. **Local project settings** — `.claude/settings.local.json`
4. **Shared project settings** — `.claude/settings.json`
5. **User settings** — `~/.claude/settings.json`

If a tool is denied at any level, no other level can allow it.

## Extend permissions with hooks

PreToolUse hooks run before the permission prompt. Hook output can deny, force a prompt, or skip the prompt.

Hook decisions interact with rules:
- A blocking hook (exit code 2) stops the call before rules run — overrides even allow rules
- A hook returning `"allow"` does NOT override deny or ask rules; those still apply

To run all Bash without prompts except specific blocked commands: add `"Bash"` to allow and register a PreToolUse hook that rejects those commands.

## Working directories

By default Claude has access to the launch directory. To extend:
- CLI: `--add-dir <path>`
- Session: `/add-dir`
- Persistent: `additionalDirectories` in settings files

Additional directories do NOT make `.claude/` config from those dirs fully active. What IS loaded from `--add-dir` directories:
- Skills in `.claude/skills/` (with live reload)
- `enabledPlugins` and `extraKnownMarketplaces` from `.claude/settings.json`
- CLAUDE.md files only when `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` is set

## Permissions and sandboxing

Complementary layers:
- **Permissions** — control which tools Claude Code can use (all tools)
- **Sandboxing** — OS-level enforcement for Bash commands and child processes only

Use both for defense-in-depth:
- Deny rules block even the attempt
- Sandbox prevents Bash from reaching restricted resources even if prompt injection bypasses Claude
- `sandbox.filesystem` settings + Read/Edit deny rules merge into the final sandbox boundary
- WebFetch permission rules + `sandbox.allowedDomains`/`deniedDomains` combine

With `autoAllowBashIfSandboxed: true` (default), sandboxed Bash runs without prompting even if `ask: Bash(*)` is set. Explicit deny rules still apply.

## Managed settings

Settings that administrators deploy and that cannot be overridden by user or project settings.

### Managed-only settings (only effective in managed settings)

| Setting | Description |
|---|---|
| `allowedChannelPlugins` | Allowlist of channel plugins |
| `allowManagedHooksOnly` | When `true`, only managed/SDK/force-enabled-plugin hooks load; blocks all other hooks |
| `allowManagedMcpServersOnly` | When `true`, only `allowedMcpServers` from managed settings respected |
| `allowManagedPermissionRulesOnly` | When `true`, prevents user/project settings from defining allow/ask/deny rules |
| `blockedMarketplaces` | Blocklist of marketplace sources |
| `channelsEnabled` | Allow channels for the org |
| `forceRemoteSettingsRefresh` | When `true`, blocks CLI startup until remote managed settings freshly fetched |
| `pluginTrustMessage` | Custom message for plugin trust warning |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Only `filesystem.allowRead` paths from managed settings respected |
| `sandbox.network.allowManagedDomainsOnly` | Only managed `allowedDomains` and WebFetch allow rules respected |
| `strictKnownMarketplaces` | Controls which marketplace sources users can add |
| `wslInheritsWindowsSettings` | WSL reads managed settings from Windows policy chain |

`disableBypassPermissionsMode` works from any scope but is typically placed in managed settings.
