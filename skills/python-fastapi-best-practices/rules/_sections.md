# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. OpenAPI Documentation (api)

**Impact:** CRITICAL
**Description:** Comprehensive API documentation ensures discoverability and correct client integration. Every endpoint must be fully documented with descriptions, response models, and error responses.

## 2. Dependency Injection (di)

**Impact:** CRITICAL
**Description:** FastAPI's Depends() system enables clean, testable architecture. Services, database sessions, and auth must be injected, never instantiated inline.

## 3. Pydantic Models (schema)

**Impact:** HIGH
**Description:** Strict Pydantic schemas enforce data contracts between API consumers and your application. Separate schemas per operation prevent data leakage.

## 4. Router Architecture (router)

**Impact:** HIGH
**Description:** Clean router organization with slim endpoints, RESTful naming, and centralized exception handling keeps the API layer maintainable.

## 5. Async Patterns (async)

**Impact:** HIGH
**Description:** Correct async/await usage prevents event loop blocking and enables concurrent I/O for maximum throughput.

## 6. Security & Validation (security)

**Impact:** MEDIUM
**Description:** Authentication, rate limiting, and CORS configuration protect APIs from abuse and unauthorized access.

## 7. Performance (perf)

**Impact:** MEDIUM
**Description:** Connection pooling and streaming responses optimize resource usage for production workloads.
