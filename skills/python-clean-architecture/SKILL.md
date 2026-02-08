---
name: python-clean-architecture
description: |
  A comprehensive set of 30 rules for building maintainable, type-safe Python applications
  using clean architecture principles. Covers service layer isolation with bounded contexts,
  strict dependency injection, complete type safety with no Any or cast usage, immutable
  data transfer objects, disciplined project structure, and clean code patterns. Designed
  for production Python codebases that prioritize long-term maintainability and correctness.
license: MIT
metadata:
  author: python-best-practices
  version: '1.0.0'
---

# Python Clean Architecture

A skill package of 30 rules for building Python applications with clean architecture, strict typing, and domain-driven design principles. These rules enforce bounded context isolation, dependency injection, immutable data flow, and disciplined project structure.

## When to Apply

- Building new Python services or microservices from scratch
- Refactoring monolithic Python applications into bounded contexts
- Reviewing pull requests for architectural compliance
- Setting up dependency injection and service layer patterns
- Defining typed DTOs and data flow between layers
- Structuring Python packages for long-term maintainability
- Enforcing strict type safety across a codebase

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| CRITICAL | Service Layer & Bounded Contexts | Domain alignment and isolation | `service-` |
| CRITICAL | Dependency Injection | Explicit, testable dependency graphs | `di-` |
| CRITICAL | Strict Typing | Complete type safety, no escape hatches | `typing-` |
| HIGH | Data Transfer Objects | Immutable, typed data flow | `dto-` |
| HIGH | Project Structure & Isolation | Navigable, decoupled packages | `structure-` |
| MEDIUM | Clean Code | Readable, maintainable patterns | `clean-` |

## Quick Reference

### 1. Service Layer & Bounded Contexts (5 rules)
- `service-bounded-context` — Organize code by domain/bounded context, not by technical layer (CRITICAL)
- `service-layer-isolation` — Services communicate only through service interfaces, never through repos/clients directly (CRITICAL)
- `service-no-cross-repo` — A service never imports or accesses another bounded context's repository (CRITICAL)
- `service-interface-contracts` — Define Protocol/ABC for every service interface (CRITICAL)
- `service-single-responsibility` — Each service handles one bounded context's business logic (HIGH)

### 2. Dependency Injection (5 rules)
- `di-constructor-injection` — Inject dependencies via __init__, never instantiate inside methods (CRITICAL)
- `di-depend-on-abstractions` — Depend on Protocol/ABC, never on concrete implementations (CRITICAL)
- `di-container-composition-root` — Wire dependencies at the composition root only (HIGH)
- `di-no-service-locator` — Never use service locator pattern; always inject explicitly (HIGH)
- `di-scoped-lifecycle` — Match dependency lifecycle to scope (request/singleton/transient) (MEDIUM)

### 3. Strict Typing (8 rules)
- `typing-no-any` — Never use Any; use proper generics, Protocol, or Union instead (CRITICAL)
- `typing-no-cast` — Never use cast(); use isinstance type narrowing or overloads (CRITICAL)
- `typing-explicit-return` — Always annotate return types on all functions/methods (CRITICAL)
- `typing-explicit-params` — Always annotate all function parameters with types (CRITICAL)
- `typing-protocol-structural` — Use Protocol for structural subtyping over ABC where possible (HIGH)
- `typing-generic-containers` — Use Generic[T] and TypeVar for reusable typed containers (HIGH)
- `typing-newtype-ids` — Use NewType for domain identifiers to prevent type confusion (MEDIUM)
- `typing-final-constants` — Use Final for constants that should never be reassigned (MEDIUM)

### 4. Data Transfer Objects (5 rules)
- `dto-frozen-dataclass` — Use @dataclass(frozen=True, slots=True) for all DTOs (CRITICAL)
- `dto-no-raw-dicts` — Never pass raw dicts between layers; always use typed DTOs (CRITICAL)
- `dto-per-layer` — Separate DTOs for API/service/persistence layers (HIGH)
- `dto-factory-classmethods` — Use @classmethod factories to construct DTOs from different sources (HIGH)
- `dto-validation-at-boundary` — Validate data at system boundaries, trust internal DTOs (MEDIUM)

### 5. Project Structure & Isolation (4 rules)
- `structure-domain-packages` — Top-level packages = bounded contexts (users/, orders/, payments/) (HIGH)
- `structure-explicit-exports` — Use __all__ in every __init__.py to control public API (HIGH)
- `structure-no-circular-imports` — Prevent circular imports through proper layer ordering (HIGH)
- `structure-typed-settings` — Centralize config in a typed dataclass/pydantic Settings, no raw env reads (HIGH)

### 6. Clean Code (3 rules)
- `clean-composition-over-inheritance` — Prefer composition and delegation over deep inheritance (MEDIUM)
- `clean-early-return` — Use guard clauses; return/raise early to reduce nesting (MEDIUM)
- `clean-custom-exceptions` — Define domain-specific exception hierarchy; never bare except: (MEDIUM)

## How to Use

Apply these rules when writing or reviewing Python code in projects that follow clean architecture principles. Rules are ordered by impact -- start with CRITICAL rules to establish the architectural foundation, then layer in HIGH and MEDIUM rules for refinement.

Each rule includes incorrect and correct code examples. Use the prefix to quickly identify which category a rule belongs to when referencing in code reviews or linter configurations.

## Full Compiled Document

For the complete expanded reference with all 30 rules and their full code examples, see [AGENTS.md](./AGENTS.md).
