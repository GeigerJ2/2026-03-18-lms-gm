---
layout: default
---

# Types: Function signatures are your API contract

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
def order(chips: int, fish: int, timeout: float = 10.0) -> Receipt: ...
```

</v-click>

<v-click>

- Users read signatures **before** reading docstrings
- IDEs show signatures on hover — they're the first thing people see
- A well-typed signature answers: *what do I pass in, what do I get back?*
- **If your signature needs a paragraph to explain, redesign the API**

</v-click>

---

# Types: `stubgen` — Python's "header files"

Generate `.pyi` stub files that show only the API surface

- `stubgen` (from mypy) or `pyright --createstub` auto-generates them

<v-click>

```bash
# Generate stubs for your package
stubgen -p mypackage -o stubs/
```

</v-click>

<v-click>

- `.pyi` files are like C header files — types and signatures, no implementation
- Great for reviewing your public API at a glance
- Friendlier to LLM context than the full implementation 😉

</v-click>

---

# Types: `stubgen` — aiida-core example

`Waiting` — the state that manages a `CalcJob` while it runs on the remote computer (~280 lines, excerpt):

```python
# stubs/aiida/engine/processes/calcjobs/tasks.pyi
class Waiting(plumpy.process_states.Waiting):
    def __init__(
        self, process: CalcJob, done_callback: Callable[..., Any] | None,
        msg: str | None = None, data: Any | None = None,
    ) -> None: ...
    @property
    def monitors(self) -> CalcJobMonitors | None: ...
    @property
    def process(self) -> CalcJob: ...
    def upload(self) -> Waiting: ...
    def submit(self) -> Waiting: ...
    def update(self) -> Waiting: ...
    def stash(self, monitor_result: CalcJobMonitorResult | None = None) -> Waiting: ...
    def retrieve(self, monitor_result: CalcJobMonitorResult | None = None) -> Waiting: ...
    def parse(
        self, retrieved_temporary_folder: str, exit_code: ExitCode | None = None,
    ) -> plumpy.process_states.Running: ...
    def interrupt(self, reason: Any) -> plumpy.futures.Future | None: ...
```

<v-click>

The stub reveals the lifecycle at a glance: upload → submit → update → retrieve → parse.

</v-click>

---

# Types: Types catch bugs and guide the API

Adding types to existing code reveals hidden issues

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Before — "works fine" (or does it?)
def get_user_name(user_id):
    user = db.find(user_id)
    if user:
        return user.name
    return None  # caller might not handle this
```

</div>
<div>

<v-click>

```python
# After — the issue now becomes obvious
def get_user_name(user_id: int) -> str:
    # mypy error: [return-value]
    user = db.find(user_id)
    if user:
        return user.name
    return None  # None is not str
```

</v-click>

</div>
</div>

<v-click>

The type checker **forces you to decide**: return `str | None`? Or raise on missing users? Either way, the caller now knows what to expect.

</v-click>

<v-click>

**In aiida-core:** adding types to the scheduler module revealed wrong return types, `str` assigned where `int` was expected, and silent `None` returns ([#7156](https://github.com/aiidateam/aiida-core/pull/7156) by @danielhollas).

<div class="absolute bottom-4 left-12 text-xs opacity-50">#7156: <i>Add typing to scheduler plugins</i></div>

</v-click>

<v-click>

**Annotate immediately**, fix flaws, avoid releasing unpolished API to avoid being stuck because of backwards compatibility.

</v-click>

---

# Types: Types reveal code smells

When the return type is too broad, your code needs refactoring

<div class="grid grid-cols-2 gap-4">
<div>

```python
def process_payment(
    order: Order
) -> str | int | dict | None:
    if order.method == "card":
        return {"tx_id": "abc", "status": "ok"}
    elif order.method == "cash":
        return 42  # receipt number
    elif order.method == "credit":
        return "pending"
    return None
```

</div>
<div>

<v-click>

```python
@dataclass
class PaymentResult:
    tx_id: str
    status: str
    receipt_number: int


# one clear return type,
# one clear responsibility
def process_payment(
    order: Order,
) -> PaymentResult: ...
```

</v-click>

</div>
</div>

<v-click>

If you struggle to write the return type, **the API is missing an abstraction**.

</v-click>

---

# Types: Types reveal code smells — aiida-core

Breaking the dict contract: `None` instead of `KeyError` (surfaced in [#7136](https://github.com/aiidateam/aiida-core/pull/7136) by @danielhollas)

<div class="grid grid-cols-2 gap-4">
<div>

```python
# aiida/common/extendeddicts.py — the root cause
class DefaultFieldsAttributeDict(AttributeDict):
    def __getitem__(self, key: str) -> Any | None:
        try:
            return super().__getitem__(key)
        except KeyError:
            if key in self._default_fields:
                return None  # breaks the dict contract
                             # — KeyError expected!
            raise
```

</div>
<div>

<v-click>

```python
# aiida/schedulers/datastructures.py — victim
class JobInfo(DefaultFieldsAttributeDict):
    if TYPE_CHECKING:
        num_machines: int        # int (not Optional)...
        allocated_machines: list[MachineInfo]  # same
```

```python
# aiida/schedulers/plugins/slurm.py
# field may be None at runtime, but type checker
# sees int — contradiction:
if (
    this_job.num_machines is not None
    and this_job.allocated_machines is not None
):  # type: ignore[redundant-expr]
    ...
```

</v-click>

</div>
</div>

<div class="absolute bottom-4 left-12 text-xs opacity-50">#7136: <i>Strict typing for aiida.schedulers module</i></div>

---

# Types: TypedDict and dataclasses over plain dicts

Structure your data, don't just bag it

```python
# The "stringly typed" approach — no safety, no autocomplete
user = {"name": "Alice", "age": 30, "emial": "alice@example.com"}
#                                    ^^^^^ typo goes unnoticed
```

<div class="grid grid-cols-2 gap-4">

<v-click>

<div>

```python
# TypedDict — lightweight, catches key typos
from typing import TypedDict


class User(TypedDict):
    name: str
    age: int
    email: str


user: User = {"name": "Alice", "age": 30, "emial": "..."}
#                                          ^^^^^^ error!
```

</div>

</v-click>

<v-click>

<div>

```python
# dataclass — type safety + methods + immutability
# (also: NamedTuple for lightweight immutable records)
from dataclasses import dataclass


@dataclass(frozen=True)
class User:
    name: str
    age: int
    email: str
```

</div>

</v-click>

</div>

---

# Types: Literals and Enums over bare strings

Constrain your inputs

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Fragile — any typo silently passes
def set_log_level(level: str) -> None: ...

set_log_level("wraning")  # silent misbehavior
```

<v-click>

```python
# Better — Literal constrains the values
from typing import Literal

LogLevel = Literal[
    "debug", "info", "warning", "error",
]

def set_log_level(level: LogLevel) -> None: ...

set_log_level("wraning")  # type error!

```

</v-click>

</div>
<div>

<v-click>

```python
# Best — Enum with exhaustiveness
from enum import Enum
from typing import assert_never

class LogLevel(Enum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"

def set_log_level(level: LogLevel) -> None:
    match level:
        case LogLevel.DEBUG: ...
        case _:
            assert_never(level)  # error!
```

Adding a new member → every `match` that forgot it becomes a type error.

</v-click>

</div>
</div>

---

# Types: Literals and Enums — in aiida-core

<div class="grid grid-cols-2 gap-4">
<div>

```python
# aiida/common/links.py
class LinkType(Enum):
    CREATE = 'create'
    RETURN = 'return'
    INPUT_CALC = 'input_calc'
    INPUT_WORK = 'input_work'
    CALL_CALC = 'call_calc'
    CALL_WORK = 'call_work'

# aiida/common/datastructures.py
class CalcJobState(Enum):
    UPLOADING = 'uploading'
    SUBMITTING = 'submitting'
    WITHSCHEDULER = 'withscheduler'
    RETRIEVING = 'retrieving'
    PARSING = 'parsing'
```

</div>
<div>

```python
# aiida/common/datastructures.py
class StashMode(Enum):
    COPY = 'copy'
    COMPRESS_TAR = 'tar'
    COMPRESS_TARBZ2 = 'tar.bz2'
    COMPRESS_TARGZ = 'tar.gz'
    COMPRESS_TARXZ = 'tar.xz'

# aiida/orm/entities.py
class EntityTypes(Enum):
    AUTHINFO = 'authinfo'
    COMMENT = 'comment'
    COMPUTER = 'computer'
    GROUP = 'group'
    LOG = 'log'
    NODE = 'node'
    USER = 'user'
    LINK = 'link'
```

</div>
</div>

<v-click>

Impossible to pass `"uploding"` by accident — the type checker catches it.

Adding a new enum member forces all `match` statements to be updated.

</v-click>

---

# Types: `TypeAlias` — name your complex types

Tame your type annotations

```python
# Without aliases — good luck reading this
def handle(
    callback: Callable[[Request, dict[str, list[str]]], Awaitable[Response | None]],
    middlewares: list[Callable[[Request], Awaitable[Request]]],
) -> None: ...
```

<v-click>

```python
# With aliases — intent is clear
from typing import TypeAlias

Headers: TypeAlias = dict[str, list[str]]
Handler: TypeAlias = Callable[[Request, Headers], Awaitable[Response | None]]
Middleware: TypeAlias = Callable[[Request], Awaitable[Request]]


def handle(callback: Handler, middlewares: list[Middleware]) -> None: ...
```

</v-click>

<v-click>

- Aliases are **documentation** — they name a concept, not just a shape
- Reusable across your codebase — change the definition in one place

</v-click>

---

# Types: `TypeAlias` — in aiida-core

graph traversal module uses aliases for complex types ✅

```python
# aiida/tools/graph/age_entities.py
from typing_extensions import TypeAlias

_NodeOrGroupCls: TypeAlias = 'type[orm.Node] | type[orm.Group]'
_ContainerTypes: TypeAlias = 'list[Any] | tuple[Any, ...] | set[Any]'
_EdgeType: TypeAlias = 'type[LinkQuadruple] | type[GroupNodeEdge]'
_BasketKeys: TypeAlias = Literal['nodes', 'groups', 'nodes_nodes', 'groups_nodes']
```

<v-click>

`CalcJobNode` repeats the same complex type 5 times ❌

```python
# aiida/orm/nodes/process/calculation/calcjob.py
def _validate_retrieval_directive(
    directives: Sequence[Union[str, Tuple[str, str, str]]]) -> None: ...

def set_retrieve_list(
    self, retrieve_list: Sequence[Union[str, Tuple[str, str, str]]]) -> None: ...

def get_retrieve_list(
    self) -> Optional[Sequence[Union[str, Tuple[str, str, str]]]]: ...

# Should be:  RetrievalDirective: TypeAlias = Sequence[str | tuple[str, str, str]]
```

</v-click>

---

# Types: Generics — preserving type information

Don't lose type precision through transformations

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Without generics — loses the element type
def first(items: list[Any]) -> Any:
    return items[0]


val = first([1, 2, 3])  # val: Any
val.upper()  # no error — crashes at runtime!
```

</div>
<div>

<v-click>

```python
# With generics — return type tracks input
from typing import TypeVar

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]


val = first([1, 2, 3])  # val: int
val.upper()  # error: int has no .upper()
```

</v-click>

</div>
</div>

<v-click>

Generics keep the **type checker informed** across function boundaries.

</v-click>

---

# Types: Protocols and abstract interfaces

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

- **ABCs** = nominal subtyping (must inherit). **Protocols** = structural subtyping (just match the shape).
- Works with third-party classes you don't control — `io.BytesIO` satisfies `Readable` without knowing about it
- Makes code **testable** — pass a fake that matches the protocol

</v-click>

---

# Types: `@overload` and `@singledispatch`

<div class="grid grid-cols-2 gap-4">
<div>

**`@overload`** — narrow return types statically

```python
# aiida/orm/querybuilder.py
@overload
def first(
    self, flat: Literal[False] = ...,
) -> list[Any] | None: ...
@overload
def first(
    self, flat: Literal[True],
) -> Any | None: ...

def first(
    self, flat: bool = False,
) -> list[Any] | Any | None: ...
```

No `@overload`, always `list[Any] | Any | None`.

</div>
<div>

<v-click>

**`@singledispatch`** — runtime dispatch by type

```python
# aiida/orm/convert.py
@singledispatch
def get_orm_entity(backend_entity):
    raise TypeError(...)

@get_orm_entity.register(BackendComputer)
def _(backend_entity):
    return Computer.from_backend_entity(backend_entity)
```

Replaces `isinstance` chains — new types register handlers without modifying the function.

</v-click>

</div>
</div>

<v-click>

**`@overload`** is purely static — one function body, multiple type signatures. <br>
**`@singledispatch`** is runtime — separate function bodies, dispatched by the first argument's type.

</v-click>

---

# Types: Variance

Why `list[Dog]` is not `list[Animal]`

<div class="grid grid-cols-2 gap-4">
<div>

```python
class Animal: ...
class Dog(Animal): ...
class Cat(Animal): ...

# ❌ list is mutable → invariant
def add_cat(animals: list[Animal]) -> None:
    animals.append(Cat())

dogs: list[Dog] = [Dog(), Dog()]
add_cat(dogs)  # type error!
# if allowed: dogs would contain a Cat
```

</div>
<div>

<v-click>

```python
# ✅ Sequence is read-only → covariant
from collections.abc import Sequence

def count_legs(animals: Sequence[Animal]) -> int:
    return sum(4 for _ in animals)

count_legs(dogs)  # works!
# can't insert a Cat — no append, no __setitem__
```

</v-click>

</div>
</div>

<v-click>

- **`list` is mutable** → **invariant**: if `list[Dog]` were accepted as `list[Animal]`, a function could `append(Cat())` — your `list[Dog]` would secretly contain a `Cat`. The type checker would have lied. So neither is a subtype of the other.
- **`Sequence` is read-only** → **covariant**: no `append`, no `__setitem__` — nothing wrong can be inserted. Every `Dog` *is* an `Animal`, so every read is safe. `Sequence[Dog]` *is* a `Sequence[Animal]`.

</v-click>

---

# Types: Postel's Law and Liskov Substitution Principle

> "Be liberal in what you accept, be conservative in what you send." — Postel's Law (RFC 761)

Accept broad input types (`Sequence`, `Mapping`), return narrow/specific types (`list`, `dict`).

<v-click>

**Liskov Substitution Principle** (LSP): a subclass must accept everything its parent accepts. If it narrows parameter types, code that passes a base-type argument to the subclass will break at runtime.

</v-click>

<v-click>

Subclasses **extend** the parent — they should support the same feature set, not narrow it.

</v-click>

<div class="grid grid-cols-2 gap-4">
<div>

<v-click>

```python
# ❌ narrows Food → CatFood
class Animal:
    def feed(self, food: Food) -> None: ...

class Cat(Animal):
    def feed(self, food: CatFood) -> None: ...

feed_all([Cat()], DogFood())  # crash!
```

</v-click>

</div>
<div>

<v-click>

```python
# ✅ generics — no override needed
F = TypeVar("F", bound=Food)

class Animal(Generic[F]):
    def feed(self, food: F) -> None: ...

class Cat(Animal[CatFood]):
    def feed(self, food: CatFood) -> None: ...
```

</v-click>

</div>
</div>

<v-click>

**In aiida-core:** `CalcJob.define(spec: CalcJobProcessSpec)` narrows from `Process.define(spec: ProcessSpec)` — same pattern, suppressed with `type: ignore[override]`.

</v-click>

---

# Types: `type: ignore` in aiida-core `v2.8.0`

(Still) 367 `type: ignore` across the codebase (343 in `src/`, 24 in tests)

| **Category** | **Count** | **What it reveals** |
|---|---|---|
| `attr-defined` | 94 | Heavy dynamic attribute access (ORM, mixins) |
| `arg-type` | 43 | API polymorphism that types can't express |
| `union-attr` | 31 | Missing type narrowing |
| `return-value` | 27 | Return type doesn't match annotation |
| `assignment` | 27 | Variables change type mid-function |
| `override` | 18 | Inheritance hierarchy violates LSP |

Each `type: ignore` is a **decision**: fix the design, or document the debt.
