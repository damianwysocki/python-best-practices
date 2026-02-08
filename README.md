# Python Agent Skills

A collection of AI coding agent skills for Python best practices. These skills provide packaged instructions that extend AI agent capabilities with deep domain knowledge about Python architecture, frameworks, and patterns.

Skills follow the [Agent Skills](https://github.com/vercel-labs/agent-skills) format — each contains a `SKILL.md` definition, compiled `AGENTS.md` guide, and individual rule files.

## Skills

### [python-clean-architecture](skills/python-clean-architecture)

30 rules across 6 categories covering clean architecture patterns, strict typing, dependency injection, DTOs, and project structure. Prioritized from CRITICAL (service boundaries, typing) to MEDIUM (clean code patterns).

### [python-fastapi-best-practices](skills/python-fastapi-best-practices)

28 rules across 7 categories for FastAPI applications. Covers OpenAPI documentation, Pydantic models, dependency injection, async patterns, router architecture, security, and performance.

### [python-django-best-practices](skills/python-django-best-practices)

23 rules across 5 categories for Django and Django REST Framework. Addresses service layer integration, model design, DRF patterns, query optimization, and testing.

### [python-sqlalchemy-best-practices](skills/python-sqlalchemy-best-practices)

19 rules across 5 categories for SQLAlchemy 2.0. Covers session management, model design with typed mapped columns, query patterns, migrations, and engine configuration.

## Installation

### Requirements

- **Claude Code CLI** with agent skills support (requires `skills.sh`)
- **npm/npx** installed for skill package management

### Recommended: Install via npx

The fastest way to install these skills is using npx:

```bash
npx skill add damianwysocki/python-best-practices
```

This installs all four skills to your `~/.claude/skills/` directory.

### Alternative: Manual Installation

If you prefer to manually copy skills to your project:

```bash
cp -r skills/python-clean-architecture ~/.claude/skills/
cp -r skills/python-fastapi-best-practices ~/.claude/skills/
cp -r skills/python-django-best-practices ~/.claude/skills/
cp -r skills/python-sqlalchemy-best-practices ~/.claude/skills/
```

### As Context

You can also add the `AGENTS.md` file from any skill directly to your project's context without installing as a skill.

## Structure

Each skill follows a consistent format:

```
skill-name/
├── SKILL.md          # Skill definition with frontmatter and quick reference
├── AGENTS.md         # Compiled full guide with all rules expanded inline
├── README.md         # Human-readable documentation
├── metadata.json     # Version, organization, abstract, references
└── rules/
    ├── _sections.md  # Section definitions with impact levels
    ├── _template.md  # Template for creating new rules
    └── *.md          # Individual rule files (prefix-name.md)
```

## License

MIT
