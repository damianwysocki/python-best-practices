---
name: python-sqlalchemy-best-practices
description: |
  Comprehensive best practices for SQLAlchemy 2.0+ covering session management,
  model design with type annotations, query patterns using the modern select() API,
  Alembic migration strategies, and production engine configuration. These 19 rules
  ensure type-safe, performant, and maintainable database code in Python applications.
license: MIT
metadata:
  author: python-best-practices
  version: '1.0.0'
---

# Python SQLAlchemy Best Practices

A curated set of 19 rules for writing modern, type-safe SQLAlchemy 2.0+ code. These
rules cover the full lifecycle of database interaction: engine configuration, session
management, model design with `Mapped` annotations, query composition using the
`select()` API, and schema evolution with Alembic. Each rule includes incorrect and
correct Python code examples drawn from real-world patterns.

## When to Apply

- When building new Python applications that use SQLAlchemy for database access
- When migrating from SQLAlchemy 1.x to the 2.0 API style
- When reviewing pull requests that touch ORM models, queries, or migrations
- When designing the data access layer for FastAPI, Flask, or other web frameworks
- When configuring SQLAlchemy engines and connection pools for production deployment
- When setting up Alembic for schema migration management

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| CRITICAL | Session Management | Prevents connection leaks and thread-safety bugs | `session-` |
| CRITICAL | Model Design | Enables type-safe, modern ORM models | `model-` |
| HIGH | Query Patterns | Eliminates N+1 queries and ensures type-safe results | `query-` |
| MEDIUM | Migration Patterns | Ensures safe, reversible schema evolution | `migration-` |
| MEDIUM | Connection & Engine | Prevents connection exhaustion in production | `engine-` |

## Quick Reference

### 1. Session Management

- **session-context-manager** (CRITICAL) -- Always use context manager (`with Session`) for session lifecycle
- **session-no-global** (CRITICAL) -- Never use global/module-level sessions; inject via dependency injection
- **session-unit-of-work** (HIGH) -- One session = one unit of work; commit once at the end
- **session-async-engine** (HIGH) -- Use `create_async_engine` + `AsyncSession` for async applications

### 2. Model Design

- **model-declarative-base** (CRITICAL) -- Use `DeclarativeBase` with type-annotated `mapped_column` (SQLAlchemy 2.0 style)
- **model-mapped-column** (CRITICAL) -- Use `Mapped[type]` + `mapped_column()` for all columns
- **model-relationship-typed** (HIGH) -- Type all relationships with `Mapped[list[Child]]` or `Mapped[Parent | None]`
- **model-mixins-timestamps** (HIGH) -- Use mixins for common columns: timestamps, soft-delete, UUID PKs
- **model-table-args** (MEDIUM) -- Define indexes, constraints, and naming conventions in `__table_args__`

### 3. Query Patterns

- **query-select-statement** (CRITICAL) -- Use `select()` statement style (2.0 API), never legacy `session.query()`
- **query-eager-loading** (HIGH) -- Configure `joinedload`/`selectinload` explicitly per query
- **query-repository-class** (HIGH) -- Encapsulate queries in typed Repository classes with Protocol interface
- **query-typed-results** (HIGH) -- Type query results: `Sequence[Model]`, `ScalarResult[Model]`
- **query-no-implicit-autoflush** (MEDIUM) -- Disable autoflush; flush explicitly before reads that depend on writes

### 4. Migration Patterns

- **migration-alembic-autogenerate** (MEDIUM) -- Use Alembic with autogenerate; review every generated migration
- **migration-reversible** (MEDIUM) -- Always implement `downgrade()`; never use `pass`
- **migration-data-separate** (MEDIUM) -- Separate schema migrations from data migrations

### 5. Connection & Engine

- **engine-pool-config** (MEDIUM) -- Configure `pool_size`, `max_overflow`, `pool_recycle` for production
- **engine-naming-convention** (LOW) -- Set metadata `naming_convention` for consistent constraint names

## How to Use

Apply these rules when writing or reviewing SQLAlchemy code. Start with the CRITICAL
rules in Session Management and Model Design, which prevent the most common bugs
(connection leaks, thread-safety issues, missing type safety). Then adopt the HIGH
rules for Query Patterns to eliminate N+1 queries and improve testability. Finally,
apply MEDIUM and LOW rules for migration safety and production configuration.

Each rule file in the `rules/` directory contains a detailed explanation with
incorrect and correct code examples. The full compiled document is available in
[AGENTS.md](./AGENTS.md).

## Full Compiled Document

For the complete reference with all 19 rules expanded inline, organized by category,
see [AGENTS.md](./AGENTS.md).
