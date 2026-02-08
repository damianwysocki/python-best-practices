---
title: Use Meta.constraints for Database-Level Data Integrity
impact: HIGH
impactDescription: Guarantees data integrity
tags: models, constraints, database, integrity, validation
---

## Use Meta.constraints for Database-Level Data Integrity

Application-level validation can be bypassed by raw SQL, management commands,
or bulk operations. Database constraints are the last line of defense. Use
`Meta.constraints` with `CheckConstraint` and `UniqueConstraint` to enforce
invariants at the database level.

**Incorrect (relying only on application validation):**

```python
# models.py
class Booking(models.Model):
    room = models.ForeignKey(Room, on_delete=models.CASCADE)
    check_in = models.DateField()
    check_out = models.DateField()
    guests = models.PositiveIntegerField()

    def clean(self):
        if self.check_out <= self.check_in:
            raise ValidationError("check_out must be after check_in")
        # This is never enforced at the DB level!
```

**Correct (database-level constraints):**

```python
# models.py
from django.db.models import Q, F, CheckConstraint, UniqueConstraint

class Booking(models.Model):
    room = models.ForeignKey(Room, on_delete=models.CASCADE, related_name="bookings")
    check_in = models.DateField()
    check_out = models.DateField()
    guests = models.PositiveIntegerField()

    class Meta:
        constraints = [
            CheckConstraint(
                check=Q(check_out__gt=F("check_in")),
                name="booking_checkout_after_checkin",
            ),
            CheckConstraint(
                check=Q(guests__gte=1, guests__lte=20),
                name="booking_guests_range",
            ),
            UniqueConstraint(
                fields=["room", "check_in"],
                name="booking_unique_room_date",
            ),
        ]
```

Name constraints with a `<table>_<description>` pattern so migration errors
are easy to trace. Combine with `clean()` for user-friendly error messages.

Reference: https://docs.djangoproject.com/en/5.0/ref/models/constraints/
