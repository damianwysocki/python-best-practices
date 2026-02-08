# Python FastAPI Best Practices

A skill package containing 28 battle-tested rules for building production-grade FastAPI applications. Designed for AI agents, LLMs, and developers who need actionable guidance on async API development with Python.

## What This Covers

This package encodes best practices across the full FastAPI stack: OpenAPI documentation, dependency injection, Pydantic model design, router architecture, async patterns, security, and performance optimization. Each rule includes an explanation of why it matters, an incorrect code example showing the anti-pattern, and a correct code example showing the recommended approach.

## Categories

| # | Category | Prefix | Rules | Impact |
|---|----------|--------|-------|--------|
| 1 | OpenAPI Documentation | `api-` | 6 | CRITICAL |
| 2 | Dependency Injection | `di-` | 4 | CRITICAL |
| 3 | Pydantic Models | `schema-` | 5 | HIGH |
| 4 | Router Architecture | `router-` | 4 | HIGH |
| 5 | Async Patterns | `async-` | 4 | HIGH |
| 6 | Security & Validation | `security-` | 3 | MEDIUM |
| 7 | Performance | `perf-` | 2 | MEDIUM |

**Total: 28 rules**

## File Structure

```
skills/python-fastapi-best-practices/
  SKILL.md              # Machine-readable skill definition with frontmatter
  AGENTS.md             # All 28 rules expanded inline (single document)
  metadata.json         # Version, references, and abstract
  README.md             # This file
  rules/
    _sections.md        # Section definitions, ordering, and impact levels
    _template.md        # Template for creating new rules
    api-*.md            # OpenAPI Documentation rules
    di-*.md             # Dependency Injection rules
    schema-*.md         # Pydantic Models rules
    router-*.md         # Router Architecture rules
    async-*.md          # Async Patterns rules
    security-*.md       # Security & Validation rules
    perf-*.md           # Performance rules
```

## How to Use

1. **By priority:** Start with CRITICAL rules (`api-`, `di-`), then HIGH, then MEDIUM.
2. **By category:** Browse rules by prefix to focus on a specific area.
3. **Full reference:** Read `AGENTS.md` for all 28 rules in a single document.
4. **Individual rules:** Each file in `rules/` is self-contained with frontmatter, explanation, and code examples.

## Adding New Rules

1. Copy `rules/_template.md` and rename it with the appropriate section prefix (e.g., `api-new-rule.md`).
2. Fill in the YAML frontmatter: `title`, `impact`, `tags`.
3. Write the explanation, incorrect example, and correct example.
4. Add the rule to the appropriate section in `SKILL.md` and `AGENTS.md`.
5. Update `_sections.md` if the rule count changes.

## References

- [FastAPI Documentation](https://fastapi.tiangolo.com)
- [Pydantic Documentation](https://docs.pydantic.dev/latest/)
- [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html)
- [Starlette Documentation](https://www.starlette.io)
- [OpenAPI Specification](https://swagger.io/specification/)
