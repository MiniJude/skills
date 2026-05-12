# Skills

Personal agent skills for Codex, Claude Code, Trae, OpenClaw, and other tools that support the Agent Skills format.

This repository keeps only reusable personal skills. Project-specific memory, ports, deployment notes, and troubleshooting rules should stay inside the project that owns them.

## Skills

| Skill | Description |
| --- | --- |
| [`cloud-codex-installer`](skills/cloud-codex-installer) | Interactively installs and verifies Codex on a cloud server. |
| [`frontend-design-preferences`](skills/frontend-design-preferences) | Personal frontend UI/UX preferences for components, loading states, responsive layouts, and interaction consistency. |

## Install

List available skills:

```bash
pnpx skills add MiniJude/skills --list
```

Install all skills globally for Codex:

```bash
pnpx skills add MiniJude/skills --skill '*' -g -a codex --copy -y
```

Install one skill globally for Codex:

```bash
pnpx skills add MiniJude/skills --skill cloud-codex-installer -g -a codex --copy -y
```

Install frontend UI preferences:

```bash
pnpx skills add MiniJude/skills --skill frontend-design-preferences -g -a codex --copy -y
```

Install for other agents:

```bash
pnpx skills add MiniJude/skills --skill '*' -g -a claude-code --copy -y
pnpx skills add MiniJude/skills --skill '*' -g -a trae --copy -y
pnpx skills add MiniJude/skills --skill '*' -g -a openclaw --copy -y
```

Install from a local clone:

```bash
pnpx skills add . --skill '*' -g -a codex --copy -y
```

## Structure

```text
skills/
  <skill-name>/
    SKILL.md
```

Each `SKILL.md` should include front matter with `name` and `description`.

## Add A Skill

1. Create `skills/<skill-name>/SKILL.md`.
2. Keep the skill reusable across projects.
3. Put project-specific details in that project's own `.codex/skills/` directory instead.
4. Verify it is discoverable:

```bash
pnpx skills add . --list
```
