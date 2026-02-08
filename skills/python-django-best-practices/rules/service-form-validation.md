---
title: Use Forms/Serializers for Validation, Services for Logic
impact: HIGH
impactDescription: Clear validation boundaries
tags: service-layer, validation, serializers, forms
---

## Use Forms/Serializers for Validation, Services for Logic

Serializers and forms own input validation (field types, required fields,
uniqueness, format). Services own business rules (authorization, state
transitions, cross-entity consistency). Mixing the two creates serializers
that depend on database state or services that re-validate field types.

**Incorrect (business validation in serializer):**

```python
# serializers.py
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.DecimalField(max_digits=10, decimal_places=2)

    def validate(self, attrs):
        from_acc = Account.objects.get(id=attrs["from_account"])
        if from_acc.balance < attrs["amount"]:
            raise serializers.ValidationError("Insufficient funds")
        if from_acc.is_frozen:
            raise serializers.ValidationError("Account is frozen")
        return attrs
```

**Correct (serializer validates shape, service validates business rules):**

```python
# serializers.py
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.DecimalField(
        max_digits=10, decimal_places=2, min_value=Decimal("0.01"),
    )

# services/transfer_service.py
class InsufficientFundsError(Exception):
    pass

def execute_transfer(*, from_account_id: int, to_account_id: int, amount: Decimal) -> Transfer:
    from_acc = Account.objects.select_for_update().get(id=from_account_id)
    if from_acc.is_frozen:
        raise AccountFrozenError(from_acc.id)
    if from_acc.balance < amount:
        raise InsufficientFundsError(from_acc.id, amount)
    to_acc = Account.objects.select_for_update().get(id=to_account_id)
    return _perform_transfer(from_acc, to_acc, amount)
```

This separation lets you reuse the same service from CLI commands, tasks, or
other services without going through serializer machinery.

Reference: https://www.django-rest-framework.org/api-guide/serializers/#validation
