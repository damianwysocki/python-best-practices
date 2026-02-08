---
name: python-django-best-practices
description: |
  Comprehensive best practices for Django and Django REST Framework projects.
  Covers service layer architecture, model design, DRF patterns, query
  optimization, and testing strategies. Enforces clean separation of concerns,
  database-level integrity, and performance-aware query patterns across all
  Django application layers.
license: MIT
metadata:
  author: python-best-practices
  version: '1.0.0'
---

# Python Django Best Practices

A curated set of 23 rules for building maintainable, performant, and testable Django applications with Django REST Framework. These rules encode hard-won lessons from production Django projects and establish clear boundaries between HTTP, business logic, and persistence layers.

## When to Apply

- Building Django web applications or REST APIs with Django REST Framework
- Designing model schemas for relational databases (PostgreSQL, MySQL)
- Structuring service layers to separate business logic from views
- Optimizing ORM queries for performance at scale
- Writing tests for Django services and API endpoints
- Reviewing pull requests in Django/DRF codebases
- Setting up new Django projects with clean architecture patterns

## Rule Categories by Priority

| Priority | Category              | Impact   | Prefix     |
|----------|-----------------------|----------|------------|
| 1        | Service Layer Integration | CRITICAL | `service-` |
| 2        | Model Design          | CRITICAL | `model-`   |
| 3        | DRF Patterns          | HIGH     | `drf-`     |
| 4        | Query Optimization    | HIGH     | `query-`   |
| 5        | Testing               | MEDIUM   | `test-`    |

## Quick Reference

### 1. Service Layer Integration (CRITICAL)

- `service-not-in-views` -- Business logic in service layer, not in views or serializers
- `service-not-in-models` -- Models handle persistence schema only, never business logic
- `service-thin-views` -- Views/ViewSets handle HTTP concerns only, delegate to services
- `service-form-validation` -- Use forms/serializers for validation, services for logic

### 2. Model Design (CRITICAL)

- `model-abstract-base` -- Use abstract base models for shared fields (created_at, updated_at, uuid)
- `model-explicit-related-name` -- Always set related_name on ForeignKey and ManyToMany
- `model-custom-managers` -- Use custom Managers for reusable query encapsulation
- `model-db-constraints` -- Use Meta.constraints for database-level data integrity
- `model-typed-fields` -- Use django-stubs with explicit type annotations on all fields

### 3. DRF Patterns (HIGH)

- `drf-serializer-per-action` -- Separate serializers for create/update/list/detail actions
- `drf-viewset-minimal` -- Keep ViewSets thin; override only what is needed
- `drf-spectacular-docs` -- Use drf-spectacular for OpenAPI 3.0 with descriptions and examples
- `drf-pagination-standard` -- Use CursorPagination or LimitOffsetPagination with consistent format
- `drf-permission-classes` -- Fine-grained permission classes per view, not global
- `drf-filter-backends` -- Use django-filter with typed FilterSet classes

### 4. Query Optimization (HIGH)

- `query-select-prefetch` -- Always use select_related/prefetch_related for accessed relations
- `query-no-n-plus-one` -- Detect and prevent N+1 queries with django-query-count or nplusone
- `query-only-defer` -- Use .only()/.defer() to load only needed columns
- `query-bulk-operations` -- Use bulk_create/bulk_update for batch operations, never loops
- `query-exists-over-count` -- Use .exists() instead of .count() > 0 for existence checks

### 5. Testing (MEDIUM)

- `test-factory-boy` -- Use factory_boy for test data; never raw Model.objects.create() in tests
- `test-service-unit` -- Unit test services with mocked repositories/dependencies
- `test-api-integration` -- Integration test API endpoints with APIClient

## How to Use

**In code reviews:** Reference rules by prefix (e.g., "This violates `service-not-in-views` -- business logic should be in the service layer, not the view").

**In new projects:** Start with the Service Layer Integration and Model Design categories to establish clean architecture from day one.

**In existing projects:** Prioritize Query Optimization rules for immediate performance wins, then incrementally adopt service layer patterns.

**In CI pipelines:** Use `nplusone` and `django-query-count` for automated N+1 detection. Use `mypy` with `django-stubs` for type safety.

## Full Compiled Document

For the complete reference with all rules expanded inline, including code examples, see [AGENTS.md](AGENTS.md).
