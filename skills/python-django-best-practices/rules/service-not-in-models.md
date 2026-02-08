---
title: Models Handle Persistence Schema Only, Never Business Logic
impact: CRITICAL
impactDescription: Keeps models focused and testable
tags: service-layer, models, architecture, separation-of-concerns
---

## Models Handle Persistence Schema Only, Never Business Logic

Django models should define fields, relationships, indexes, constraints, and
lightweight computed properties (`@property`). Business workflows, side effects,
and cross-model orchestration belong in the service layer. Fat models become
untestable monoliths that mix persistence with domain rules.

**Incorrect (business logic inside a model):**

```python
# models.py
class Subscription(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    status = models.CharField(max_length=20, default="active")
    expires_at = models.DateTimeField()

    def renew(self):
        if self.status != "active":
            raise ValueError("Cannot renew inactive subscription")
        charge = StripeGateway().charge(self.user, self.plan.price)
        if charge.success:
            self.expires_at += timedelta(days=30)
            self.save()
            send_renewal_email.delay(self.user.id)
        return charge
```

**Correct (model defines schema, service handles renewal):**

```python
# models.py
class Subscription(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    status = models.CharField(max_length=20, default="active")
    expires_at = models.DateTimeField()

    @property
    def is_active(self) -> bool:
        return self.status == "active" and self.expires_at > timezone.now()

# services/subscription_service.py
def renew_subscription(*, subscription: Subscription, gateway: PaymentGateway) -> Charge:
    if not subscription.is_active:
        raise SubscriptionInactiveError(subscription.id)
    charge = gateway.charge(subscription.user, subscription.plan.price)
    if charge.success:
        subscription.expires_at += timedelta(days=30)
        subscription.save(update_fields=["expires_at"])
        send_renewal_email.delay(subscription.user_id)
    return charge
```

Model properties that are pure data derivations (e.g., `is_active`) are
acceptable. Anything that triggers side effects must live in a service.

Reference: https://docs.djangoproject.com/en/5.0/topics/db/models/
