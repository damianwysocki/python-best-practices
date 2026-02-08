---
title: Unit Test Services with Mocked Dependencies
impact: MEDIUM
impactDescription: Fast isolated service tests
tags: testing, services, unit-tests, mocking, pytest
---

## Unit Test Services with Mocked Repositories/Dependencies

Service functions contain business logic and should be tested in isolation
from the database and external APIs. Mock the data layer and side effects
to keep tests fast (milliseconds, not seconds) and focused on behavior.

**Incorrect (service test hitting the database):**

```python
# tests/test_billing_service.py
@pytest.mark.django_db
class TestBillingService:
    def test_charge_user(self):
        user = User.objects.create(email="a@b.com", password="x")
        account = Account.objects.create(user=user, balance=Decimal("100"))
        # Hits DB, slow, tests persistence + logic together
        result = charge_user(user_id=user.id, amount=Decimal("30"))
        account.refresh_from_db()
        assert account.balance == Decimal("70")
        assert result.success is True
```

**Correct (mocked dependencies, pure logic test):**

```python
# tests/test_billing_service.py
from unittest.mock import patch, MagicMock

class TestBillingService:
    @patch("services.billing_service.Account.objects")
    @patch("services.billing_service.send_receipt")
    def test_charge_user_deducts_balance(self, mock_send, mock_qs):
        account = MagicMock(balance=Decimal("100"), is_frozen=False)
        mock_qs.select_for_update.return_value.get.return_value = account

        result = charge_user(user_id=1, amount=Decimal("30"))

        assert account.balance == Decimal("70")
        account.save.assert_called_once_with(update_fields=["balance"])
        mock_send.delay.assert_called_once_with(1, Decimal("30"))
        assert result.success is True

    @patch("services.billing_service.Account.objects")
    def test_charge_user_rejects_frozen_account(self, mock_qs):
        account = MagicMock(is_frozen=True)
        mock_qs.select_for_update.return_value.get.return_value = account

        with pytest.raises(AccountFrozenError):
            charge_user(user_id=1, amount=Decimal("30"))
```

Keep one integration test that hits the real database for each service to
verify ORM interactions, but the bulk of test cases should be fast unit tests.

Reference: https://docs.pytest.org/en/stable/how-to/monkeypatch.html
