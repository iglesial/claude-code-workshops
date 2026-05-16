---
name: cc-hooks
description: 'This skill should be used when configuring Claude Code hooks: writing PreToolUse, PostToolUse, Stop, UserPromptSubmit, SessionStart, Notification, FileChanged, or other lifecycle hooks in settings.json; implementing hook scripts (command, HTTP, MCP, prompt, agent types); understanding the stdin JSON input / stdout JSON output / exit code protocol; blocking or allowing tool calls from hooks; injecting context into sessions; or explaining how hooks interact with permissions and sandboxing.'
---

# Claude Code Hooks

## Purpose

Guide users in configuring and implementing Claude Code hooks — shell commands, HTTP endpoints, MCP tools, prompts, or agents that fire at lifecycle events to observe, block, modify, or extend Claude's behavior.

## Reference material

Load `references/hooks.md` for the complete hooks reference: all event types, matcher syntax, input/output protocol, exit codes, JSON output schema, per-event `hookSpecificOutput` fields, environment variables, and annotated script examples.

## Workflow

### 1. Identify the goal

Map the user's intent to the right hook event:

| Goal | Hook event |
|---|---|
| Block / allow a tool call before it runs | `PreToolUse` |
| React after a tool succeeds | `PostToolUse` |
| React after a tool fails | `PostToolUseFailure` |
| Auto-approve or deny a permission dialog | `PermissionRequest` |
| Run checks before Claude finishes | `Stop` |
| Inject context or validate before prompt | `UserPromptSubmit` |
| Load environment at session start | `SessionStart` |
| Watch a file for changes | `FileChanged` |
| React to directory changes | `CwdChanged` |
| Observe notifications (logging only) | `Notification` |

### 2. Choose the correct settings file

| Scope | File |
|---|---|
| Organization (non-overridable) | Managed settings |
| Project (shared with team) | `.claude/settings.json` |
| Project (personal) | `.claude/settings.local.json` |
| User (all projects) | `~/.claude/settings.json` |

Hooks load only from the project `.claude/` folder and `~/.claude/settings.json` — not from parent directories or `--add-dir` paths (except plugins).

### 3. Write the settings.json entry

Basic shape — every hook has a matcher at the outer level and one or more handler objects inside:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-bash.sh"
          }
        ]
      }
    ]
  }
}
```

Use the `if` field for finer matching (permission-rule syntax):

```json
{
  "type": "command",
  "command": "...",
  "if": "Bash(git push *)"
}
```

### 4. Implement the hook script

All `command` hooks receive JSON on stdin and communicate via exit code + stdout:

| Exit code | Effect |
|---|---|
| `0` | Success — parse stdout for optional JSON output |
| `2` | Blocking error — stderr sent to Claude, tool call blocked |
| Any other | Non-blocking error — first stderr line shown in transcript |

Minimal blocking hook pattern:

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

To send a structured decision instead of just an exit code, write JSON to stdout:

```bash
jq -n '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "deny",
    permissionDecisionReason: "Database writes are not allowed"
  }
}'
```

### 5. Key patterns to know

- **`permissionDecision`** (`deny`/`allow`/`ask`/`defer`) — PreToolUse only; does NOT override explicit deny rules in permissions settings
- **`decision: "block"`** — PostToolUse / Stop; stops the agentic loop before the next model call
- **`additionalContext`** — inject text into Claude's context window (UserPromptSubmit, SessionStart, PostToolUse, etc.)
- **`CLAUDE_ENV_FILE`** — append `export VAR=value` lines here to persist env vars across tool calls (SessionStart, FileChanged, CwdChanged)
- **Exec form** (use `args` array) — avoids shell interpretation, pass paths safely; **shell form** (no `args`) — enables pipes, `&&`, variable expansion
- **Parallel execution** — all matching hooks run in parallel; deduplicated if identical

Reference `references/hooks.md` for the full input schema for each event, all `hookSpecificOutput` fields, HTTP/MCP/prompt/agent hook types, and the complete lifecycle diagram.
