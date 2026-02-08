# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Service Layer & Bounded Contexts (service)

**Impact:** CRITICAL
**Description:** Rules for organizing code around domain bounded contexts and isolating service communication through well-defined interfaces. Ensures each bounded context owns its full vertical slice and prevents cross-context coupling through repositories or internal implementation details.

---

## 2. Dependency Injection (di)

**Impact:** CRITICAL
**Description:** Rules for explicit, testable dependency graphs using constructor injection and abstraction-based wiring. Covers depending on Protocol/ABC rather than concrete implementations, centralizing wiring at a composition root, and avoiding anti-patterns like service locators.

---

## 3. Strict Typing (typing)

**Impact:** CRITICAL
**Description:** Rules for achieving complete type safety across the codebase with no escape hatches. Prohibits Any and cast(), requires explicit return and parameter annotations, and promotes Protocol for structural subtyping, Generic[T] for reusable containers, NewType for domain identifiers, and Final for constants.

---

## 4. Data Transfer Objects (dto)

**Impact:** HIGH
**Description:** Rules for immutable, typed data flow between architectural layers. Requires frozen dataclasses for all DTOs, prohibits raw dicts, enforces separate DTOs per layer, uses classmethod factories for construction, and validates data at system boundaries.

---

## 5. Project Structure & Isolation (structure)

**Impact:** HIGH
**Description:** Rules for navigable, decoupled Python packages organized by business domain. Covers top-level packages mirroring bounded contexts, explicit exports via __all__, preventing circular imports through strict dependency ordering, and centralizing configuration in typed settings.

---

## 6. Clean Code (clean)

**Impact:** MEDIUM
**Description:** Rules for readable, maintainable code patterns that reduce complexity. Covers preferring composition over deep inheritance hierarchies, using guard clauses with early returns to reduce nesting, and defining domain-specific exception hierarchies for precise error handling.
