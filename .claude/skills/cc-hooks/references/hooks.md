# Claude Code Hooks — Complete Reference

Source: https://code.claude.com/docs/en/hooks

## Hook handler types

| Type | Description |
|---|---|
| `command` | Shell script receiving JSON on stdin |
| `http` | HTTP POST to an endpoint with JSON body |
| `mcp_tool` | Call to a tool on a connected MCP server |
| `prompt` | Single-turn LLM evaluation returning yes/no |
| `agent` | Subagent that can use tools to verify conditions |

## Configuration format

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolName",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/script.sh",
            "if": "Bash(rm *)",
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

### Hook file locations

| File | Scope | Shareable |
|---|---|---|
| `~/.claude/settings.json` | All projects | No |
| `.claude/settings.json` | Single project | Yes |
| `.claude/settings.local.json` | Single project | No |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes |
| Skill/agent frontmatter | While component active | Yes |

Hooks load only from the project `.claude/` folder and `~/.claude/` — not from `--add-dir` directories (except plugins).

## Matcher syntax

| Matcher | Evaluates | Example |
|---|---|---|
| Tool events | Tool name | `Bash`, `Edit\|Write`, `mcp__memory__.*` |
| `SessionStart` | Session source | `startup`, `resume`, `clear`, `compact` |
| `Setup` | CLI flag | `init`, `maintenance` |
| `SessionEnd` | End reason | `clear`, `resume`, `logout`, `other` |
| `Notification` | Notification type | `permission_prompt`, `auth_success` |
| `SubagentStart/Stop` | Agent type | `general-purpose`, `Explore`, custom names |
| `PreCompact/PostCompact` | Trigger | `manual`, `auto` |
| `FileChanged` | Filenames | `.envrc\|.env` (literal paths, not globs) |

**Syntax rules:**
- `"*"` or omitted → match all
- Letters/digits/`_` only → exact string match
- Contains other characters → JavaScript regex

## Common hook fields

| Field | Type | Description |
|---|---|---|
| `type` | string | `command`, `http`, `mcp_tool`, `prompt`, `agent` |
| `if` | string | Permission-rule syntax for fine-grained matching (tool events only) |
| `timeout` | number | Seconds before canceling (default: 600 for command/http/mcp_tool, 30 for prompt, 60 for agent) |
| `statusMessage` | string | Message shown while hook runs |
| `once` | boolean | Run once per session then remove |

The `if` field uses permission rule syntax: `Bash(git push *)`, `Edit(*.ts)`, etc. It matches more narrowly than `matcher`.

## Command hook fields

| Field | Type | Description |
|---|---|---|
| `command` | string | Shell command or executable path |
| `args` | array | Argument array — enables exec form (no shell interpretation) |
| `async` | boolean | Run in background, don't wait for result |
| `asyncRewake` | boolean | Background, but exit code 2 wakes Claude |
| `shell` | string | `"bash"` (default) or `"powershell"` |

**Exec form** (`args` present): each element passed as one verbatim argument, no shell expansion. Use for paths with spaces or variables: `"command": "node", "args": ["${CLAUDE_PLUGIN_ROOT}/script.js"]`

**Shell form** (`args` absent): full shell tokenization, pipes, `&&`, variable expansion available.

## HTTP hook fields

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/pre-tool-use",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 600
}
```

- Sends hook input as POST body with `Content-Type: application/json`
- 2xx → success; non-2xx → non-blocking error
- Returns same JSON output format as command hooks

## MCP tool hook fields

```json
{
  "type": "mcp_tool",
  "server": "my_server",
  "tool": "security_scan",
  "input": { "file_path": "${tool_input.file_path}" }
}
```

Tool text output is treated as command-hook stdout; valid JSON output is processed as a decision.

## Input / output protocol

### Common input fields (all events)

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default|plan|acceptEdits|auto|dontAsk|bypassPermissions",
  "effort": { "level": "low|medium|high|xhigh|max" },
  "hook_event_name": "EventName",
  "agent_id": "subagent-id",
  "agent_type": "agent-name"
}
```

### Exit codes

| Code | Effect |
|---|---|
| `0` | Success — stdout parsed for optional JSON output; plain text added as context |
| `2` | Blocking error — stderr sent to Claude; tool call blocked, prompt rejected, etc. |
| Other | Non-blocking error — first stderr line shown in transcript; execution continues |

### JSON output fields (exit 0)

```json
{
  "continue": false,
  "stopReason": "Why stopped",
  "suppressOutput": false,
  "systemMessage": "Warning shown to user",
  "terminalSequence": "\033]777;notify;Title;Body\007",
  "hookSpecificOutput": {
    "hookEventName": "EventName",
    "additionalContext": "Text injected into Claude's context"
  }
}
```

`continue: false` stops Claude entirely. `terminalSequence` accepts OSC 0/1/2/9/99/777 and BEL only.

## Hook events — detailed reference

### PreToolUse

Runs before any tool call. Can block, allow, modify input, or defer to permission rules.

**Extra input fields:**
```json
{
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf /" },
  "tool_use_id": "uuid"
}
```

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "PreToolUse",
  "permissionDecision": "deny|allow|ask|defer",
  "permissionDecisionReason": "Why",
  "modifiedInput": { "command": "safe-alternative" },
  "additionalContext": "Context for Claude"
}
```

**Important**: `permissionDecision: "allow"` from a hook does NOT override explicit deny rules in permissions settings. Deny rules always take precedence. Hook blocking (exit 2) takes precedence over allow rules.

### PostToolUse

Runs after tool succeeds. Cannot un-run the tool, but can block the agentic loop.

**Extra input fields:**
```json
{
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/file.ts" },
  "tool_output": "File written successfully"
}
```

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "PostToolUse",
  "additionalContext": "Lint output appended to context"
}
```

**To stop the loop:** use top-level `"decision": "block"` with `"reason"`.

### PostToolUseFailure

Runs after a tool fails. Same input/output shape as PostToolUse, with `tool_output` containing the error.

### PermissionRequest

Runs when a permission dialog appears. Can auto-approve or deny.

**Extra input fields:**
```json
{
  "tool_name": "Bash",
  "tool_input": { "command": "git push" },
  "description": "Execute a shell command"
}
```

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "PermissionRequest",
  "decision": {
    "behavior": "allow|deny",
    "updatedInput": { "command": "verified-safe-command" },
    "saveAsPermissionRule": "Bash(git push *)"
  }
}
```

### PermissionDenied

Runs when auto-mode classifier denies a tool. Cannot override the denial, but can tell Claude to retry.

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "PermissionDenied",
  "retry": true
}
```

### Stop

Runs when Claude finishes responding. Can block to force continuation.

**Extra input fields:**
```json
{
  "has_tool_calls": false,
  "stop_reason": "end_turn"
}
```

**To block:** top-level `"decision": "block"` with `"reason"`.

### UserPromptSubmit

Runs before Claude processes a user prompt. Default timeout: 30s.

**Extra input fields:**
```json
{ "prompt": "Write a function to calculate factorial" }
```

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "UserPromptSubmit",
  "additionalContext": "Injected context string",
  "sessionTitle": "Optional session title"
}
```

Plain stdout (non-JSON) also adds context without needing `hookSpecificOutput`.

**To block prompt:** top-level `"decision": "block"` with `"reason"`.

### SessionStart

Runs when a session begins or resumes. Only `command` and `mcp_tool` types supported.

**Extra input fields:**
```json
{
  "source": "startup|resume|clear|compact",
  "model": "claude-sonnet-4-6"
}
```

**`hookSpecificOutput` fields:**
```json
{
  "hookEventName": "SessionStart",
  "additionalContext": "Current branch: main\nOpen PRs: 3"
}
```

**Persist env vars** via `$CLAUDE_ENV_FILE`:
```bash
echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
```

### SubagentStart / SubagentStop

Fire when a subagent spawns or finishes. `SubagentStop` is the `Stop` equivalent for subagents.

**Extra input fields:**
```json
{
  "agent_type": "Explore",
  "agent_id": "agent-uuid"
}
```

### Notification

Fires when Claude Code sends a notification. Cannot block. Use for logging/observability.

**Matcher values:** `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_*`

### FileChanged

Fires when a watched file changes. Matcher specifies literal filenames (not globs) to watch.

```json
{
  "matcher": ".envrc|.env",
  "hooks": [{ "type": "command", "command": "./reload-env.sh" }]
}
```

**Extra input fields:**
```json
{
  "file_path": "/home/user/.envrc",
  "change_type": "modified"
}
```

Supports `$CLAUDE_ENV_FILE` for env var persistence.

### CwdChanged

Fires when the working directory changes (e.g., `cd`).

**Extra input fields:**
```json
{
  "previous_cwd": "/old/path",
  "new_cwd": "/new/path"
}
```

Supports `$CLAUDE_ENV_FILE`. Useful with `direnv` for reactive environment management.

### ConfigChange

Fires when a settings file changes during a session.

**Matcher values:** `user_settings`, `project_settings`, `local_settings`, `policy_settings`, `skills`

Can block to prevent the configuration change from taking effect.

### InstructionsLoaded

Fires when `CLAUDE.md` or `.claude/rules/*.md` loads. Observability only — cannot block.

**Matcher values:** `session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact`

**Extra input fields:**
```json
{
  "file_path": "/path/to/CLAUDE.md",
  "memory_type": "Project",
  "load_reason": "session_start",
  "globs": ["src/**"]
}
```

## Environment variables available to hooks

| Variable | Description |
|---|---|
| `$CLAUDE_PROJECT_DIR` | Project root directory |
| `$CLAUDE_PLUGIN_ROOT` | Plugin installation directory |
| `$CLAUDE_PLUGIN_DATA` | Plugin persistent data directory |
| `$CLAUDE_ENV_FILE` | Path to persist env vars (SessionStart, Setup, CwdChanged, FileChanged only) |
| `$CLAUDE_EFFORT` | Current effort level (`low`/`medium`/`high`/`xhigh`/`max`) |
| `$CLAUDE_CODE_REMOTE` | `"true"` in remote web environments |

Plus all environment variables inherited from the parent process.

## Portable path references

Use `$CLAUDE_PROJECT_DIR` to make hook paths relocatable:

```json
{
  "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/validate.sh",
  "args": []
}
```

Or in shell form with quoting: `"\"${CLAUDE_PROJECT_DIR}\"/.claude/hooks/validate.sh"`

## Execution rules

- **Parallel**: all matching hooks for an event run in parallel; identical handlers are deduplicated
- **Timeout defaults**: 600s (command/http/mcp_tool), 30s (prompt), 60s (agent); `UserPromptSubmit` uses 30s
- **Working directory**: wherever the hook process starts
- **Session cleanup**: hooks defined in skill/agent frontmatter are removed when the component finishes

## Disabling hooks

- **Remove** individual hooks: delete from settings JSON
- **Disable all hooks**: set `"disableAllHooks": true` in any settings file (respects managed settings precedence)
- **View hooks**: type `/hooks` in Claude Code

## Full lifecycle order

```
Setup → SessionStart →
  UserPromptSubmit → UserPromptExpansion →
  [Agentic loop:
    PreToolUse → PermissionRequest / PermissionDenied →
    PostToolUse / PostToolUseFailure → PostToolBatch →
    SubagentStart / SubagentStop
  ] →
  Stop / StopFailure
→ SessionEnd

Async (fire independently):
  FileChanged, CwdChanged, ConfigChange,
  InstructionsLoaded, Notification,
  WorktreeCreate / WorktreeRemove
```

## Annotated examples

### Block dangerous Bash commands (PreToolUse, exit 2)

```bash
#!/bin/bash
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // ""')

if echo "$command" | grep -qE 'rm -rf|drop table'; then
  echo "Destructive command blocked" >&2
  exit 2
fi
exit 0
```

### Block via JSON output (PreToolUse, exit 0)

```bash
#!/bin/bash
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // ""')

if echo "$command" | grep -q 'git push'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Direct push is not allowed; open a PR instead"
    }
  }'
else
  exit 0
fi
```

### Auto-approve safe commands (PermissionRequest)

```bash
#!/bin/bash
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // ""')

if [[ "$command" =~ ^(git\ status|npm\ test|ls|pwd)$ ]]; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PermissionRequest",
      decision: { behavior: "allow" }
    }
  }'
else
  exit 0
fi
```

### Inject context for deploy-related prompts (UserPromptSubmit)

```bash
#!/bin/bash
input=$(cat)
prompt=$(echo "$input" | jq -r '.prompt // ""')

if [[ "$prompt" =~ deploy|release ]]; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "UserPromptSubmit",
      additionalContext: "Deployment checklist:\n- All tests passing\n- PR reviewed\n- Changelog updated"
    }
  }'
else
  exit 0
fi
```

### Run linter after file write and block on failure (PostToolUse)

```bash
#!/bin/bash
input=$(cat)
file=$(echo "$input" | jq -r '.tool_input.file_path // ""')

if [[ "$file" =~ \.ts$ ]]; then
  if ! npx eslint "$file" 2>&1; then
    jq -n '{
      decision: "block",
      reason: "ESLint failed — fix errors before continuing"
    }'
    exit 0
  fi
fi
exit 0
```

### Load environment at session start (SessionStart)

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
  echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
fi

jq -n '{
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: "Environment loaded. NODE_ENV=development"
  }
}'
```
