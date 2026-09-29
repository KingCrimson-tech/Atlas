# OOP Week 2 - Polymorphism, Relations, Copy, Typing, Design

## 1. Polymorphism - one interface, many behaviors

> **Definition:** Polymorphism (Greek: many forms) is the ability of a single interface, method name, or operator to exhibit different behaviors depending on the object type or arguments it operates on.
> **In simple terms:** Same call `send(msg)`, different result for Email vs SMS. Same Play button in Spotify vs YouTube.
> **Why it matters:** Lets you write generic code (`deploy(uploader)`) that works with new types without modification — core of plugins and frameworks.

**Software ex:** `render()` on `MarkdownRenderer` vs `HtmlRenderer`. `send()` on `EmailNotifier` vs `SMSNotifier`.

## 2. Compile-time Polymorphism (static / early binding)

> **Definition:** Compile-time (static) polymorphism resolves which method/operator to invoke before program execution (at compile time). Achieved via function overloading and operator overloading.
> **In simple terms:** Compiler picks the version based on argument count/type.
> **Why it matters:** Faster dispatch, but Python has no true compile-time — you simulate it, and interviews test if you know that.

### a) Function Overloading

> **Definition:** Function overloading is defining multiple methods with the same name but different parameter counts or types within the same scope.
> **In simple terms:** `add(a,b)` and `add(a,b,c)` — same name, different inputs.
> **Why it matters:** Common in Java/C++; in Python you must use defaults/`*args`/`singledispatch` instead.

```python
# defaults + *args (Python way)
class Analytics:
    def track(self, event, user_id=None, props=None):
        print(event, user_id, props or {})

# type-based dispatch
from functools import singledispatchmethod

class Serializer:
    @singledispatchmethod
    def dump(self, data):
        raise TypeError("unsupported")

    @dump.register
    def _(self, data: dict):
        return f"json:{data}"

    @dump.register
    def _(self, data: list):
        return f"csv:{data}"
```

Pros: readable, less duplication. Cons: overuse = ambiguity, hard debug.

### b) Operator Overloading

> **Definition:** Operator overloading is redefining the behavior of built-in operators (`+`, `==`, `len()`) for user-defined types by implementing dunder methods (`__add__`, `__eq__`, `__len__`).
> **In simple terms:** Teach `+` to add `Money` objects, not just ints.
> **Why it matters:** Makes domain types (`Money`, `Version`, `Vector`) readable instead of `m1.add(m2)`.

**Software ex:** `Money(10, "INR") + Money(5, "INR")`, `Version("1.2") < Version("2.0")`.

```python
class Money:
    def __init__(self, amount: float, currency="INR"):
        self.amount = amount
        self.currency = currency

    def __add__(self, other: "Money"):
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    def __repr__(self):
        return f"{self.amount} {self.currency}"

print(Money(10) + Money(5))  # 15 INR
```

Pros: natural syntax. Cons: confusing if abused, harder debug.

## 3. Runtime Polymorphism (dynamic / late binding)

> **Definition:** Runtime (dynamic) polymorphism resolves which method to invoke during program execution based on the actual object type, achieved via method overriding and dynamic binding.
> **In simple terms:** `uploader.upload()` decides S3 vs local only when it runs.
> **Why it matters:** Enables extensibility — add a new uploader class without touching `deploy()`.

### a) Function Overriding

> **Definition:** Function overriding is redefining a parent-class method in a child class with the same name and signature to provide specialized behavior.
> **In simple terms:** Parent says generic `upload()`, child replaces it with S3 logic.
> **Why it matters:** Basis for frameworks: base view/handler + custom subclass behavior.

```python
class BaseUploader:
    def upload(self, file: bytes) -> str:
        raise NotImplementedError

class S3Uploader(BaseUploader):
    def upload(self, file: bytes) -> str:
        return "s3://bucket/key"

class LocalUploader(BaseUploader):
    def upload(self, file: bytes) -> str:
        return "/tmp/key"
```

### b) Dynamic Binding / Duck Typing

> **Definition:** Dynamic (late) binding is the runtime mechanism of selecting the method implementation based on the actual object, not the reference type. Python's version is duck typing: "if it has `upload()`, it qualifies."
> **In simple terms:** No `virtual` keyword needed — Python figures it out at call time.
> **Why it matters:** Explains why Python needs no interfaces for polymorphism, but tracing bugs is harder.

```python
def deploy(uploader: BaseUploader, file: bytes):
    print(uploader.upload(file))  # bound at runtime

deploy(S3Uploader(), b"data")
deploy(LocalUploader(), b"data")
```

Pros: flexible, extensible. Cons: slower, more memory, harder to trace.

| Compile-time | Runtime |
|---|---|
| resolved before run | resolved during run |
| overloading, operators | overriding, duck typing |
| faster | slightly slower |
| no inheritance needed | needs parent/Protocol |
| ex: `Serializer.dump(dict/list)` | ex: `uploader.upload()` |

**Overloading vs Overriding:**
> **Definition:** Overloading = same name, different params, same class, static binding. Overriding = same name, same signature, parent->child, dynamic binding.
> **Why it matters:** Most-asked polymorphism comparison.

| | Overloading | Overriding |
|---|---|---|
| What | same name, diff params | child redefines parent method |
| Where | same class | parent -> child |
| Binding | static | dynamic |
| Python | simulate via defaults/singledispatch | native via inheritance |

## 4. Object Relationships

> **Definition:** Object relationships describe how objects connect and own each other: Association (uses), Aggregation (has, weak), Composition (owns, strong).
> **Why it matters:** System-design interviews judge whether you pick weak vs strong ownership correctly.

### a) Association - Uses-A, independent lifecycles

> **Definition:** Association is a loose relationship where one object uses another for an operation, but both exist independently; destroying one does not destroy the other.
> **In simple terms:** `AuthService` borrows `Logger`.
> **Why it matters:** Models service dependencies without ownership coupling.

```python
class Logger:
    def log(self, msg: str): print(msg)

class AuthService:
    def __init__(self, logger: Logger):
        self.logger = logger
    def login(self, user: str):
        self.logger.log(f"{user} logged in")
```

### b) Aggregation - Has-A, weak ownership

> **Definition:** Aggregation is a Has-A relationship where a parent contains child objects passed in from outside; children survive if the parent is destroyed.
> **In simple terms:** `Team` has `Developer`s, but devs exist even if team disbands.
> **Why it matters:** Correct for collections of independent entities (team members, library books).

```python
class Developer:
    def __init__(self, name: str): self.name = name

class Team:
    def __init__(self, devs: list[Developer]):
        self.devs = devs  # passed in from outside
```

### c) Composition - Part-Of, strong ownership

> **Definition:** Composition is a strong Part-Of relationship where the parent creates and owns its children; children cannot exist independently and die with the parent.
> **In simple terms:** `Order` creates its `LineItem`s; delete order, items vanish.
> **Why it matters:** Enforces lifecycle + data integrity (order items, document paragraphs).

```python
class LineItem:
    def __init__(self, sku: str, qty: int):
        self.sku, self.qty = sku, qty

class Order:
    def __init__(self, items: list[tuple[str, int]]):
        self.items = [LineItem(s, q) for s, q in items]  # created inside
```

Rule: Association < Aggregation < Composition (increasing ownership).

## 5. Shallow vs Deep Copy

> **Definition:** A shallow copy creates a new top-level object but shares references to nested objects; a deep copy recursively clones the object and all nested objects into fully independent memory.
> **In simple terms:** Shallow = new folder, same shared files. Deep = full photocopy.
> **Why it matters:** Prevents aliasing bugs where editing a "copy" corrupts the original (common in configs, caches, test fixtures).

```python
import copy
orig = {"user": "ram", "tags": ["admin", "dev"]}

shallow = copy.copy(orig)      # new dict, same inner list
deep = copy.deepcopy(orig)     # new dict + new inner list

shallow["tags"].append("x")  # mutates orig too!
deep["tags"].append("y")     # orig unaffected
```

| | Shallow | Deep |
|---|---|---|
| Copies | top level only | recursively |
| Memory | shared | separate |
| Speed | fast, low mem | slow, high mem |
| Risk | side effects | safe |

Use `copy.copy` for flat configs, `deepcopy` for nested state (cloning request context, test fixtures).

## 6. Static vs Dynamic Typing

> **Definition:** Static typing checks variable types at compile time (types declared, errors early). Dynamic typing checks types at runtime (no declarations, variables can change type).
> **In simple terms:** Java catches type errors before running; Python catches them while running.
> **Why it matters:** Explains Python's speed of development vs Java's safety; justify type hints + mypy.

```python
x = 10
x = "now a string"  # legal in Python, illegal in Java
```

Mitigate with hints + mypy: `def charge(amount: float) -> bool: ...`

| | Static | Dynamic |
|---|---|---|
| Check | compile | runtime |
| Flex | low | high |
| Errors | early | late |
| Speed | faster | slower |
| Ex | Java, C++ | Python, JS |

## 7. Strong vs Weak Typing

> **Definition:** Strong typing forbids implicit conversion between unrelated types (must convert explicitly). Weak typing automatically coerces types when needed.
> **In simple terms:** Python refuses `"total: " + 100`; JavaScript silently gives `"total: 100"`.
> **Why it matters:** Python is dynamic + strong — flexible but safe from silent coercion bugs; key interview distinction.

```python
# Python (strong): raises TypeError
"total: " + 100  # TypeError, must do "total: " + str(100)
# JS (weak): "total: 100" via implicit coercion
```

| | Strong | Weak |
|---|---|---|
| Rules | strict, explicit conversion | auto-coerce |
| Safety | high | low |
| Bugs | fewer silent bugs | surprising outputs |
| Ex | Python | JavaScript, PHP |

## 8. Coupling & Cohesion

> **Definition:** Coupling measures dependency between modules (want low). Cohesion measures how focused the responsibilities inside a single module are (want high).
> **In simple terms:** Low coupling = USB charger works with any phone. High cohesion = calculator app only calculates.
> **Why it matters:** The one-line design quality test: "low coupling, high cohesion" is what reviewers look for.

**Coupling:** High = `OrderService` directly creates `MySQLConnection`, `RazorpayClient`. Low = inject interfaces.

```python
# low coupling via DI
class OrderService:
    def __init__(self, db, gateway):
        self.db, self.gateway = db, gateway
```

**Cohesion:** High = `PasswordHasher` only hashes/verifies. Low = `Utils` does hashing + email + PDF.

| | Coupling | Cohesion |
|---|---|---|
| Measures | inter-module dependency | intra-module focus |
| Want | low | high |
| Bad | change ripples everywhere | god class, hard test |
| Fix | DI, interfaces | single responsibility |

## 9. Encapsulation vs Abstraction

> **Definition:** Encapsulation wraps data + methods and restricts access (data hiding for security). Abstraction hides implementation and exposes only essential behavior (complexity hiding for simplicity).
> **In simple terms:** Encapsulation = pill capsule protects medicine. Abstraction = car steering hides engine.
> **Why it matters:** Most-confused pair in interviews; one line separates them: encapsulation hides data, abstraction hides how.

| | Encapsulation | Abstraction |
|---|---|---|
| What | wrap data + methods, restrict access | hide how, show what |
| Hides | data | implementation |
| Via | classes, `_`/`__`, `@property` | ABC, Protocol |
| Goal | security, validation | simplicity |
| Ex | `BankAccount.__balance` | `PaymentGateway.charge()` |
| Level | implementation | design |

Encapsulation supports abstraction: hide data so you can expose a clean API.
