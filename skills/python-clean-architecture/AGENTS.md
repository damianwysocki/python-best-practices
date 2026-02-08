# Python Clean Architecture

**Version 1.0.0** | Python Clean Architecture & Strict Typing | February 2026

> Note: This document contains 30 rules organized across 6 categories for building maintainable, type-safe Python applications using clean architecture principles. All rules include real-world Python code examples demonstrating incorrect and correct patterns.

## Abstract

This guide codifies 30 architectural rules for Python projects that prioritize long-term maintainability, strict type safety, and domain-driven design. The rules enforce bounded context isolation, explicit dependency injection, complete type annotations without escape hatches, immutable data transfer objects, disciplined project structure, and clean code patterns. Following these rules produces codebases that are navigable by business domain, testable in isolation, and resilient to change.

## Table of Contents

1. [Service Layer & Bounded Contexts](#service-layer--bounded-contexts)
   - 1.1 [Organize Code by Bounded Context](#11-organize-code-by-bounded-context)
   - 1.2 [Isolate Service Communication Through Interfaces](#12-isolate-service-communication-through-interfaces)
   - 1.3 [No Cross-Context Repository Access](#13-no-cross-context-repository-access)
   - 1.4 [Define Protocol/ABC for Every Service Interface](#14-define-protocolabc-for-every-service-interface)
   - 1.5 [Single Responsibility per Service](#15-single-responsibility-per-service)
2. [Dependency Injection](#dependency-injection)
   - 2.1 [Inject Dependencies via Constructor](#21-inject-dependencies-via-constructor)
   - 2.2 [Depend on Abstractions, Not Concretions](#22-depend-on-abstractions-not-concretions)
   - 2.3 [Wire Dependencies at the Composition Root](#23-wire-dependencies-at-the-composition-root)
   - 2.4 [Never Use Service Locator Pattern](#24-never-use-service-locator-pattern)
   - 2.5 [Match Dependency Lifecycle to Scope](#25-match-dependency-lifecycle-to-scope)
3. [Strict Typing](#strict-typing)
   - 3.1 [Never Use Any Type](#31-never-use-any-type)
   - 3.2 [Never Use cast()](#32-never-use-cast)
   - 3.3 [Always Annotate Return Types](#33-always-annotate-return-types)
   - 3.4 [Always Annotate All Function Parameters](#34-always-annotate-all-function-parameters)
   - 3.5 [Prefer Protocol for Structural Subtyping](#35-prefer-protocol-for-structural-subtyping)
   - 3.6 [Use Generic[T] for Reusable Typed Containers](#36-use-generict-for-reusable-typed-containers)
   - 3.7 [Use NewType for Domain Identifiers](#37-use-newtype-for-domain-identifiers)
   - 3.8 [Use Final for Constants](#38-use-final-for-constants)
4. [Data Transfer Objects](#data-transfer-objects)
   - 4.1 [Use Frozen Dataclasses for DTOs](#41-use-frozen-dataclasses-for-dtos)
   - 4.2 [Never Pass Raw Dicts Between Layers](#42-never-pass-raw-dicts-between-layers)
   - 4.3 [Separate DTOs per Layer](#43-separate-dtos-per-layer)
   - 4.4 [Use Classmethod Factories for DTO Construction](#44-use-classmethod-factories-for-dto-construction)
   - 4.5 [Validate Data at System Boundaries](#45-validate-data-at-system-boundaries)
5. [Project Structure & Isolation](#project-structure--isolation)
   - 5.1 [Top-Level Packages as Bounded Contexts](#51-top-level-packages-as-bounded-contexts)
   - 5.2 [Use __all__ to Control Public API](#52-use-__all__-to-control-public-api)
   - 5.3 [Prevent Circular Imports](#53-prevent-circular-imports)
   - 5.4 [Centralize Config in Typed Settings](#54-centralize-config-in-typed-settings)
6. [Clean Code](#clean-code)
   - 6.1 [Prefer Composition Over Inheritance](#61-prefer-composition-over-inheritance)
   - 6.2 [Use Guard Clauses and Early Returns](#62-use-guard-clauses-and-early-returns)
   - 6.3 [Define Domain-Specific Exception Hierarchy](#63-define-domain-specific-exception-hierarchy)

---

## Service Layer & Bounded Contexts

### 1.1 Organize Code by Bounded Context

**Impact: CRITICAL** -- Domain alignment clarity

Structure your project around domain bounded contexts (e.g., `users/`, `orders/`, `payments/`) rather than technical layers (e.g., `models/`, `services/`, `repositories/`). Each bounded context encapsulates its own models, services, repositories, and APIs, making the codebase navigable by business capability.

**Incorrect (organized by technical layer):**

```python
# project/models/user.py
class User:
    ...

# project/models/order.py
class Order:
    ...

# project/services/user_service.py
from project.models.user import User

class UserService:
    ...

# project/services/order_service.py
from project.models.order import Order
from project.models.user import User  # cross-cutting concern buried in layers

class OrderService:
    ...
```

**Correct (organized by bounded context):**

```python
# project/users/domain.py
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class User:
    user_id: str
    email: str
    name: str

# project/users/service.py
from project.users.domain import User
from project.users.repository import UserRepository

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

# project/orders/domain.py
@dataclass(frozen=True, slots=True)
class Order:
    order_id: str
    buyer_id: str
    total: Decimal

# project/orders/service.py
from project.orders.domain import Order
from project.orders.repository import OrderRepository

class OrderService:
    def __init__(self, repo: OrderRepository) -> None:
        self._repo = repo
```

When every bounded context owns its full vertical slice, changes to one domain rarely ripple into another.

---

### 1.2 Isolate Service Communication Through Interfaces

**Impact: CRITICAL** -- Prevents coupling leaks

Services must communicate only through well-defined service interfaces (Protocol or ABC). A service should never directly import or call another bounded context's repository, database client, or internal implementation detail.

**Incorrect (service directly uses another context's repository):**

```python
# orders/service.py
from users.repository import UserRepository  # direct repo import

class OrderService:
    def __init__(self, order_repo: OrderRepository, user_repo: UserRepository) -> None:
        self._order_repo = order_repo
        self._user_repo = user_repo  # leaking internal detail

    def place_order(self, user_id: str, items: list[str]) -> Order:
        user = self._user_repo.get_by_id(user_id)  # reaching into users internals
        return self._order_repo.create(user.email, items)
```

**Correct (service communicates through a service interface):**

```python
# users/interface.py
from typing import Protocol

class UserServiceInterface(Protocol):
    def get_user_email(self, user_id: str) -> str: ...

# orders/service.py
from users.interface import UserServiceInterface

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        user_service: UserServiceInterface,
    ) -> None:
        self._order_repo = order_repo
        self._user_service = user_service

    def place_order(self, user_id: str, items: list[str]) -> Order:
        email = self._user_service.get_user_email(user_id)
        return self._order_repo.create(email, items)
```

This ensures that internal implementation changes in one bounded context do not force changes in another.

---

### 1.3 No Cross-Context Repository Access

**Impact: CRITICAL** -- Enforces domain boundaries

A service must never import or access another bounded context's repository. Repositories are internal implementation details of their owning context. Cross-context data needs must flow through service interfaces.

**Incorrect (order service imports user repository):**

```python
# orders/service.py
from orders.repository import OrderRepository
from users.repository import UserRepository  # violation: cross-context repo access

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        user_repo: UserRepository,
    ) -> None:
        self._order_repo = order_repo
        self._user_repo = user_repo

    def create_order(self, user_id: str, amount: Decimal) -> Order:
        user = self._user_repo.find_by_id(user_id)
        if user.credit < amount:
            raise InsufficientCredit()
        return self._order_repo.save(Order(user_id=user_id, amount=amount))
```

**Correct (order service uses user service interface):**

```python
# users/interface.py
from typing import Protocol
from decimal import Decimal

class UserCreditChecker(Protocol):
    def has_sufficient_credit(self, user_id: str, amount: Decimal) -> bool: ...

# orders/service.py
from orders.repository import OrderRepository
from users.interface import UserCreditChecker

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        credit_checker: UserCreditChecker,
    ) -> None:
        self._order_repo = order_repo
        self._credit_checker = credit_checker

    def create_order(self, user_id: str, amount: Decimal) -> Order:
        if not self._credit_checker.has_sufficient_credit(user_id, amount):
            raise InsufficientCredit()
        return self._order_repo.save(Order(user_id=user_id, amount=amount))
```

This preserves the encapsulation of each bounded context's persistence layer.

---

### 1.4 Define Protocol/ABC for Every Service Interface

**Impact: CRITICAL** -- Enables testability and swapping

Every service that will be consumed by another bounded context or injected as a dependency must have a corresponding Protocol or ABC defining its contract. This enables testing with mocks and swapping implementations without changing consumers.

**Incorrect (no interface defined, depending on concrete class):**

```python
from infrastructure.postgres_repo import PostgresUserRepository

class UserService:
    def __init__(self, repo: PostgresUserRepository) -> None:
        self._repo = repo  # tied to Postgres forever

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)
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

---

### 1.5 Single Responsibility per Service

**Impact: HIGH** -- Reduces service bloat

Each service class should handle the business logic for exactly one bounded context or subdomain. Avoid "god services" that orchestrate multiple unrelated concerns. When a service grows beyond its context, extract a new bounded context.

**Incorrect (one service handling multiple domains):**

```python
class AppService:
    def __init__(
        self,
        user_repo: UserRepository,
        order_repo: OrderRepository,
        payment_gateway: PaymentGateway,
        email_client: EmailClient,
    ) -> None:
        self._user_repo = user_repo
        self._order_repo = order_repo
        self._payment_gateway = payment_gateway
        self._email_client = email_client

    def register_user(self, email: str, name: str) -> User: ...
    def place_order(self, user_id: str, items: list[str]) -> Order: ...
    def process_payment(self, order_id: str) -> Payment: ...
    def send_receipt(self, payment_id: str) -> None: ...
```

**Correct (each service owns one context):**

```python
# users/service.py
class UserService:
    def __init__(self, user_repo: UserRepository) -> None:
        self._user_repo = user_repo

    def register(self, email: str, name: str) -> User: ...

# orders/service.py
class OrderService:
    def __init__(self, order_repo: OrderRepository) -> None:
        self._order_repo = order_repo

    def place_order(self, user_id: str, items: list[str]) -> Order: ...

# payments/service.py
class PaymentService:
    def __init__(self, gateway: PaymentGateway) -> None:
        self._gateway = gateway

    def process(self, order_id: str) -> Payment: ...
```

When orchestration across contexts is needed, create a thin application service or use an event-driven approach rather than merging domains.

---

## Dependency Injection

### 2.1 Inject Dependencies via Constructor

**Impact: CRITICAL** -- Explicit dependency graph

Always inject dependencies through `__init__` parameters. Never instantiate collaborators inside methods or at class level. Constructor injection makes the dependency graph explicit, simplifies testing, and enables the composition root to control wiring.

**Incorrect (instantiating dependencies inside methods):**

```python
class OrderService:
    def place_order(self, user_id: str, items: list[str]) -> Order:
        repo = PostgresOrderRepository()  # hidden dependency
        notifier = SmtpNotifier()  # hidden dependency
        order = repo.create(user_id, items)
        notifier.notify(user_id, f"Order {order.id} placed")
        return order
```

**Correct (injecting via __init__):**

```python
class OrderService:
    def __init__(
        self,
        repo: OrderRepository,
        notifier: Notifier,
    ) -> None:
        self._repo = repo
        self._notifier = notifier

    def place_order(self, user_id: str, items: list[str]) -> Order:
        order = self._repo.create(user_id, items)
        self._notifier.notify(user_id, f"Order {order.id} placed")
        return order
```

Constructor injection ensures every dependency is visible in the signature, making it trivial to supply test doubles and reason about what a class needs.

---

### 2.2 Depend on Abstractions, Not Concretions

**Impact: CRITICAL** -- Decouples implementation details

All injected dependencies should be typed as Protocol or ABC, never as concrete classes. This follows the Dependency Inversion Principle and allows swapping implementations without modifying consumers.

**Incorrect (depending on concrete implementation):**

```python
from infrastructure.postgres_repo import PostgresUserRepository

class UserService:
    def __init__(self, repo: PostgresUserRepository) -> None:
        self._repo = repo  # tied to Postgres forever

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)
```

**Correct (depending on abstraction):**

```python
from typing import Protocol

class UserRepository(Protocol):
    def find_by_id(self, user_id: str) -> User: ...
    def save(self, user: User) -> None: ...

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo  # accepts any conforming implementation

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)
```

Now `PostgresUserRepository`, `InMemoryUserRepository`, or any future implementation can be plugged in without touching `UserService`.

---

### 2.3 Wire Dependencies at the Composition Root

**Impact: HIGH** -- Centralizes object creation

All dependency wiring should happen in a single composition root (typically `main.py`, `container.py`, or a factory function). Business logic modules should never create their own dependencies or reference the DI container.

**Incorrect (wiring scattered across modules):**

```python
# orders/service.py
from infrastructure.postgres_repo import PostgresOrderRepository
from infrastructure.smtp_notifier import SmtpNotifier

class OrderService:
    def __init__(self) -> None:
        self._repo = PostgresOrderRepository(dsn="postgres://...")
        self._notifier = SmtpNotifier(host="smtp.example.com")
```

**Correct (wiring at composition root):**

```python
# orders/service.py
class OrderService:
    def __init__(self, repo: OrderRepository, notifier: Notifier) -> None:
        self._repo = repo
        self._notifier = notifier

# container.py  (composition root)
from orders.service import OrderService
from infrastructure.postgres_repo import PostgresOrderRepository
from infrastructure.smtp_notifier import SmtpNotifier

def create_order_service(settings: Settings) -> OrderService:
    repo = PostgresOrderRepository(dsn=settings.database_url)
    notifier = SmtpNotifier(host=settings.smtp_host)
    return OrderService(repo=repo, notifier=notifier)

# main.py
def main() -> None:
    settings = Settings.from_env()
    order_service = create_order_service(settings)
    app = create_app(order_service=order_service)
    app.run()
```

Centralizing wiring makes it easy to see the full dependency graph and swap implementations for different environments.

---

### 2.4 Never Use Service Locator Pattern

**Impact: HIGH** -- Hides dependency graph

Avoid the service locator pattern where classes look up their dependencies from a global registry at runtime. This hides the dependency graph, makes testing harder, and couples code to the locator.

**Incorrect (service locator pattern):**

```python
class ServiceLocator:
    _services: dict[type, object] = {}

    @classmethod
    def register(cls, interface: type, impl: object) -> None:
        cls._services[interface] = impl

    @classmethod
    def get(cls, interface: type) -> object:
        return cls._services[interface]

class OrderService:
    def place_order(self, user_id: str) -> Order:
        repo = ServiceLocator.get(OrderRepository)  # hidden lookup
        notifier = ServiceLocator.get(Notifier)  # hidden lookup
        order = repo.create(user_id)
        notifier.notify(order)
        return order
```

**Correct (explicit constructor injection):**

```python
class OrderService:
    def __init__(
        self,
        repo: OrderRepository,
        notifier: Notifier,
    ) -> None:
        self._repo = repo
        self._notifier = notifier

    def place_order(self, user_id: str) -> Order:
        order = self._repo.create(user_id)
        self._notifier.notify(order)
        return order
```

Explicit injection keeps the dependency graph visible at the call site and in the constructor signature, making both testing and reasoning straightforward.

---

### 2.5 Match Dependency Lifecycle to Scope

**Impact: MEDIUM** -- Prevents resource leaks

Ensure each dependency's lifecycle matches its intended scope. Singletons (e.g., connection pools) live for the application lifetime. Request-scoped dependencies (e.g., database sessions) are created and destroyed per request. Transient dependencies are created fresh each time.

**Incorrect (singleton session causes request leakage):**

```python
db_session = SessionLocal()  # one session for entire app lifetime

def create_user_service() -> UserService:
    return UserService(session=db_session)  # all requests share one session
```

**Correct (request-scoped session):**

```python
from contextlib import contextmanager
from collections.abc import Generator

engine = create_engine(settings.database_url)  # singleton: app lifetime
SessionLocal = sessionmaker(bind=engine)  # singleton: factory

@contextmanager
def request_session() -> Generator[Session, None, None]:
    session = SessionLocal()  # transient: per-request
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

def create_user_service(session: Session) -> UserService:
    repo = SqlAlchemyUserRepository(session=session)  # request-scoped
    return UserService(repo=repo)
```

Mismatched lifecycles lead to subtle bugs: shared sessions leak state between requests, and transient connection pools waste resources.

---

## Strict Typing

### 3.1 Never Use Any Type

**Impact: CRITICAL** -- Destroys type safety

Never use `Any` in type annotations. It silently disables type checking for everything it touches. Instead, use proper generics with `TypeVar`, `Protocol` for structural typing, or `Union`/`object` when the type is genuinely unknown.

**Incorrect (using Any):**

```python
from typing import Any

def process_event(event: Any) -> Any:
    return event.handle()  # no type safety, no autocompletion

def store_items(items: list[Any]) -> None:
    for item in items:
        item.save()  # could fail at runtime with no warning

cache: dict[str, Any] = {}
```

**Correct (using proper types):**

```python
from typing import Protocol, TypeVar

class Handleable(Protocol):
    def handle(self) -> EventResult: ...

def process_event(event: Handleable) -> EventResult:
    return event.handle()  # fully type-checked

class Saveable(Protocol):
    def save(self) -> None: ...

def store_items(items: list[Saveable]) -> None:
    for item in items:
        item.save()  # type checker verifies .save() exists

T = TypeVar("T")
cache: dict[str, T]  # generic, preserves type info
```

If you find yourself reaching for `Any`, it usually means you need a Protocol, a TypeVar, or a redesign of the interface.

---

### 3.2 Never Use cast()

**Impact: CRITICAL** -- Bypasses type verification

Never use `typing.cast()`. It tells the type checker to trust you without any runtime verification, which masks bugs. Instead, use `isinstance` type narrowing, `@overload`, or redesign the interface so the type is known statically.

**Incorrect (using cast to force types):**

```python
from typing import cast

def get_user_name(data: dict[str, object]) -> str:
    name = data["name"]
    return cast(str, name)  # no runtime check, could be int or None

def process_response(response: Response) -> UserDTO:
    payload = response.json()
    return cast(UserDTO, payload)  # dangerous: payload is a dict, not UserDTO
```

**Correct (using isinstance narrowing):**

```python
def get_user_name(data: dict[str, object]) -> str:
    name = data["name"]
    if not isinstance(name, str):
        raise TypeError(f"Expected str for 'name', got {type(name).__name__}")
    return name  # type checker knows this is str

def process_response(response: Response) -> UserDTO:
    payload = response.json()
    return UserDTO.from_dict(payload)  # explicit construction with validation
```

Runtime type narrowing with `isinstance` gives you both type safety and runtime correctness guarantees that `cast()` cannot provide.

---

### 3.3 Always Annotate Return Types

**Impact: CRITICAL** -- Catches silent bugs

Every function and method must have an explicit return type annotation. Without it, the type checker infers the return type, which can silently change when the implementation changes, propagating incorrect types throughout the codebase.

**Incorrect (missing return annotations):**

```python
class UserService:
    def __init__(self, repo: UserRepository):
        self._repo = repo

    def get_user(self, user_id: str):  # what does this return?
        return self._repo.find_by_id(user_id)

    def get_user_name(self, user_id: str):
        user = self.get_user(user_id)
        if user is None:
            return None  # caller might not handle None
        return user.name
```

**Correct (explicit return annotations):**

```python
class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)

    def get_user_name(self, user_id: str) -> str | None:
        user = self.get_user(user_id)
        if user is None:
            return None
        return user.name
```

Explicit return types serve as documentation, catch refactoring errors early, and prevent type inference from silently widening or narrowing.

---

### 3.4 Always Annotate All Function Parameters

**Impact: CRITICAL** -- Enables static analysis

Every function parameter must have a type annotation. Unannotated parameters default to `Any` in most type checkers, which silently disables type checking for all code that touches that parameter.

**Incorrect (unannotated parameters):**

```python
def create_order(user_id, items, discount=0.0):
    total = sum(item.price for item in items)
    return Order(user_id=user_id, total=total * (1 - discount))

def send_notification(user, message, urgent=False):
    if urgent:
        user.send_sms(message)
    else:
        user.send_email(message)
```

**Correct (fully annotated parameters):**

```python
from decimal import Decimal

def create_order(
    user_id: str,
    items: list[OrderItem],
    discount: Decimal = Decimal("0.0"),
) -> Order:
    total = sum(item.price for item in items)
    return Order(user_id=user_id, total=total * (1 - discount))

def send_notification(
    user: User,
    message: str,
    urgent: bool = False,
) -> None:
    if urgent:
        user.send_sms(message)
    else:
        user.send_email(message)
```

Full parameter annotations let the type checker verify every call site, catching type mismatches before runtime.

---

### 3.5 Prefer Protocol for Structural Subtyping

**Impact: HIGH** -- Reduces coupling overhead

Use `Protocol` for structural subtyping (duck typing with type safety) instead of ABC when implementations do not need to explicitly inherit. Protocol lets any class that has the right methods satisfy the type, without requiring an inheritance relationship.

**Incorrect (forcing ABC inheritance):**

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def find_by_id(self, entity_id: str) -> dict: ...

    @abstractmethod
    def save(self, entity: dict) -> None: ...

# Every implementation must explicitly inherit
class PostgresRepository(Repository):
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# Third-party adapter cannot inherit without wrapping
class ThirdPartyAdapter(Repository):  # forced coupling
    ...
```

**Correct (using Protocol):**

```python
from typing import Protocol

class Repository(Protocol):
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# No inheritance needed -- structural conformance
class PostgresRepository:
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# Third-party adapters automatically conform if methods match
class ThirdPartyAdapter:
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...
```

Reserve ABC for cases where you need shared implementation (template method pattern) or runtime `isinstance` checks with `runtime_checkable`.

---

### 3.6 Use Generic[T] for Reusable Typed Containers

**Impact: HIGH** -- Preserves type information

When building reusable classes that operate on different types, use `Generic[T]` with `TypeVar` to preserve type information through the container. This avoids falling back to `Any` or `object` which lose type specificity.

**Incorrect (losing type information):**

```python
class ResultWrapper:
    def __init__(self, value: object, error: str | None = None) -> None:
        self.value = value
        self.error = error

    def unwrap(self) -> object:
        if self.error:
            raise ValueError(self.error)
        return self.value  # caller must cast or guess the type

result = ResultWrapper(value=User(name="Alice"))
user = result.unwrap()  # type is 'object', not 'User'
```

**Correct (using Generic[T]):**

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Result(Generic[T]):
    def __init__(self, value: T, error: str | None = None) -> None:
        self._value = value
        self._error = error

    def unwrap(self) -> T:
        if self._error:
            raise ValueError(self._error)
        return self._value

result = Result(value=User(name="Alice"))
user = result.unwrap()  # type checker knows this is 'User'
```

Generic containers preserve full type information, enabling autocompletion, refactoring support, and compile-time error detection.

---

### 3.7 Use NewType for Domain Identifiers

**Impact: MEDIUM** -- Prevents ID mix-ups

Use `NewType` to create distinct types for domain identifiers (user IDs, order IDs, etc.). This prevents accidentally passing a user ID where an order ID is expected, even though both are strings at runtime.

**Incorrect (raw strings for all IDs):**

```python
def get_order(order_id: str) -> Order: ...
def get_user(user_id: str) -> User: ...

# Bug: accidentally swapped arguments -- no type error
order = get_order(user_id)  # passes type check, fails at runtime
user = get_user(order_id)   # passes type check, fails at runtime
```

**Correct (NewType for distinct IDs):**

```python
from typing import NewType

UserId = NewType("UserId", str)
OrderId = NewType("OrderId", str)

def get_order(order_id: OrderId) -> Order: ...
def get_user(user_id: UserId) -> User: ...

user_id = UserId("usr_123")
order_id = OrderId("ord_456")

order = get_order(user_id)   # type error: expected OrderId, got UserId
user = get_user(order_id)    # type error: expected UserId, got OrderId
order = get_order(order_id)  # correct
```

`NewType` has zero runtime overhead -- it is an identity function at runtime but provides full compile-time type distinction.

---

### 3.8 Use Final for Constants

**Impact: MEDIUM** -- Prevents accidental reassignment

Mark module-level and class-level constants with `Final` to signal that they must never be reassigned. The type checker will flag any reassignment as an error, preventing accidental mutation of configuration values and magic numbers.

**Incorrect (plain variables for constants):**

```python
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30.0
API_VERSION = "v2"

# Somewhere deep in the codebase...
MAX_RETRIES = 10  # accidental reassignment, no warning
```

**Correct (Final annotation):**

```python
from typing import Final

MAX_RETRIES: Final = 3
DEFAULT_TIMEOUT: Final[float] = 30.0
API_VERSION: Final[str] = "v2"

# Somewhere deep in the codebase...
MAX_RETRIES = 10  # type error: cannot assign to final name

class HttpClient:
    BASE_URL: Final[str] = "https://api.example.com"
    TIMEOUT: Final[int] = 30

    def connect(self) -> None:
        self.BASE_URL = "http://other.com"  # type error
```

`Final` provides a clear contract that a value is not meant to change, catching mistakes that would otherwise lead to hard-to-debug configuration drift.

---

## Data Transfer Objects

### 4.1 Use Frozen Dataclasses for DTOs

**Impact: CRITICAL** -- Guarantees immutability

All Data Transfer Objects must use `@dataclass(frozen=True, slots=True)`. Frozen dataclasses are immutable after creation, preventing accidental mutation as data flows between layers. The `slots=True` option reduces memory usage and speeds up attribute access.

**Incorrect (mutable class or plain dict):**

```python
class UserDTO:
    def __init__(self, user_id: str, name: str, email: str) -> None:
        self.user_id = user_id
        self.name = name
        self.email = email

dto = UserDTO(user_id="123", name="Alice", email="alice@example.com")
dto.email = "hacked@evil.com"  # mutation allowed, causes bugs downstream
```

**Correct (frozen dataclass):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class UserDTO:
    user_id: str
    name: str
    email: str

dto = UserDTO(user_id="123", name="Alice", email="alice@example.com")
dto.email = "hacked@evil.com"  # raises FrozenInstanceError

# To create a modified copy, use dataclasses.replace
from dataclasses import replace
updated = replace(dto, email="alice@newdomain.com")
```

Immutable DTOs eliminate an entire class of bugs where data is accidentally changed in transit between services, layers, or threads.

---

### 4.2 Never Pass Raw Dicts Between Layers

**Impact: CRITICAL** -- Eliminates key typos

Never use plain `dict` objects to pass data between architectural layers (API, service, persistence). Raw dicts have no type safety, no autocompletion, and make typos in key names invisible until runtime.

**Incorrect (raw dicts flowing between layers):**

```python
# api/routes.py
def create_user_endpoint(request: Request) -> dict:
    data = request.json()
    result = user_service.create_user(data)  # dict in
    return {"id": result["id"], "name": result["nme"]}  # typo: "nme"

# service.py
def create_user(data: dict) -> dict:
    user = repo.save({"name": data["name"], "email": data["email"]})
    return {"id": user["id"], "name": user["name"]}  # no type safety
```

**Correct (typed DTOs):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CreateUserRequest:
    name: str
    email: str

@dataclass(frozen=True, slots=True)
class UserResponse:
    user_id: str
    name: str
    email: str

# api/routes.py
def create_user_endpoint(request: Request) -> UserResponse:
    dto = CreateUserRequest(name=request.json()["name"], email=request.json()["email"])
    return user_service.create_user(dto)

# service.py
def create_user(self, request: CreateUserRequest) -> UserResponse:
    user = self._repo.save(request.name, request.email)
    return UserResponse(user_id=user.id, name=user.name, email=user.email)
```

Typed DTOs turn runtime KeyError and typo bugs into compile-time type errors.

---

### 4.3 Separate DTOs per Layer

**Impact: HIGH** -- Decouples layer changes

Define distinct DTO types for each architectural layer: API request/response models, service-layer DTOs, and persistence models. Sharing a single model across layers creates tight coupling where a database schema change forces API changes and vice versa.

**Incorrect (single model shared across layers):**

```python
@dataclass
class User:
    id: int
    name: str
    email: str
    password_hash: str  # exposed to API layer
    created_at: datetime  # database concern in API
    internal_score: float  # internal detail leaked everywhere
```

**Correct (separate DTOs per layer):**

```python
from dataclasses import dataclass

# API layer
@dataclass(frozen=True, slots=True)
class CreateUserRequest:
    name: str
    email: str
    password: str

@dataclass(frozen=True, slots=True)
class UserResponse:
    user_id: str
    name: str
    email: str

# Service layer
@dataclass(frozen=True, slots=True)
class UserEntity:
    user_id: str
    name: str
    email: str
    password_hash: str

# Persistence layer
@dataclass(frozen=True, slots=True)
class UserRow:
    id: int
    user_id: str
    name: str
    email: str
    password_hash: str
    created_at: datetime
    internal_score: float
```

Each layer evolves independently. Adding a database column does not change the API contract, and changing the API response does not affect persistence.

---

### 4.4 Use Classmethod Factories for DTO Construction

**Impact: HIGH** -- Centralizes conversion logic

Use `@classmethod` factory methods to construct DTOs from different sources (database rows, API payloads, other DTOs). This keeps conversion logic co-located with the type definition and provides a clear, discoverable API for constructing objects.

**Incorrect (conversion logic scattered across codebase):**

```python
# In the API handler
user_dto = UserDTO(
    user_id=row["id"],
    name=row["name"],
    email=row["email"],
)

# In the service layer -- same conversion duplicated
user_dto = UserDTO(
    user_id=db_user.id,
    name=db_user.name,
    email=db_user.email,
)
```

**Correct (classmethod factories on the DTO):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class UserDTO:
    user_id: str
    name: str
    email: str

    @classmethod
    def from_db_row(cls, row: UserRow) -> "UserDTO":
        return cls(
            user_id=row.user_id,
            name=row.name,
            email=row.email,
        )

    @classmethod
    def from_api_payload(cls, payload: CreateUserRequest) -> "UserDTO":
        return cls(
            user_id=generate_id(),
            name=payload.name,
            email=payload.email,
        )

# Usage is clean and discoverable
user = UserDTO.from_db_row(row)
user = UserDTO.from_api_payload(request)
```

Factory classmethods centralize conversion logic, eliminate duplication, and make it easy to find all the ways a DTO can be constructed.

---

### 4.5 Validate Data at System Boundaries

**Impact: MEDIUM** -- Trusts internal data flow

Validate and sanitize data at the system boundaries (API endpoints, message consumers, file readers) where external input enters your system. Once data passes boundary validation and is converted to internal DTOs, trust those DTOs within the domain layer without re-validating.

**Incorrect (validation scattered everywhere):**

```python
class OrderService:
    def create_order(self, user_id: str, amount: float) -> Order:
        if not user_id:  # redundant if boundary already validated
            raise ValueError("user_id required")
        if amount <= 0:  # redundant validation
            raise ValueError("amount must be positive")
        if not isinstance(amount, (int, float)):  # redundant type check
            raise TypeError("amount must be numeric")
        return self._repo.save(Order(user_id=user_id, amount=amount))
```

**Correct (validate at boundary, trust internally):**

```python
# api/routes.py  (system boundary)
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    user_id: str = Field(min_length=1)
    amount: Decimal = Field(gt=0)

def create_order_endpoint(request: Request) -> OrderResponse:
    validated = CreateOrderRequest.model_validate(request.json())
    dto = CreateOrderInput(user_id=validated.user_id, amount=validated.amount)
    return order_service.create_order(dto)

# orders/service.py  (trusts validated DTO)
@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    user_id: str
    amount: Decimal

class OrderService:
    def create_order(self, input: CreateOrderInput) -> Order:
        # No re-validation needed -- boundary already guaranteed correctness
        return self._repo.save(Order(user_id=input.user_id, amount=input.amount))
```

Boundary validation eliminates defensive checks throughout the domain, keeping business logic clean and focused.

---

## Project Structure & Isolation

### 5.1 Top-Level Packages as Bounded Contexts

**Impact: HIGH** -- Navigable domain structure

Organize top-level Python packages to mirror your domain's bounded contexts. Each package (e.g., `users/`, `orders/`, `payments/`) contains its own domain models, services, repositories, and API routes. Shared infrastructure lives in a separate `infrastructure/` or `shared/` package.

**Incorrect (technical layer structure):**

```python
project/
    models/
        user.py
        order.py
        payment.py
    services/
        user_service.py
        order_service.py
        payment_service.py
    repositories/
        user_repo.py
        order_repo.py
        payment_repo.py
    api/
        user_routes.py
        order_routes.py
```

**Correct (domain package structure):**

```python
project/
    users/
        __init__.py
        domain.py        # User entity, value objects
        service.py       # UserService
        repository.py    # UserRepository protocol + implementation
        api.py           # user-related API routes
        interface.py     # protocols for cross-context use
    orders/
        __init__.py
        domain.py
        service.py
        repository.py
        api.py
    payments/
        __init__.py
        domain.py
        service.py
        repository.py
        api.py
    shared/
        __init__.py
        database.py      # shared DB engine, session factory
        settings.py      # typed application settings
    main.py              # composition root
```

A developer looking at the top-level directory immediately understands the business domains the system handles.

---

### 5.2 Use __all__ to Control Public API

**Impact: HIGH** -- Prevents internal leakage

Define `__all__` in every `__init__.py` to explicitly declare the public API of each package. This prevents internal implementation details from leaking to consumers and makes it clear what is intended for external use.

**Incorrect (no __all__, everything is public):**

```python
# users/__init__.py
from users.domain import User, UserStatus, _hash_password
from users.service import UserService
from users.repository import PostgresUserRepository, _build_query

# Consumers can import internal helpers
from users import _hash_password  # implementation detail exposed
from users import _build_query    # internal repo helper exposed
```

**Correct (explicit __all__):**

```python
# users/__init__.py
from users.domain import User, UserStatus
from users.service import UserService
from users.interface import UserServiceInterface

__all__ = [
    "User",
    "UserStatus",
    "UserService",
    "UserServiceInterface",
]

# Now:
from users import User            # works, part of public API
from users import _hash_password  # still importable but linters/IDEs warn
# from users import * only exports what's in __all__
```

Every `__init__.py` should have `__all__` listing only the symbols that form the package's public contract.

---

### 5.3 Prevent Circular Imports

**Impact: HIGH** -- Avoids import failures

Prevent circular imports by enforcing a strict dependency direction: API depends on Service, Service depends on Domain and Repository interfaces, Domain depends on nothing. If two modules need each other, extract the shared abstraction into a separate module.

**Incorrect (circular dependency):**

```python
# users/service.py
from users.repository import UserRepository  # service -> repo

class UserService:
    def get_user(self, user_id: str) -> User: ...

# users/repository.py
from users.service import UserService  # repo -> service (circular!)

class UserRepository:
    def __init__(self, service: UserService) -> None:
        self._service = service  # wrong: repos should not depend on services
```

**Correct (unidirectional dependency flow):**

```python
# users/domain.py  (depends on nothing)
@dataclass(frozen=True, slots=True)
class User:
    user_id: str
    name: str
    email: str

# users/interfaces.py  (depends on domain only)
from typing import Protocol
from users.domain import User

class UserRepository(Protocol):
    def find_by_id(self, user_id: str) -> User | None: ...
    def save(self, user: User) -> None: ...

# users/service.py  (depends on domain + interfaces)
from users.domain import User
from users.interfaces import UserRepository

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

# users/postgres_repo.py  (depends on domain + interfaces)
from users.domain import User
from users.interfaces import UserRepository

class PostgresUserRepository:
    def find_by_id(self, user_id: str) -> User | None: ...
    def save(self, user: User) -> None: ...
```

The dependency graph flows in one direction: `api -> service -> interfaces <- implementations`.

---

### 5.4 Centralize Config in Typed Settings

**Impact: HIGH** -- Eliminates scattered env reads

Centralize all configuration in a typed dataclass or Pydantic `BaseSettings` class. Never read environment variables directly with `os.getenv()` scattered throughout the codebase. A single settings object provides type safety, validation, and a clear inventory of all configuration.

**Incorrect (scattered env reads):**

```python
import os

class OrderService:
    def process(self) -> None:
        max_retries = int(os.getenv("MAX_RETRIES", "3"))  # scattered
        timeout = float(os.getenv("ORDER_TIMEOUT", "30"))  # no validation
        api_key = os.getenv("PAYMENT_API_KEY")  # might be None

class UserService:
    def __init__(self) -> None:
        self.db_url = os.getenv("DATABASE_URL")  # duplicated pattern
```

**Correct (typed settings class):**

```python
from dataclasses import dataclass
import os

@dataclass(frozen=True, slots=True)
class Settings:
    database_url: str
    payment_api_key: str
    max_retries: int = 3
    order_timeout: float = 30.0

    @classmethod
    def from_env(cls) -> "Settings":
        return cls(
            database_url=cls._require_env("DATABASE_URL"),
            payment_api_key=cls._require_env("PAYMENT_API_KEY"),
            max_retries=int(os.getenv("MAX_RETRIES", "3")),
            order_timeout=float(os.getenv("ORDER_TIMEOUT", "30.0")),
        )

    @staticmethod
    def _require_env(key: str) -> str:
        value = os.getenv(key)
        if value is None:
            raise RuntimeError(f"Missing required env var: {key}")
        return value

# main.py (composition root)
settings = Settings.from_env()  # validated once at startup
order_service = OrderService(max_retries=settings.max_retries)
```

All configuration is discoverable in one place, validated at startup, and injected into services as typed values.

---

## Clean Code

### 6.1 Prefer Composition Over Inheritance

**Impact: MEDIUM** -- Reduces fragile hierarchies

Favor composition and delegation over deep inheritance hierarchies. Inheritance creates tight coupling between parent and child classes, making changes to the base class risky. Composition allows mixing behaviors flexibly and testing components in isolation.

**Incorrect (deep inheritance hierarchy):**

```python
class BaseRepository:
    def connect(self) -> None: ...
    def disconnect(self) -> None: ...

class CachedRepository(BaseRepository):
    def get_from_cache(self, key: str) -> object: ...

class LoggedCachedRepository(CachedRepository):
    def log(self, message: str) -> None: ...

class UserRepository(LoggedCachedRepository):
    def find_user(self, user_id: str) -> User:
        self.log(f"Finding user {user_id}")
        cached = self.get_from_cache(user_id)
        if cached:
            return cached
        self.connect()
        # ... fragile chain of inherited behavior
```

**Correct (composition with delegation):**

```python
from typing import Protocol

class Cache(Protocol):
    def get(self, key: str) -> object | None: ...
    def set(self, key: str, value: object) -> None: ...

class Logger(Protocol):
    def info(self, message: str) -> None: ...

class UserRepository:
    def __init__(
        self,
        db: DatabaseConnection,
        cache: Cache,
        logger: Logger,
    ) -> None:
        self._db = db
        self._cache = cache
        self._logger = logger

    def find_user(self, user_id: str) -> User:
        self._logger.info(f"Finding user {user_id}")
        cached = self._cache.get(user_id)
        if cached is not None:
            return cached
        user = self._db.query(User, user_id)
        self._cache.set(user_id, user)
        return user
```

Composition keeps each concern (caching, logging, persistence) independently testable and replaceable.

---

### 6.2 Use Guard Clauses and Early Returns

**Impact: MEDIUM** -- Reduces nesting depth

Use guard clauses to handle error conditions and edge cases at the top of a function, then return or raise early. This eliminates deep nesting and keeps the "happy path" at the lowest indentation level, improving readability.

**Incorrect (deeply nested conditionals):**

```python
def process_order(order: Order | None, user: User | None) -> OrderResult:
    if order is not None:
        if user is not None:
            if order.status == OrderStatus.PENDING:
                if user.is_active:
                    if order.total > Decimal("0"):
                        result = payment_service.charge(user, order.total)
                        if result.success:
                            return OrderResult(status="completed")
                        else:
                            return OrderResult(status="payment_failed")
                    else:
                        return OrderResult(status="invalid_total")
                else:
                    return OrderResult(status="inactive_user")
            else:
                return OrderResult(status="not_pending")
        else:
            return OrderResult(status="no_user")
    else:
        return OrderResult(status="no_order")
```

**Correct (guard clauses with early return):**

```python
def process_order(order: Order | None, user: User | None) -> OrderResult:
    if order is None:
        return OrderResult(status="no_order")
    if user is None:
        return OrderResult(status="no_user")
    if order.status != OrderStatus.PENDING:
        return OrderResult(status="not_pending")
    if not user.is_active:
        return OrderResult(status="inactive_user")
    if order.total <= Decimal("0"):
        return OrderResult(status="invalid_total")

    result = payment_service.charge(user, order.total)
    if not result.success:
        return OrderResult(status="payment_failed")

    return OrderResult(status="completed")
```

The guard clause version reads top-to-bottom with no nesting, making the logic immediately clear.

---

### 6.3 Define Domain-Specific Exception Hierarchy

**Impact: MEDIUM** -- Enables precise error handling

Define a custom exception hierarchy for each bounded context. Never use bare `except:` or catch `Exception` broadly. Domain exceptions make error handling precise, self-documenting, and allow callers to handle specific failure modes.

**Incorrect (generic exceptions and bare except):**

```python
class OrderService:
    def place_order(self, user_id: str, amount: Decimal) -> Order:
        try:
            user = self._user_service.get_user(user_id)
            if user.credit < amount:
                raise Exception("Not enough credit")  # generic
            return self._repo.save(Order(user_id=user_id, amount=amount))
        except:  # bare except catches SystemExit, KeyboardInterrupt
            print("Something went wrong")
            return None
```

**Correct (domain exception hierarchy):**

```python
# orders/exceptions.py
class OrderError(Exception):
    """Base exception for the orders bounded context."""

class InsufficientCreditError(OrderError):
    def __init__(self, user_id: str, required: Decimal, available: Decimal) -> None:
        self.user_id = user_id
        self.required = required
        self.available = available
        super().__init__(
            f"User {user_id} needs {required} but has {available}"
        )

class OrderNotFoundError(OrderError):
    def __init__(self, order_id: str) -> None:
        self.order_id = order_id
        super().__init__(f"Order {order_id} not found")

# orders/service.py
class OrderService:
    def place_order(self, user_id: str, amount: Decimal) -> Order:
        user = self._user_service.get_user(user_id)
        if user.credit < amount:
            raise InsufficientCreditError(user_id, amount, user.credit)
        return self._repo.save(Order(user_id=user_id, amount=amount))

# api/routes.py
def create_order_endpoint(request: Request) -> Response:
    try:
        order = order_service.place_order(request.user_id, request.amount)
        return Response(status=201, body=order)
    except InsufficientCreditError as exc:
        return Response(status=402, body={"error": str(exc)})
    except OrderError as exc:
        return Response(status=400, body={"error": str(exc)})
```

Domain exceptions carry context, enable precise error handling, and never swallow unexpected errors.
