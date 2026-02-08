# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Session Management (session)

**Impact:** CRITICAL
**Description:** Correct session lifecycle management prevents resource leaks and data corruption. Always use context managers, never global sessions, and commit once per unit of work.

## 2. Model Design (model)

**Impact:** CRITICAL
**Description:** SQLAlchemy 2.0 models use DeclarativeBase with Mapped[T] type annotations and mapped_column() for full type safety and IDE support.

## 3. Query Patterns (query)

**Impact:** HIGH
**Description:** Modern select() statement API, explicit eager loading, typed repository classes, and proper result typing ensure correct and performant database access.

## 4. Migration Patterns (migration)

**Impact:** MEDIUM
**Description:** Alembic migrations with autogenerate, reversible upgrades, and separated data migrations keep database schema evolution safe and predictable.

## 5. Connection & Engine (engine)

**Impact:** MEDIUM
**Description:** Production engine configuration with proper pool sizing, recycling, and consistent naming conventions ensures reliability under load.
