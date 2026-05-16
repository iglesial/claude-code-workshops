# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository designs and delivers Claude Code workshops for companies. The primary artifact being built is `workshops/governance-and-enablement/` — a one-day in-person workshop for 10 people (~5 business, ~5 technical) covering: full-stack governance (CLAUDE.md + settings.json + hooks), knowledge management, commands & skills, MCP, and subagents.

The 10 workshop issues tracking this build are at https://github.com/iglesial/claude-code-workshops/issues (build order: #1 first — all others depend on it).

## Packmind CLI

Skills and standards are distributed via Packmind. Key commands:

```bash
packmind-cli --version                              # check installed version
packmind-cli install                                # sync packmind.json dependencies into .claude/skills/
packmind-cli skills add <path/to/skill-folder>      # register a skill with Packmind
packmind-cli packages add --to <slug> --skill <slug> # add skill to a package
packmind-cli install --list                         # list available packages
```

Dependencies are declared in `packmind.json` and locked in `packmind-lock.json`. After editing `packmind.json`, run `packmind-cli install` to sync.

## Skill Structure

Skills live in `.claude/skills/<skill-name>/` and follow this layout:

```
skill-name/
├── SKILL.md          # required — YAML frontmatter (name, description) + instructions
├── README.md         # human-facing docs
├── LICENSE.txt       # license
├── scripts/          # executable code (Python/Bash) for deterministic tasks
├── references/       # docs Claude loads into context as needed
└── assets/           # files used in output (templates, images) — not loaded into context
```

`SKILL.md` frontmatter fields `name` and `description` determine when Claude auto-triggers the skill — write them precisely. Use third-person in descriptions ("This skill should be used when...").

To create a new skill from scratch, use the init script bundled in `packmind-create-skill`:

```bash
python3 .claude/skills/packmind-create-skill/scripts/init_skill.py <skill-name> --path .claude/skills
```

To validate before distributing:

```bash
python3 .claude/skills/packmind-create-skill/scripts/quick_validate.py <path/to/skill-folder>
```

## GitHub Issues

Issues are managed with the `gh` CLI via the `github-issue-manager` skill. All issues use labels: `bug`, `feature`, `chore`, `docs`. Issue body structure: Summary → Context → Acceptance Criteria → Test Cases → Technical Notes.

```bash
gh issue list --limit 20
gh issue view <NUMBER>
gh issue list --json number,title --jq '.[] | "#\(.number) \(.title)"'
```

## Workshop Content Conventions

Workshop materials live under `workshops/governance-and-enablement/` with this structure:

- `sample-project/` — the fictional Acme Corp project used in all demos (must be self-contained)
- `facilitator/` — one guide per module with exact demo prompts and expected outputs
- `exercises/` — attendee exercise sheets, split into `*-business.md` and `*-technical.md` tracks where diverging
- `take-home/` — reference cards sized for one A4 page per role

All demo prompts in facilitator guides must be tested against `sample-project/` with actual expected output captured verbatim before the guide is considered done.
