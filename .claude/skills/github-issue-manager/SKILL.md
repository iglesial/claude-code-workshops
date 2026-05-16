---
name: 'github-issue-manager'
description: 'Create, read, list, and summarize GitHub issues using the GitHub CLI (gh). Use when the user asks to create issues, read tickets, inspect bugs, summarize issues, or turn requirements into GitHub issues.'
---

# GitHub Issue Manager

Manage GitHub issues using the `gh` CLI. Create well-structured issues, read existing tickets, and track project work.

## Prerequisites

- Current directory is a GitHub repository
- `gh` CLI installed and authenticated
- Repository exists on GitHub

## Operating Principles

- Act autonomously - don't ask for confirmation before creating/reading
- Infer reasonable defaults when information is missing
- Only ask questions if issue number or scope is genuinely ambiguous
- Use professional, clear formatting
- Reference PRD.md or project docs when available for context

---

## Creating Issues

### Issue Types

| Type | Label | Use For |
|------|-------|---------|
| Bug | `bug` | Something broken or not working as expected |
| Feature | `feature` | New functionality or capability |
| Chore | `chore` | Maintenance, refactoring, dependencies |
| Spike | `spike` | Research or investigation task |
| Documentation | `docs` | Documentation improvements |

### Quality Standards

Every issue must include:
- **Title**: Clear, specific, actionable (imperative mood)
- **Summary**: 1-2 sentences explaining the what and why
- **Acceptance Criteria**: Checkboxes defining "done"
- **Test Cases**: Specific scenarios to verify the implementation works correctly
- **Labels**: At least one type label

Optional sections (add when relevant):
- **Context**: Background info, links to PRD sections
- **Technical Notes**: Implementation hints, affected files
- **Dependencies**: Blockers or related issues

### Command Template

```bash
gh issue create \
  --title "<TITLE>" \
  --body "$(cat <<'EOF'
## Summary
<1-2 sentence description>

## Context
<Background, links to PRD sections if relevant>

## Acceptance Criteria
- [ ] <Criterion 1>
- [ ] <Criterion 2>
- [ ] <Criterion 3>

## Test Cases
- <Scenario>: <Expected outcome>
- <Scenario>: <Expected outcome>

## Technical Notes
<Optional: implementation hints, affected files>
EOF
)" \
  --label "<label1>,<label2>"
```

### Examples

**Bug Report:**
```bash
gh issue create \
  --title "Fix credit balance not updating after Stripe webhook" \
  --body "$(cat <<'EOF'
## Summary
User credit balance doesn't reflect purchases until page refresh. The Stripe webhook handler isn't updating DynamoDB in real-time.

## Context
Related to billing flow in PRD.md section 3.3

## Acceptance Criteria
- [ ] Credit balance updates within 5 seconds of successful payment
- [ ] No page refresh required
- [ ] Webhook handler logs success/failure

## Test Cases
- Successful payment via Stripe: balance updates without refresh within 5s
- Failed payment webhook: balance unchanged, error logged
- Duplicate webhook delivery: balance only incremented once

## Technical Notes
Check `src/api/webhooks/stripe.ts` and DynamoDB update logic.
EOF
)" \
  --label "bug"
```

**Feature Request:**
```bash
gh issue create \
  --title "Add model deployment progress indicator" \
  --body "$(cat <<'EOF'
## Summary
Show real-time deployment progress when user deploys a model, instead of just a spinner.

## Context
PRD.md Flow 2 & 3 - Deploy Model

## Acceptance Criteria
- [ ] Progress bar shows stages: Validating → Building → Deploying → Ready
- [ ] Estimated time remaining displayed
- [ ] User can navigate away and return to see progress
- [ ] Error state shows which stage failed

## Test Cases
- Deploy model successfully: progress bar advances through all stages to Ready
- Deploy model with build failure: progress stops at Building, error message shown
- Navigate away and return mid-deploy: progress resumes at correct stage
- Deploy two models concurrently: each shows independent progress

## Technical Notes
Will need polling or WebSocket for status updates from infrastructure.
EOF
)" \
  --label "feature"
```

---

## Reading Issues

### View Single Issue

```bash
gh issue view <NUMBER>
```

With structured data:
```bash
gh issue view <NUMBER> --json title,body,state,labels,assignees,author,comments
```

### List Issues

List open issues:
```bash
gh issue list
```

With filters:
```bash
# By label
gh issue list --label "bug"

# By state
gh issue list --state closed

# By assignee
gh issue list --assignee "@me"

# Limit results
gh issue list --limit 20

# Combined
gh issue list --label "feature" --state open --limit 10
```

### Search Issues

```bash
gh issue list --search "deployment"
```

---

## Summarizing Issues

When asked to summarize issues, provide:

1. **Status Overview**: Open/closed count, labels distribution
2. **Priority Items**: Bugs and blockers first
3. **Themes**: Group related issues
4. **Actionable Next Steps**: What to work on next

Example output format:
```
## Issue Summary

**Open:** 12 | **Closed:** 34

### By Type
- Bugs: 3 (1 critical)
- Features: 7
- Chores: 2

### Priority Items
1. #45 - Fix auth token expiration (bug, critical)
2. #42 - Credit balance sync issue (bug)

### Themes
- **Authentication**: #45, #38, #31
- **Billing**: #42, #40, #39

### Recommended Next
Start with #45 (critical bug blocking users)
```

---

## Batch Operations

### Create Multiple Issues from PRD

When asked to create issues from a PRD or spec:

1. Read the PRD file
2. Identify distinct work items
3. Group into logical issues (not too granular)
4. Create issues with references back to PRD sections
5. Report created issue numbers

```bash
# After creating, list what was made:
gh issue list --limit 10 --json number,title --jq '.[] | "#\(.number) \(.title)"'
```

### Close Issues

```bash
gh issue close <NUMBER>
gh issue close <NUMBER> --comment "Completed in PR #123"
```

### Edit Issues

```bash
gh issue edit <NUMBER> --title "New title"
gh issue edit <NUMBER> --add-label "priority"
gh issue edit <NUMBER> --add-assignee "@me"
```

---

## Project Context

When creating issues for this project (Fluffy Cake):

- Reference PRD.md sections when relevant
- Use labels: `bug`, `feature`, `chore`, `spike`, `docs`
- Consider MVP scope (model deployment/inference focus)
- Tag with component: `auth`, `billing`, `models`, `ui`

---

## Troubleshooting

**"gh: command not found"**
```bash
# Install gh CLI
# macOS: brew install gh
# Windows: winget install GitHub.cli
# Linux: see https://github.com/cli/cli/blob/trunk/docs/install_linux.md
```

**"not logged in"**
```bash
gh auth login
```

**"repository not found"**
```bash
# Ensure you're in a git repo with GitHub remote
git remote -v
```