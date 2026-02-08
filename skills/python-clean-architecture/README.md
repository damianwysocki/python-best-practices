# Python Clean Architecture

A skill package of 30 rules for building Python applications with clean architecture, strict typing, and domain-driven design principles.

## What This Covers

These rules enforce bounded context isolation, explicit dependency injection, complete type annotations (no `Any`, no `cast`), immutable data transfer objects, disciplined project structure, and clean code patterns. They are designed for production Python codebases that prioritize long-term maintainability and correctness.

## Categories

| # | Category | Prefix | Rules | Impact |
|---|----------|--------|-------|--------|
| 1 | Service Layer & Bounded Contexts | `service-` | 5 | CRITICAL |
| 2 | Dependency Injection | `di-` | 5 | CRITICAL |
| 3 | Strict Typing | `typing-` | 8 | CRITICAL |
| 4 | Data Transfer Objects | `dto-` | 5 | HIGH |
| 5 | Project Structure & Isolation | `structure-` | 4 | HIGH |
| 6 | Clean Code | `clean-` | 3 | MEDIUM |

## How to Use

- **Quick lookup:** Open [SKILL.md](./SKILL.md) for a concise table of all 30 rules with one-line descriptions, organized by category and impact level.
- **Full guide:** Open [AGENTS.md](./AGENTS.md) for the complete expanded reference with detailed explanations and Python code examples for every rule.
- **Individual rules:** Browse the `rules/` directory. Each file is a standalone rule with frontmatter metadata, explanation, incorrect/correct code examples, and a reference link.
- **Sections overview:** See [rules/_sections.md](./rules/_sections.md) for section definitions, ordering, and impact levels.

## Impact Levels

| Level | Meaning |
|-------|---------|
| **CRITICAL** | Foundational rules. Violating these undermines the entire architecture. |
| **HIGH** | Important rules that prevent common structural and coupling problems. |
| **MEDIUM** | Refinement rules that improve readability and maintainability. |

## Adding a New Rule

1. Copy `rules/_template.md` to a new file named `{prefix}-{slug}.md` (e.g., `typing-no-untyped-defs.md`).
2. Fill in the frontmatter: `title`, `impact`, `impactDescription`, and `tags`.
3. Write the rule explanation, then add **Incorrect** and **Correct** Python code examples.
4. Add a `Reference:` link at the bottom pointing to relevant documentation.
5. Update the Quick Reference table in `SKILL.md` and the corresponding section in `AGENTS.md`.

## File Structure

```
skills/python-clean-architecture/
  metadata.json          # Package metadata and references
  README.md              # This file
  SKILL.md               # Quick reference with frontmatter
  AGENTS.md              # Full compiled document with all rules
  rules/
    _sections.md         # Section definitions and ordering
    _template.md         # Template for new rules
    service-*.md         # Service layer rules (5)
    di-*.md              # Dependency injection rules (5)
    typing-*.md          # Strict typing rules (8)
    dto-*.md             # Data transfer object rules (5)
    structure-*.md       # Project structure rules (4)
    clean-*.md           # Clean code rules (3)
```

## License

MIT
