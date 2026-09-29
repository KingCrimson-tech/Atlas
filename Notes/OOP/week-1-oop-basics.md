# OOP Week 1 - Basics, Constructor, Encapsulation, Abstraction, Inheritance

## 1. POP vs OOP

> **Definition:** POP (Procedural Oriented Programming) is a paradigm where a program is structured as a sequence of procedures/functions operating on shared data. OOP (Object-Oriented Programming) is a paradigm where a program is structured as a collection of objects, each bundling data and the methods that operate on it.
> **In simple terms:** POP = functions + global data. OOP = objects talking to objects.
> **Why it matters:** Interviews ask this first to check if you know when to use scripts vs modular systems.

- **POP:** Top-down. Break problem into functions. Data shared via globals/params. Ex: C script with global `users[]` mutated by `add_user()`, `delete_user()`.
- **OOP:** Bottom-up. Model entities as classes, compose system from objects. Ex: `User`, `Order`, `PaymentGateway` interacting.

| | POP | OOP |
|---|---|---|
| Unit | functions | classes / objects |
| Focus | procedure | data + behavior |
| Data | global, exposed | hidden, via methods |
| Reuse | functions | inheritance, composition |
| Modeling | hard | natural |
| Best for | small scripts | large systems |
| Ex | C | Python, Java |

## 2. Class, Object, Attributes, Methods

> **Definition:** A class is a user-defined blueprint that defines attributes (state) and methods (behavior) for its instances. An object is an instance of a class with its own state, behavior, and identity.
> **In simple terms:** Class = design of an invoice form. Object = filled invoice INV-101.
> **Why it matters:** Every OOP question builds on this; confusing class vs object loses marks instantly.

- **Class:** No memory until instantiated.
- **Object:** Has state (values), behavior (methods), identity (`id(obj)`).
- **Attribute:** Variable storing object state.
- **Method:** Function inside a class defining behavior.

**Software ex:** `Invoice` class -> objects `inv_101`, `inv_102`.

```python
class Invoice:
    def __init__(self, inv_id: str, amount: float):
        self.inv_id = inv_id      # attribute
        self.amount = amount      # attribute
        self.paid = False

    def mark_paid(self):          # method = behavior
        self.paid = True

inv_101 = Invoice("INV-101", 499.0)
inv_101.mark_paid()
```

### Message Passing

> **Definition:** Message passing is communication between objects where one object invokes a method on another object, optionally passing arguments and receiving a result.
> **In simple terms:** Calling `gateway.charge(500)` = sending "charge 500" message to gateway.
> **Why it matters:** Explains how decoupled services interact; basis for MVC, microservices.

Ex: `checkout_service.process(order)` calls `payment_gateway.charge(amount)`.

## 3. The 4 Pillars

> **Definition:** The four foundational mechanisms of OOP are: (1) Encapsulation — bundling data with methods and restricting access, (2) Abstraction — exposing what an object does while hiding how, (3) Inheritance — acquiring properties/behavior from an existing class, (4) Polymorphism — same interface exhibiting multiple behaviors.
> **In simple terms:** Hide data, hide complexity, reuse code, same call different result.
> **Why it matters:** Most common OOP interview question: "What are the 4 pillars with examples?"

## 4. Constructor - `__init__`

> **Definition:** A constructor is a special method automatically invoked at object creation to initialize the object's state. In Python it is `__init__(self, ...)` (allocation is `__new__`).
> **In simple terms:** Setup that runs when you create an object, so it never starts empty/invalid.
> **Why it matters:** Used to enforce required fields (ex: every `Order` must have `user_id`) and avoid scattered init code.

```python
class DatabaseConnection:
    def __init__(self, host="localhost", port=5432):
        self.host = host
        self.port = port
        self.connected = False
```

Rules: auto-called, no return value, one `__init__` per class in Python (no true overloading — use defaults).

### Types

> **Definition:** Default constructor takes no extra arguments; parameterized takes arguments; copy constructor creates a new object from an existing one; private constructor restricts instantiation to inside the class; constructor overloading provides multiple constructors with different signatures.
> **In simple terms:** Defaults = factory settings; parameterized = custom order; copy = photocopy; private = internal-only creation.
> **Why it matters:** Maps directly to real APIs: defaults for config, params for user input, copy for cloning.

- **Default:** `User()` uses defaults. Ex: fresh app install with default settings.
- **Parameterized:** `User("ram", "admin")`. Ex: signup form input.
- **Copy:** Python has no built-in copy ctor; use `copy` module or classmethod.

```python
import copy

class AppConfig:
    def __init__(self, theme, plugins: list):
        self.theme = theme
        self.plugins = plugins

    @classmethod
    def from_existing(cls, other: "AppConfig"):
        return cls(other.theme, list(other.plugins))

c1 = AppConfig("dark", ["git"])
c2 = AppConfig.from_existing(c1)
c3 = copy.deepcopy(c1)
```

- **Private ctor:** No `private` in Python. Use `__new__` singleton when only one instance should exist (logger, config manager).

```python
class Logger:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

- **Overloading:** Not native in Python. Simulate with defaults / `*args`.

```python
class SearchAPI:
    def __init__(self, query, limit=10, filters=None):
        self.query = query
        self.limit = limit
        self.filters = filters or {}
```

## 5. `self` (this keyword)

> **Definition:** `self` (called `this` in Java/C++) is a reference to the current object, automatically passed as the first parameter of instance methods, used to access instance state and methods.
> **In simple terms:** `self.token` = "my token", `token` = local argument.
> **Why it matters:** Explains Python method signatures and fixes the classic "missing self" bug.

Uses: access instance vars, disambiguate locals, call other methods, pass current object.

```python
class Session:
    def __init__(self, token: str):
        self.token = token

    def is_valid(self):
        return bool(self.token)
```

## 6. Encapsulation

> **Definition:** Encapsulation is the bundling of data (attributes) and the methods operating on that data into a single unit (class), while restricting direct access to internal state and exposing only controlled access points.
> **In simple terms:** Same as a Stripe SDK: internals hidden, you only use `charge()` with validation.
> **Why it matters:** Prevents invalid state (negative balance) and lets you change internals without breaking callers.

Python conventions (no true access modifiers):
- `x` public
- `_x` protected (convention only)
- `__x` private (name-mangled to `_Class__x`)

### Data Hiding

> **Definition:** Data hiding is the practice of making attributes inaccessible directly from outside the class (via `private`/`__`), so they can only be touched through methods.
> **Why it matters:** Stops external code from corrupting state.

### Getters / Setters

> **Definition:** Getters and setters are public methods (or `@property`) that provide controlled read/write access to private data, with validation and side effects.
> **Why it matters:** Lets you add validation/logging later without changing the API.

### Controlled Access

> **Definition:** Controlled access means all state changes go through validated methods rather than direct assignment.
> **Why it matters:** Core of secure APIs — you deposit via `deposit()`, never by setting `balance` directly.

```python
class BankAccount:
    def __init__(self, balance: float):
        self.__balance = balance  # private

    @property
    def balance(self):            # getter
        return self.__balance

    def deposit(self, amt: float):  # controlled setter
        if amt <= 0:
            raise ValueError("amount must be > 0")
        self.__balance += amt

    def withdraw(self, amt: float):
        if amt > self.__balance:
            raise ValueError("insufficient funds")
        self.__balance -= amt

acc = BankAccount(1000)
acc.deposit(500)
print(acc.balance)  # 1500
```

Pros: security, validation, maintainability, modularity.
Cons: more boilerplate, slight overhead, over-restriction hurts flexibility.

**Data Hiding vs Encapsulation:**
> **Definition:** Data hiding restricts direct access to data; encapsulation is the broader design of wrapping data + methods + access control into a maintainable unit.
> **Why it matters:** Interview trap — hiding is a tactic, encapsulation is the principle.

| Data Hiding | Encapsulation |
|---|---|
| restrict access (`__x`) | wrap data + methods + access |
| narrow, security-focused | broad, design + security |
| via `private`/`__` | via classes, properties, methods |

## 7. Abstraction - What, not How

> **Definition:** Abstraction is the process of exposing only essential features (what an object does) while hiding internal implementation details (how it does it).
> **In simple terms:** `requests.post(url, json=...)` — you don't handle sockets/TLS.
> **Why it matters:** Lets teams use a service without understanding internals; enables swapping implementations.

### Abstract Class

> **Definition:** An abstract class (via `abc.ABC`) is a class that cannot be instantiated and may declare abstract methods (no body) plus concrete methods (with body), forcing subclasses to implement the abstract part.
> **In simple terms:** Template with blanks: "every payment gateway must implement `charge()`, here's shared `receipt()` for free."
> **Why it matters:** Enforces contracts while sharing common code.

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: float) -> bool: ...

    def receipt(self):  # concrete shared logic
        return "receipt sent"

class RazorpayGateway(PaymentGateway):
    def charge(self, amount: float) -> bool:
        print(f"charging {amount} via Razorpay")
        return True
```

### Interface

> **Definition:** An interface defines only method signatures with no implementation. In Python it is expressed as a `Protocol` (structural typing) or a pure-ABC (all-abstract). A class "implements" it by providing those methods.
> **In simple terms:** A capability checklist: "anything with `send(msg)` counts as a Notifier."
> **Why it matters:** Enables polymorphism across unrelated classes and clean dependency injection.

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, msg: str) -> None: ...

class EmailNotifier:
    def send(self, msg: str) -> None:
        print(f"email: {msg}")

def alert(n: Notifier, msg: str):
    n.send(msg)
```

| | Abstract class | Interface / Protocol |
|---|---|---|
| Contains | abstract + concrete | only signatures |
| Instantiation | no | no |
| Use when | shared base logic | unrelated classes, same capability |
| Abstraction | partial | full |

## 8. Inheritance - IS-A reuse

> **Definition:** Inheritance is a mechanism where a child (derived) class acquires attributes and methods of a parent (base) class, and can reuse, extend, or override them.
> **In simple terms:** `NotFoundError` gets everything from `ApiError` plus its own status code.
> **Why it matters:** Primary reuse tool; also the most abused — prefer shallow hierarchies and composition.

- **Parent:** class being inherited from. **Child:** class inheriting.
- **IS-A:** `Admin IS-A User`. If not true, use composition (HAS-A) instead.

```python
class ApiError(Exception):
    def __init__(self, msg, status=500):
        super().__init__(msg)
        self.status = status

class NotFoundError(ApiError):
    def __init__(self, msg="not found"):
        super().__init__(msg, status=404)
```

### `super()`

> **Definition:** `super()` returns a proxy to the parent class, used to call parent constructors/methods from the child, especially in overriding and cooperative multiple inheritance.
> **Why it matters:** Ensures parent init runs; required for correct MRO in Django/DRF views, exception hierarchies.

### Types

> **Definition:** Single = one parent; Multiple = several parents; Hierarchical = one parent, many children; Multilevel = chain of parents; Hybrid = combination of the above.
> **Why it matters:** Asked with examples; multiple/hybrid introduce MRO/diamond problems.

```python
# 1. Single: PostgresPool -> ConnectionPool
class ConnectionPool: ...
class PostgresPool(ConnectionPool): ...

# 2. Multiple: Admin(User, Permissions)
class User: ...
class Permissions: ...
class Admin(User, Permissions): ...

# 3. Hierarchical: EmailAlert(BaseAlert), SMSAlert(BaseAlert)
class BaseAlert: ...
class EmailAlert(BaseAlert): ...
class SMSAlert(BaseAlert): ...

# 4. Multilevel: Request -> AuthRequest -> AdminRequest
class Request: ...
class AuthRequest(Request): ...
class AdminRequest(AuthRequest): ...

# 5. Hybrid: mix of above (check Admin.__mro__)
```

Check MRO: `Admin.__mro__`. Prefer composition over deep hierarchies.
