# Claude Code Sandboxing — Complete Reference

Source: https://code.claude.com/docs/en/sandboxing

## Overview

Sandboxing uses OS-level primitives to enforce filesystem and network isolation on Bash commands and all their child processes. It complements (but does not replace) the permissions system.

- **macOS**: Seatbelt
- **Linux / WSL2**: bubblewrap + socat
- **WSL1**: not supported (lacks required kernel namespace primitives)
- **Windows native**: not yet supported

**Both** filesystem and network isolation are needed for effective security — without network isolation a compromised agent can exfiltrate files; without filesystem isolation it can backdoor system resources.

## Enabling the sandbox

Run `/sandbox` in Claude Code to open the mode selection menu. Or set in settings:

```json
{
  "sandbox": {
    "enabled": true
  }
}
```

By default, if the sandbox cannot start (missing deps, unsupported platform), Claude Code warns and continues without sandboxing. To make this a hard failure:

```json
{
  "sandbox": {
    "failIfUnavailable": true
  }
}
```

## Sandbox modes

**Auto-allow mode**: Sandboxed Bash commands run without permission prompts. Commands that cannot be sandboxed (e.g. need network access to a non-allowed host) fall back to the regular permission flow. Explicit deny rules always apply. `rm`/`rmdir` targeting `/`, home directory, or other critical paths still prompt.

**Regular permissions mode**: All Bash commands go through the standard permission flow even when sandboxed. More control, more prompts.

Auto-allow works independently of the `defaultMode` permission setting — even outside "accept edits" mode, sandboxed Bash runs without prompting.

## Filesystem isolation

Default behavior:
- **Write**: current working directory and subdirectories only
- **Read**: entire filesystem, except denied paths

### Settings keys

All `sandbox.filesystem` arrays **merge across settings scopes** (managed + user + project + local) rather than being replaced.

| Key | Effect |
|---|---|
| `sandbox.filesystem.allowWrite` | Grant write access to additional paths |
| `sandbox.filesystem.denyWrite` | Block write access to specific paths |
| `sandbox.filesystem.denyRead` | Block read access to specific paths |
| `sandbox.filesystem.allowRead` | Re-allow reads within a denied region (takes precedence over denyRead) |

When `allowManagedReadPathsOnly` is enabled in managed settings, only managed `allowRead` entries are respected; user, project, and local entries are ignored. `denyRead` still merges from all sources.

### Path prefix semantics (sandbox settings)

| Prefix | Meaning | Example |
|---|---|---|
| `/path` | Absolute from filesystem root | `/tmp/build` → `/tmp/build` |
| `~/path` | Relative to home directory | `~/.kube` → `$HOME/.kube` |
| `./path` or bare `path` | Project-root-relative (project settings) or `~/.claude`-relative (user settings) | `./output` → `<project-root>/output` |

**Important**: These differ from permission rule path prefixes. In sandbox settings `/tmp/build` IS an absolute path. In `Read`/`Edit` permission rules, `/path` means project-root-relative and `//path` means absolute.

The older `//path` prefix for absolute paths still works in sandbox settings.

### Example: block home directory reads but allow project

In `.claude/settings.json`:

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "denyRead": ["~/"],
      "allowRead": ["."]
    }
  }
}
```

`.` resolves to the project root in project settings, to `~/.claude` in user settings.

### Enforcement scope

Filesystem restrictions are enforced at the OS level — they apply to all subprocess commands including `kubectl`, `terraform`, `npm`, and their child processes, not just Claude's built-in file tools.

## Network isolation

Network access is controlled through a proxy running outside the sandbox.

- **New domain requests**: trigger a permission prompt (unless `allowManagedDomainsOnly` is enabled, which auto-blocks non-listed domains)
- **Proxy inspection**: The built-in proxy enforces the allowlist by hostname and does NOT terminate or inspect TLS traffic
- **All subprocesses**: restrictions apply to all scripts and programs spawned by sandbox commands

### Settings keys

```json
{
  "sandbox": {
    "network": {
      "allowedDomains": ["registry.npmjs.org", "github.com"],
      "deniedDomains": ["blocked.example.com"],
      "httpProxyPort": 8080,
      "socksProxyPort": 8081
    }
  }
}
```

`deniedDomains` blocks specific domains even when a broader `allowedDomains` wildcard would otherwise permit them.

### Security limitation — domain fronting

The proxy makes its allow decision from the client-supplied hostname without inspecting TLS. Code inside the sandbox can potentially use domain fronting or similar techniques to reach hosts outside the allowlist. For stronger guarantees, use a custom proxy that terminates TLS and inspects traffic.

Allowing broad domains like `github.com` can create data exfiltration paths.

## Excluding commands from the sandbox

Some tools are incompatible with the sandbox. Use `excludedCommands` to force them through the regular permission flow:

```json
{
  "sandbox": {
    "excludedCommands": ["docker *", "my-tool"]
  }
}
```

Common incompatibilities:
- **docker** — incompatible; add `"docker *"` to `excludedCommands`
- **watchman** — incompatible; use `jest --no-watchman` instead
- **WSL2 Windows binaries** (`cmd.exe`, `powershell.exe`, anything under `/mnt/c/`) — sandbox blocks the Unix socket handoff; add to `excludedCommands`

## Escape hatch

When a command fails due to sandbox restrictions, Claude may retry it with `dangerouslyDisableSandbox`. Those commands go through the normal permission flow. To disable the escape hatch entirely:

```json
{
  "sandbox": {
    "allowUnsandboxedCommands": false
  }
}
```

When disabled, `dangerouslyDisableSandbox` is ignored and all commands must either run sandboxed or be in `excludedCommands`.

## Platform prerequisites

### macOS

Works out of the box. No additional installation needed.

### Linux / WSL2

Install bubblewrap and socat:

```bash
# Ubuntu/Debian
sudo apt-get install bubblewrap socat

# Fedora
sudo dnf install bubblewrap socat
```

**Ubuntu 24.04+**: Default AppArmor policy blocks bubblewrap from creating user namespaces. Add an AppArmor profile:

```bash
sudo tee /etc/apparmor.d/bwrap > /dev/null <<'EOF'
abi <abi/4.0>,
include <tunables/global>

profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,
  include if exists <local/bwrap>
}
EOF

sudo systemctl reload apparmor
```

This profile applies only to `bwrap` itself, not to commands inside the sandbox.

### WSL1

Not supported. Upgrade to WSL2 or run without sandboxing.

### WSL2 — Windows binary limitation

Sandboxed commands cannot launch Windows binaries (`cmd.exe`, `powershell.exe`, `/mnt/c/` executables). Add those to `excludedCommands`.

### Linux — weak nested sandbox mode

`enableWeakerNestedSandbox` enables sandboxing inside Docker environments without privileged namespaces. This considerably weakens security — only use when additional isolation is enforced externally.

## How sandbox and permissions interact

| Layer | Scope | Applies to |
|---|---|---|
| Permissions (allow/deny/ask rules) | All tools | Bash, Read, Edit, WebFetch, MCP, Agent |
| Sandbox filesystem | OS-level enforcement | Bash commands and their child processes only |
| Sandbox network | OS-level enforcement | Bash commands and their child processes only |

Permissions are evaluated first (before any tool runs). The sandbox then enforces boundaries at the OS level.

`Read`/`Edit` deny permission rules AND `sandbox.filesystem.denyRead`/`denyWrite` are merged into the final sandbox configuration — both apply simultaneously.

`WebFetch` allow/deny rules AND sandbox `allowedDomains`/`deniedDomains` combine for network control.

## What sandboxing does NOT cover

- **Built-in file tools** (Read, Edit, Write) — controlled by the permissions system, not the sandbox
- **Computer use** — runs on the actual desktop, not in an isolated environment

## Security limitations summary

1. **TLS inspection**: built-in proxy does not inspect HTTPS traffic — domain fronting is possible
2. **Unix sockets**: `allowUnixSockets` can grant access to powerful services (e.g. Docker socket = host escape)
3. **Broad write permissions**: allowing writes to `$PATH` dirs, shell config files (`.bashrc`, `.zshrc`), or system config can enable privilege escalation
4. **Weak nested sandbox** (`enableWeakerNestedSandbox`): considerably weaker — use only with external isolation

## Custom proxy configuration

For organizations requiring TLS inspection or custom filtering:

```json
{
  "sandbox": {
    "network": {
      "httpProxyPort": 8080,
      "socksProxyPort": 8081
    }
  }
}
```

Install the proxy's CA certificate inside the sandbox so TLS connections succeed.

## Open source sandbox runtime

The sandbox runtime is available as an npm package for use in other agent projects:

```bash
npx @anthropic-ai/sandbox-runtime <command-to-sandbox>
```

Source: https://github.com/anthropic-experimental/sandbox-runtime

## Settings precedence for sandbox arrays

`allowWrite`, `denyRead`, `denyWrite`, `allowRead`, `allowedDomains`, `deniedDomains` all **merge** across settings scopes (managed + user + project + local). They are never replaced — each scope adds to the combined list. This lets managed settings establish a baseline that users and projects extend without overriding.
