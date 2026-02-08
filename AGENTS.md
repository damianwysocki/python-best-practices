# Python Agent Skills — Repository Guide

This document helps AI coding agents work with the Python agent skills repository.

## Directory Structure

```
skills/
├── python-clean-architecture/       # 30 rules — architecture, typing, DI, DTOs
├── python-fastapi-best-practices/   # 28 rules — FastAPI, Pydantic, async
├── python-django-best-practices/    # 23 rules — Django, DRF, query optimization
└── python-sqlalchemy-best-practices/ # 19 rules — SQLAlchemy 2.0, sessions, migrations
```

## Skill Format

Each skill directory contains:

| File | Purpose |
|------|---------|
| `SKILL.md` | Skill definition with YAML frontmatter, categories, and quick reference |
| `AGENTS.md` | Compiled full guide with all rules expanded inline |
| `README.md` | Human-readable documentation and usage guide |
| `metadata.json` | Version, organization, abstract, and references |
| `rules/_sections.md` | Section definitions with impact levels |
| `rules/_template.md` | Template for creating new rules |
| `rules/*.md` | Individual rule files with YAML frontmatter |

## SKILL.md Format

```yaml
---
name: skill-name
description: |
  What triggers this skill and when to use it.
license: MIT
metadata:
  author: python-best-practices
  version: '1.0.0'
---
```

Always uppercase, always this exact filename: `SKILL.md`.

## Rule File Format

Rule files live in `rules/` and use the naming convention `{section-prefix}-{rule-name}.md`:

```yaml
---
title: Rule Title Here
impact: CRITICAL|HIGH|MEDIUM|LOW
impactDescription: 2-5 word metric
tags: tag1, tag2, tag3
---

## Rule Title Here

Explanation of the rule and why it matters.

**Incorrect (description):**

\```python
# Bad example
\```

**Correct (description):**

\```python
# Good example
\```

Reference: [Link](https://example.com)
```

## Creating a New Rule

1. Copy `rules/_template.md` to `rules/{prefix}-{name}.md`
2. Fill in YAML frontmatter: title, impact, impactDescription, tags
3. Write explanation, incorrect example, correct example
4. Add the rule to `rules/_sections.md` under its category
5. Update `SKILL.md` quick reference
6. Rebuild `AGENTS.md` with all rules expanded inline

## Context Efficiency

- Keep individual rule files under 80 lines
- Use `SKILL.md` for quick reference (progressive disclosure)
- Use `AGENTS.md` when the full guide is needed
- Prefer specific file references over loading everything

## Naming Conventions

- Skill directories: `kebab-case` (e.g., `python-clean-architecture`)
- Rule files: `{prefix}-{name}.md` (e.g., `di-constructor-injection.md`)
- Section prefixes match category (e.g., `service-`, `typing-`, `dto-`)
