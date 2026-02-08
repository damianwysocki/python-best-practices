# Python Django Best Practices

A skill package containing 23 best-practice rules for building maintainable, performant, and testable Django applications with Django REST Framework.

## What This Covers

This package codifies production-tested patterns for Django and DRF projects. It enforces a clean architecture where views handle HTTP, services handle business logic, and models handle persistence. Every rule includes incorrect and correct code examples with explanations.

## Rule Categories

| #   | Category                  | Prefix     | Impact   | Rules |
| --- | ------------------------- | ---------- | -------- | ----- |
| 1   | Service Layer Integration | `service-` | CRITICAL | 4     |
| 2   | Model Design              | `model-`   | CRITICAL | 5     |
| 3   | DRF Patterns              | `drf-`     | HIGH     | 6     |
| 4   | Query Optimization        | `query-`   | HIGH     | 5     |
| 5   | Testing                   | `test-`    | MEDIUM   | 3     |

## Package Structure

```
python-django-best-practices/
  SKILL.md            -- Skill definition with summary and quick reference
  AGENTS.md           -- Full compiled document with all rules expanded inline
  metadata.json       -- Version, abstract, and reference links
  README.md           -- This file
  rules/
    _sections.md      -- Section definitions, ordering, and impact levels
    _template.md      -- Template for creating new rules
    service-*.md      -- Service layer integration rules (4 files)
    model-*.md        -- Model design rules (5 files)
    drf-*.md          -- DRF pattern rules (6 files)
    query-*.md        -- Query optimization rules (5 files)
    test-*.md         -- Testing rules (3 files)
```

## How to Use

**For AI agents and LLMs:** Point your agent at `SKILL.md` for the quick reference or `AGENTS.md` for the full compiled guide with all code examples.

**In code reviews:** Reference rules by prefix (e.g., "This violates `service-not-in-views`").

**In new projects:** Start with Service Layer Integration and Model Design to establish clean architecture from day one.

**In existing projects:** Prioritize Query Optimization for immediate performance wins, then incrementally adopt service layer patterns.

## How to Add a New Rule

1. Copy `rules/_template.md` to a new file using the appropriate prefix (e.g., `rules/drf-new-pattern.md`).
2. Fill in the frontmatter fields: `title`, `impact`, `impactDescription`, and `tags`.
3. Write the rule body with an explanation, an incorrect example, and a correct example.
4. Add the rule to the quick reference list in `SKILL.md` under the matching category.
5. Append the full rule content to `AGENTS.md` in the correct section.
6. If creating a new category, add it to `rules/_sections.md` first.

## References

- [Django Documentation](https://docs.djangoproject.com/en/5.0/)
- [Django REST Framework](https://www.django-rest-framework.org)
- [drf-spectacular](https://drf-spectacular.readthedocs.io)
- [django-filter](https://django-filter.readthedocs.io)
- [factory_boy](https://factoryboy.readthedocs.io)

## License

MIT
