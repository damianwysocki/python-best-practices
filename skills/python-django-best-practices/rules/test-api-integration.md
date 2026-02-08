---
title: Integration Test API Endpoints with APIClient
impact: MEDIUM
impactDescription: Validates full request cycle
tags: testing, api, integration, drf, APIClient
---

## Integration Test API Endpoints with APIClient

Unit testing services verifies business logic, but integration tests validate
the full HTTP cycle: routing, authentication, serialization, permissions, and
status codes. DRF's `APIClient` provides a clean interface for these tests
without spinning up a real server.

**Incorrect (testing views by calling methods directly):**

```python
# tests/test_views.py
class TestOrderView:
    def test_create_order(self):
        view = OrderCreateView()
        request = HttpRequest()
        request.method = "POST"
        request.POST = {"product_id": 1, "quantity": 2}
        # Bypasses middleware, auth, content negotiation
        response = view.post(request)
        assert response.status_code == 201
```

**Correct (full-stack test with APIClient):**

```python
# tests/test_api.py
import pytest
from rest_framework.test import APIClient
from tests.factories import UserFactory, ProductFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client):
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client

@pytest.mark.django_db
class TestOrderAPI:
    def test_create_order_returns_201(self, authenticated_client):
        product = ProductFactory(stock=10)
        response = authenticated_client.post(
            "/api/orders/",
            data={"product_id": product.id, "quantity": 2},
            format="json",
        )
        assert response.status_code == 201
        assert response.data["quantity"] == 2

    def test_create_order_unauthenticated_returns_401(self, api_client):
        response = api_client.post(
            "/api/orders/",
            data={"product_id": 1, "quantity": 1},
            format="json",
        )
        assert response.status_code == 401

    def test_list_orders_returns_only_own(self, authenticated_client):
        response = authenticated_client.get("/api/orders/")
        assert response.status_code == 200
        assert isinstance(response.data["results"], list)
```

Use `force_authenticate()` instead of managing tokens in tests. Use
`factory_boy` fixtures for test data and `pytest.mark.django_db` to
enable database access only in integration tests.

Reference: https://www.django-rest-framework.org/api-guide/testing/#apiclient
