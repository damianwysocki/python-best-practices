# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Service Layer Integration (service)

**Impact:** CRITICAL
**Description:** Business logic belongs in a dedicated service layer, not in views, serializers, or models. This separation enables testability and prevents fat models/views.

## 2. Model Design (model)

**Impact:** CRITICAL
**Description:** Well-designed models with abstract bases, explicit related names, custom managers, and database constraints ensure data integrity and query reusability.

## 3. DRF Patterns (drf)

**Impact:** HIGH
**Description:** Django REST Framework patterns for serializers, viewsets, documentation, pagination, permissions, and filtering that keep the API layer clean and consistent.

## 4. Query Optimization (query)

**Impact:** HIGH
**Description:** Preventing N+1 queries, using select_related/prefetch_related, and leveraging bulk operations are essential for Django application performance.

## 5. Testing (test)

**Impact:** MEDIUM
**Description:** Factory-based test data, service unit tests, and API integration tests provide confidence in application correctness.
