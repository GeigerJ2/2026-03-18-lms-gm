---
theme: seriph
background: https://cover.sli.dev
title: Why Types and API Matter
class: text-center
transition: slide-left
comark: true
duration: 35min
---

# Why Types and API Matter

Building better Python through intentional design

---

# Why should we care?

Your library's API is the **surface area** that every user touches

- A bad API leads to bugs, frustration, and misuse
- A good API makes the "right thing" easy and the "wrong thing" hard
- Types are the **machine-checkable** part of that contract

<v-click>

> "APIs, like diamonds, are forever." — Joshua Bloch

</v-click>

---

# The urllib vs requests story

A classic tale of API design

```python
# urllib — verbose, error-prone, easy to forget steps
import urllib.request
req = urllib.request.Request(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer tok123"},
)
with urllib.request.urlopen(req) as response:
    body = response.read().decode("utf-8")
```

<v-click>

```python
# requests — clean, discoverable, hard to misuse
import requests
resp = requests.get(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer tok123"},
)
body = resp.text
```

</v-click>

<v-click>

Same task. Wildly different experience. **API design matters.**

</v-click>

---

# Lessons from Pythonic API design

Key principles from Ben Hoyt's guide

- **Flat over nested** — `fishnchips.order()` not `fishnchips.api.v2.orders.create()`
- **Keyword args with defaults** over global configuration
- **Classes for state** — avoid module-level globals
- **Custom exception hierarchies** — don't return error codes
- **Short, clear names** — functions are verbs, classes are nouns

<v-click>

```python
# Good: keyword args with sensible defaults
def order(chips: int = 0, fish: int = 0, timeout: float = 10.0) -> Receipt:
    ...

# Bad: global config that affects all callers
DEFAULT_TIMEOUT = 10  # please don't mutate me
```

</v-click>

<div class="abs-br m-6 text-sm opacity-60">
  <a href="https://benhoyt.com/writings/python-api-design/" target="_blank">
    benhoyt.com/writings/python-api-design
  </a>
</div>

---

# Function signatures are your API contract

The parallel to C header files

In C, the **header file** declares the contract — types in, types out:

```c
// shop.h — this IS the API
Receipt order(int chips, int fish, float timeout);
```

<v-click>

In Python, **type-annotated function signatures** serve the same purpose:

```python
# shop.py — the signature IS the documentation
def order(chips: int, fish: int, timeout: float = 10.0) -> Receipt:
    ...
```

</v-click>

<v-click>

- Users read signatures **before** reading docstrings
- IDEs show signatures on hover — they're the first thing people see
- A well-typed signature answers: *what do I pass in, what do I get back?*
- **If your signature needs a paragraph to explain, redesign the API**

</v-click>

---

# Types catch bugs

Adding types to existing code reveals hidden issues

```python
# Before types — "works fine" (or does it?)
def get_user_name(user_id):
    user = db.find(user_id)
    if user:
        return user.name
    return None    # <- caller probably doesn't handle this
```

<v-click>

```python
# After adding types — the bug is now visible
def get_user_name(user_id: int) -> str:  # pyright: error!
    user = db.find(user_id)
    if user:
        return user.name
    return None  # None is not compatible with str
```

</v-click>

<v-click>

The type checker **forces you to decide**: should this return `str | None`? Or should it raise on missing users? Either way, the caller now knows what to expect.

</v-click>

---

# Types reveal design smells

When the return type is too broad, your code needs refactoring

```python
def process_payment(order: Order) -> str | int | dict | None:
    if order.method == "card":
        return {"tx_id": "abc", "status": "ok"}
    elif order.method == "cash":
        return 42  # receipt number
    elif order.method == "credit":
        return "pending"
    return None
```

<v-click>

If you struggle to write the return type, **the function is doing too many things**.

</v-click>

<v-click>

```python
@dataclass
class PaymentResult:
    tx_id: str
    status: str
    receipt_number: int

def process_payment(order: Order) -> PaymentResult:
    ...  # one clear return type, one clear responsibility
```

</v-click>

---

# TypedDict and dataclasses over plain dicts

Structure your data, don't just bag it

```python
# The "stringly typed" approach — no safety, no autocomplete
user = {"name": "Alice", "age": 30, "emial": "alice@example.com"}
#                                      ^^^^^ typo goes unnoticed
```

<v-click>

```python
# TypedDict — lightweight, catches key typos at check time
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int
    email: str

user: User = {"name": "Alice", "age": 30, "emial": "..."}
#                                          ^^^^^^ error!
```

</v-click>

<v-click>

```python
# dataclass — full type safety + methods + immutability option
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    name: str
    age: int
    email: str
```

</v-click>

---

# Protocols and abstract interfaces

Depend on behavior, not implementation

```python
from typing import Protocol

class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

def process_data(source: Readable) -> None:
    data = source.read()
    ...
```

<v-click>

Any object with a `.read()` method satisfies `Readable` — **no inheritance required**.

</v-click>

<v-click>

- Enables **dependency inversion** — depend on abstractions, not concretions
- Makes code **testable** — pass a fake that matches the protocol
- Matches Python's duck-typing philosophy, but **checked statically**
- Similar to Go interfaces or Rust traits

</v-click>

---

# NewType — preventing value mix-ups

Semantically different values deserve different types

```python
from typing import NewType

UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)

def get_order(order_id: OrderId) -> Order: ...

user_id = UserId(42)
get_order(user_id)  # error: UserId is not compatible with OrderId
```

<v-click>

- **Zero runtime cost** — `NewType` is erased at runtime
- Prevents an entire class of "wrong ID" bugs
- Especially valuable in codebases with many ID-like parameters

</v-click>

---

# Enums and Literals over bare strings

Constrain your inputs

```python
# Fragile — any typo silently passes
def set_log_level(level: str) -> None: ...
set_log_level("wraning")  # no error, silent misbehavior
```

<v-click>

```python
# Better — Literal constrains the values
from typing import Literal

def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None: ...
set_log_level("wraning")  # error: not assignable to Literal[...]
```

</v-click>

<v-click>

```python
# Best for complex cases — Enum with exhaustiveness
from enum import Enum

class LogLevel(Enum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"

def set_log_level(level: LogLevel) -> None: ...
```

</v-click>

---

# Exhaustiveness checking

Let the type checker verify you handled every case

```python
from enum import Enum
from typing import assert_never

class Shape(Enum):
    CIRCLE = "circle"
    SQUARE = "square"
    TRIANGLE = "triangle"

def area(shape: Shape, size: float) -> float:
    match shape:
        case Shape.CIRCLE:
            return 3.14159 * size ** 2
        case Shape.SQUARE:
            return size ** 2
        case _:
            assert_never(shape)  # error: Triangle not handled!
```

<v-click>

When you add a new enum member, **every `match` that forgot it becomes a type error**.

No more "I added a variant but forgot to update the handler" bugs.

</v-click>

---

# Generics — preserving type information

Don't lose type precision through transformations

```python
# Without generics — returns Any, caller loses type info
def first(items: list) -> Any:
    return items[0]

x = first([1, 2, 3])  # x: Any — useless
```

<v-click>

```python
# With generics — type flows through
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    return items[0]

x = first([1, 2, 3])    # x: int
y = first(["a", "b"])   # y: str
```

</v-click>

<v-click>

Generics keep the **type checker informed** across function boundaries. Especially important for container types, decorators, and utility functions.

</v-click>

---

# Putting it all together

Types and API design reinforce each other

| Principle | Benefit |
|---|---|
| Typed signatures | Self-documenting, machine-checkable contracts |
| dataclass / TypedDict | Structured data with typo detection |
| Protocols | Flexible abstractions, testability |
| NewType | Prevent value mix-ups at zero cost |
| Enums / Literals | Constrained inputs, exhaustiveness |
| Generics | Preserve type info across boundaries |

<v-click>

**Types are not just for catching bugs** — they shape how you think about your API.

When a type is hard to write, it's a signal that the design needs work.

</v-click>

---
layout: center
class: text-center
---

# Key takeaway

<div class="text-2xl mt-4 opacity-80">

Well-typed code is well-designed code.

**Invest in your API surface. Your users — and your future self — will thank you.**

</div>

---
layout: center
class: text-center
---

# References

<div class="text-left inline-block">

- Ben Hoyt — [*Designing Pythonic library APIs*](https://benhoyt.com/writings/python-api-design/) (2023)
- [PEP 544 — Protocols: Structural subtyping](https://peps.python.org/pep-0544/)
- [PEP 589 — TypedDict](https://peps.python.org/pep-0589/)
- [PEP 484 — Type Hints](https://peps.python.org/pep-0484/)
- [Python docs — `typing` module](https://docs.python.org/3/library/typing.html)
- [Pyright](https://github.com/microsoft/pyright) / [mypy](https://mypy-lang.org/) — static type checkers

</div>

<PoweredBySlidev mt-10 />
