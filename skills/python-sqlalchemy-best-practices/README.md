# Python SQLAlchemy Best Practices

A skill package containing 19 battle-tested rules for writing modern, type-safe SQLAlchemy 2.0+ code. Designed for AI agents, LLMs, and developers who need actionable guidance on database access layers in Python.

## What This Covers

This package encodes best practices across the full SQLAlchemy stack: session lifecycle management, model design with `Mapped` type annotations, query composition using the `select()` API, Alembic migration strategies, and production engine configuration. Each rule includes an explanation of why it matters, an incorrect code example showing the anti-pattern, and a correct code example showing the recommended approach.

## Categories

| # | Category | Prefix | Rules | Impact |
|---|----------|--------|-------|--------|
| 1 | Session Management | `session-` | 4 | CRITICAL |
| 2 | Model Design | `model-` | 5 | CRITICAL |
| 3 | Query Patterns | `query-` | 5 | HIGH |
| 4 | Migration Patterns | `migration-` | 3 | MEDIUM |
| 5 | Connection & Engine | `engine-` | 2 | MEDIUM |

**Total: 19 rules**

## File Structure

```
skills/python-sqlalchemy-best-practices/
  SKILL.md              # Machine-readable skill definition with frontmatter
  AGENTS.md             # All 19 rules expanded inline (single document)
  metadata.json         # Version, references, and abstract
  README.md             # This file
  rules/
    _sections.md        # Section definitions, ordering, and impact levels
    _template.md        # Template for creating new rules
    session-*.md        # Session Management rules
    model-*.md          # Model Design rules
    query-*.md          # Query Patterns rules
    migration-*.md      # Migration Patterns rules
    engine-*.md         # Connection & Engine rules
```

## How to Use

1. **By priority:** Start with CRITICAL rules (`session-`, `model-`), then HIGH (`query-`), then MEDIUM (`migration-`, `engine-`).
2. **By category:** Browse rules by prefix to focus on a specific area.
3. **Full reference:** Read `AGENTS.md` for all 19 rules in a single document.
4. **Individual rules:** Each file in `rules/` is self-contained with frontmatter, explanation, and code examples.

## Adding New Rules

1. Copy `rules/_template.md` and rename it with the appropriate section prefix (e.g., `query-new-rule.md`).
2. Fill in the YAML frontmatter: `title`, `impact`, `tags`.
3. Write the explanation, incorrect example, and correct example.
4. Add the rule to the appropriate section in `SKILL.md` and `AGENTS.md`.
5. Update `_sections.md` if the rule count changes.

## References

- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)
- [Alembic Migration Tool](https://alembic.sqlalchemy.org)
- [SQLAlchemy Async I/O](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [SQLAlchemy Mapped Attributes](https://docs.sqlalchemy.org/en/20/orm/mapped_attributes.html)
