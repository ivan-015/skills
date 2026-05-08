# skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills.

## Layout

Each skill lives in its own directory at the repo root:

```
skills/
  <skill-name>/
    SKILL.md          # required — frontmatter + instructions
    ...               # optional supporting files (scripts, templates, refs)
```

A `SKILL.md` starts with YAML frontmatter:

```markdown
---
name: skill-name
description: One-line description used to decide when to invoke the skill.
---

# Skill body
Instructions Claude follows when this skill is invoked.
```

## Using these skills

Clone or symlink individual skill directories into `~/.claude/skills/` (user-level)
or `<project>/.claude/skills/` (project-level), then reference them via `/<skill-name>`
inside Claude Code.
