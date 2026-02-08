---
name: python-fastapi-best-practices
description: |
  A comprehensive collection of 28 battle-tested rules for building production-grade
  FastAPI applications. Covers OpenAPI documentation, dependency injection, Pydantic
  schemas, router architecture, async patterns, security, and performance. Each rule
  includes incorrect and correct code examples with real-world Python/FastAPI usage.
license: MIT
metadata:
  author: python-best-practices
  version: '1.0.0'
---

# Python FastAPI Best Practices

A curated skill package of 28 rules for building robust, maintainable, and well-documented FastAPI applications. These rules encode hard-won lessons from production deployments and focus on patterns that prevent the most common and costly mistakes.

## When to Apply

- Building new FastAPI endpoints or services
- Reviewing pull requests that modify API routes, schemas, or middleware
- Setting up dependency injection for services and database sessions
- Designing Pydantic models for request/response validation
- Implementing authentication, authorization, or security middleware
- Optimizing async I/O patterns and database connection management
- Structuring a FastAPI project for long-term maintainability

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| CRITICAL | OpenAPI Documentation | Unusable API docs, missing contracts | `api-` |
| CRITICAL | Dependency Injection | Untestable code, leaked connections | `di-` |
| HIGH | Pydantic Models | Data leaks, silent coercion, duplication | `schema-` |
| HIGH | Router Architecture | Tangled logic, inconsistent errors | `router-` |
| HIGH | Async Patterns | Blocked event loops, wasted concurrency | `async-` |
| MEDIUM | Security & Validation | Auth gaps, CORS vulnerabilities | `security-` |
| MEDIUM | Performance | Connection exhaustion, memory bloat | `perf-` |

## Quick Reference

### 1. OpenAPI Documentation (6 rules)

- `api-endpoint-descriptions` — Every endpoint must have summary, description, and response_description
- `api-tags-organization` — Use tags to organize endpoints by domain; define tag metadata
- `api-response-models` — Always specify response_model with explicit status codes
- `api-error-responses` — Document all error responses with responses={} parameter
- `api-schema-examples` — Provide request/response examples via model_config json_schema_extra
- `api-versioned-prefix` — Version APIs with /api/v1/ prefix using APIRouter

### 2. Dependency Injection (4 rules)

- `di-depends-services` — Inject services via Depends(); never instantiate in endpoint body
- `di-scoped-db-session` — Use Depends() generator for database session lifecycle
- `di-abstract-depends` — Depends() on Protocol/ABC, swap implementations for testing
- `di-composable-deps` — Build complex dependencies from simple composable ones

### 3. Pydantic Models (5 rules)

- `schema-strict-mode` — Enable strict mode; use field validators for business rules
- `schema-separate-crud` — Separate CreateSchema, UpdateSchema, ResponseSchema per resource
- `schema-no-orm-in-response` — Never return ORM models directly; always map to response schema
- `schema-computed-fields` — Use @computed_field for derived data in response models
- `schema-reusable-fields` — Extract common field patterns into base schemas

### 4. Router Architecture (4 rules)

- `router-slim-endpoints` — Endpoints call services only; zero business logic in routes
- `router-resource-naming` — RESTful plural nouns for resources: /users, /orders
- `router-exception-handlers` — Register custom exception handlers; never try/except in endpoints
- `router-middleware-ordering` — Correct middleware stack: CORS, Auth, Logging, Error

### 5. Async Patterns (4 rules)

- `async-all-io` — Use async def + await for all I/O-bound operations
- `async-gather-parallel` — Use asyncio.gather() for independent async operations
- `async-no-sync-in-async` — Never call blocking I/O in async endpoints; use run_in_executor
- `async-background-tasks` — Use BackgroundTasks for fire-and-forget operations

### 6. Security & Validation (3 rules)

- `security-auth-dependency` — Implement auth as a reusable Depends() callable
- `security-rate-limiting` — Apply rate limiting via middleware or dependency
- `security-cors-explicit` — Configure CORS with explicit origins, never wildcard in production

### 7. Performance (2 rules)

- `perf-connection-pooling` — Configure async connection pool size matching worker count
- `perf-streaming-response` — Use StreamingResponse for large payloads or file downloads

## How to Use

Each rule file in the `rules/` directory is self-contained with:

1. **YAML frontmatter** — title, impact level, tags for filtering
2. **Explanation** — why the rule matters and what goes wrong without it
3. **Incorrect example** — real Python/FastAPI code showing the anti-pattern
4. **Correct example** — production-ready code showing the recommended approach
5. **References** — links to official documentation

Apply rules during development and code review. Use the impact level (CRITICAL, HIGH, MEDIUM, LOW) to prioritize which rules to enforce first in your project.

## Full Compiled Document

See [AGENTS.md](./AGENTS.md) for all 28 rules expanded inline in a single document, organized by category.
