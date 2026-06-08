# .origin

This folder contains Origin CDE workspace customizations for AI agents.

## Structure

```
.origin/
	agents/     - Custom AI agents (.agent.md or AGENTS.md files)
	skills/     - Reusable skill definitions (each skill is a folder with SKILL.md)
```

## Agents

Place `.agent.md` files in `.origin/agents/` to define custom AI agents
that are automatically available in the Origin CDE chat panel.

Agents can declare required skills in their frontmatter:

```yaml
---
name: MyAgent
description: My custom agent
skills:
	- launch
	- my-skill
---
```

## Skills

Each skill lives in its own subfolder and must contain a `SKILL.md` file:

```
.origin/skills/
	my-skill/
		SKILL.md    - Skill instructions and metadata
```

Skill `SKILL.md` frontmatter:

```yaml
---
name: my-skill
description: "What this skill does"
---
```
