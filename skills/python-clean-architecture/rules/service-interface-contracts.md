---
title: Define Protocol/ABC for Every Service Interface
impact: CRITICAL
impactDescription: Enables testability and swapping
tags: service, protocol, abc, interface, contract
---

## Define Protocol/ABC for Every Service Interface

Every service that will be consumed by another bounded context or injected as a dependency must have a corresponding Protocol or ABC defining its contract. This enables testing with mocks and swapping implementations without changing consumers.

**Incorrect (no interface defined, depending on concrete class):**

```python
# notifications/email_sender.py
import smtplib

class SmtpEmailSender:
    def send(self, to: str, subject: str, body: str) -> None:
        server = smtplib.SMTP("smtp.example.com")
        server.send_message(to, subject, body)

# orders/service.py
from notifications.email_sender import SmtpEmailSender

class OrderService:
    def __init__(self, sender: SmtpEmailSender) -> None:
        self._sender = sender  # tied to SMTP implementation
```

**Correct (Protocol defines the contract):**

```python
# notifications/interface.py
from typing import Protocol

class EmailSender(Protocol):
    def send(self, to: str, subject: str, body: str) -> None: ...

# notifications/smtp_sender.py
from notifications.interface import EmailSender

class SmtpEmailSender:
    def send(self, to: str, subject: str, body: str) -> None:
        server = smtplib.SMTP("smtp.example.com")
        server.send_message(to, subject, body)

# orders/service.py
from notifications.interface import EmailSender

class OrderService:
    def __init__(self, sender: EmailSender) -> None:
        self._sender = sender  # depends on protocol, not implementation
```

With a Protocol in place, tests can supply a fake sender and production can swap SMTP for an API-based provider with zero changes to the order service.
