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

# Types: Function signatures — in aiida-core

Good: precise generic signature — the return type *is* the documentation

```python
# aiida/common/lang.py
def type_check(
    what: T, of_type: Any, msg: str | None = None, allow_none: bool = False
) -> T:
    """Verify that object 'what' is of type 'of_type'."""
    if allow_none and what is None:
        return what
    if not isinstance(what, of_type):
        raise TypeError(msg or f"Got '{type(what)}', expecting '{of_type}'")
    return what  # returns T — same type as input
```

<v-click>

Bad: `CalcInfo` inherits from `DefaultFieldsAttributeDict` — 20+ fields, all implicitly `Optional`, no signature at all

```python
# aiida/common/datastructures.py
class CalcInfo(DefaultFieldsAttributeDict):
    _default_fields = (
        'job_environment', 'email', 'email_on_started', 'uuid',
        'prepend_text', 'append_text', 'num_machines',
        'num_mpiprocs_per_machine', 'codes_info', ...
    )
    # No constructor signature — anything goes
```

A user constructing a `CalcInfo` has no IDE autocomplete and no type checking.

</v-click>

---

# Types: `stubgen` — Python's header files

Generate `.pyi` stub files that show only the API surface

```bash
# Generate stubs for your package
stubgen -p mypackage -o stubs/
```

<v-click>

```python
# mypackage/shop.py (full implementation)
def order(chips: int, fish: int, timeout: float = 10.0) -> Receipt:
    validated = _check_availability(chips, fish)
    response = _call_api(validated, timeout)
    return Receipt.from_response(response)
```

```python
# stubs/mypackage/shop.pyi (generated — just the contract)
def order(chips: int, fish: int, timeout: float = 10.0) -> Receipt: ...
```

</v-click>

<v-click>

- `.pyi` files are **exactly** like C header files — types and signatures, no implementation
- `stubgen` (from mypy) or `pyright --createstub` auto-generates them
- Ship stubs with your package (`py.typed` marker) for downstream type checking
- Great for reviewing your public API at a glance — **if the stub looks messy, the API is messy**
- Also, messes up your LLM context much less than providing the full implementation, when discussing top-level design 😉

</v-click>

---

# Types: Types catch bugs

Adding types to existing code reveals hidden issues

```python
# Before types — "works fine" (or does it?)
def get_user_name(user_id):
    user = db.find(user_id)
    if user:
        return user.name
    return None  # <- caller probably doesn't handle this
```

<v-click>

```python
# After adding types — the bug is now visible
def get_user_name(user_id: int) -> str:  # mypy error  [return-value]
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

<!-- # Types: Types catch bugs — in aiida-core -->
<!---->
<!-- Real bugs found by adding types to 5 scheduler plugins ([#7156](https://github.com/aiidateam/aiida-core/pull/7156)) -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # 1. Wrong return type — annotation said dict, code returned str -->
<!-- def _get_detailed_job_info_command(self, job_id: str) -> dict[str, Any]:  # wrong! -->
<!--     return f"bjobs -l {escape_for_bash(job_id)}"  # actually returns str -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # 2. Wrong value type — str assigned where int expected -->
<!-- this_job.num_mpiprocs = str(element_child.data).strip()  # str, not int! -->
<!-- # Fix: -->
<!-- this_job.num_mpiprocs = int(str(element_child.data).strip()) -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # 3. Silent None return — callers didn't handle it -->
<!-- def _parse_time_string(self, string: str, fmt: str = "...") -> datetime: -->
<!--     if string == "-": -->
<!--         return None  # type error! None is not datetime -->
<!---->
<!---->
<!-- # Fix: raise ValueError instead, handle at call site -->
<!-- ``` -->
<!---->
<!-- </v-click> -->

# Types: Types catch bugs — in aiida-core

Bad: functions returning `None` instead of raising — invisible to callers

```python
# aiida/orm/logs.py
def create_entry_from_record(self, record: LogRecord) -> Optional['Log']:
    dbnode_id = record.__dict__.get('dbnode_id', None)

    if dbnode_id is None:
        return None  # caller must remember to check!
```

<v-click>

Bad: untyped utility functions — no contract at all

```python
# aiida/transports/transport.py
def validate_positive_number(ctx, param, value):  # no annotations!
    if not isinstance(value, (int, float)) or value < 0:
        from click import BadParameter
        raise BadParameter(f'{value} is not a valid positive number')
    return value
```

Without annotations, the type checker can't verify callers pass the right types — and the return type is invisible.

</v-click>

---

# Types: Types reveal design smells

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

---

# Types: Types reveal design smells

```python
@dataclass
class PaymentResult:
    tx_id: str
    status: str
    receipt_number: int


def process_payment(
    order: Order,
) -> PaymentResult: ...  # one clear return type, one clear responsibility
```

---

# Types: TypedDict and dataclasses over plain dicts

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

---

# Types: TypedDict and dataclasses over plain dicts

```python
# dataclass — full type safety + methods + immutability option
from dataclasses import dataclass


@dataclass(frozen=True)
class User:
    name: str
    age: int
    email: str
```

---

# Types: TypedDict and dataclasses — aiida-core lesson

What happens when you use a dict subclass instead

```python
# Custom dict subclass makes all fields secretly Optional
class JobInfo(DefaultFieldsAttributeDict):
    if TYPE_CHECKING:
        job_id: str  # looks required...
        num_machines: int  # looks required...
        # ...but __getitem__ returns None for missing keys!


# Every access needs:
if this_job.num_machines is not None:  # type: ignore[redundant-expr]
    ...  # "redundant" — but actually necessary due to dict magic
```

<v-click>

> "All this hacking around we need now for the types seems like a massive code smell
> that we should've never overwritten `__getitem__` like that in the first place..."
>
> The whole module should use pydantic or dataclasses in v3.

</v-click>

---

# Types: TypedDict and dataclasses — aiida-core (cont.)

What it looks like when done right:

```python
# aiida/engine/processes/exit_code.py
class ExitCode(NamedTuple):
    status: int = 0
    message: Optional[str] = None
    invalidates_cache: bool = False

    def format(self, **kwargs: str) -> 'ExitCode':
        message = self.message.format(**kwargs)
        return ExitCode(self.status, message, self.invalidates_cache)
```

<v-click>

```python
# aiida/tools/_dumping/utils.py
@dataclass(frozen=True)
class DumpTimes:
    current: datetime = field(default_factory=lambda: timezone.now())
    last: Optional[datetime] = None
```

`NamedTuple` for lightweight value objects, `@dataclass(frozen=True)` when you need immutability. Both give you type safety, autocomplete, and clear contracts — unlike dict subclasses.

</v-click>

---

# Types: Literals and Enums over bare strings

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

---

# Types: Literals and Enums over bare strings

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

---

# Types: Literals and Enums — in aiida-core

Good: Enums used extensively for constrained state

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

<v-click>

Impossible to pass `"uploding"` by accident — the type checker catches it. And adding a new state forces all `match` statements to be updated.

</v-click>

---

# Types: Exhaustiveness checking

Let the type checker verify you handled every case

```python
from enum import Enum
from typing import assert_never


class Shape(Enum):
    SQUARE = "square"
    TRIANGLE = "triangle"


def area(shape: Shape, size: float) -> float:
    match shape:
        case Shape.SQUARE:
            return size**2
        case _:
            assert_never(shape)  # error: Triangle not handled!
```

<v-click>

When you add a new enum member, **every `match` that forgot it becomes a type error**.

No more "I added a variant but forgot to update the handler" bugs.

</v-click>

---

# Types: NewType — preventing value mix-ups

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

</v-click>

---

# Types: NewType — in aiida-core

Bad: `NewType` is not used anywhere in aiida-core

Node PKs, UUIDs, job IDs, and group PKs are all plain `int` or `str`:

```python
# Nothing stops you from passing a group PK where a node PK is expected
def load_node(pk: int) -> Node: ...
def load_group(pk: int) -> Group: ...

node = load_node(group.pk)  # no error — but wrong entity!
```

<v-click>

With `NewType`, this class of bugs becomes impossible:

```python
NodePk = NewType("NodePk", int)
GroupPk = NewType("GroupPk", int)

def load_node(pk: NodePk) -> Node: ...
def load_group(pk: GroupPk) -> Group: ...

node = load_node(group.pk)  # error: GroupPk is not NodePk
```

Zero runtime cost, but prevents an entire category of "wrong ID" bugs.

</v-click>

---

# Types: `@final` and `Final` — locking things down

Prevent accidental overrides and mutations

```python
from typing import final, Final

MAX_RETRIES: Final = 3
MAX_RETRIES = 5  # error: cannot assign to Final variable
```

<v-click>

```python
from typing import final


class BaseProcessor:
    @final
    def validate(self, data: bytes) -> bool:
        """Critical validation — subclasses must not override."""
        return len(data) > 0 and data[0:4] == b"MAGIC"

class CustomProcessor(BaseProcessor):
    def validate(self, data: bytes) -> bool:  # error: cannot override final
        return True  # oops — security bypass prevented
```

</v-click>

<v-click>

- `Final` for constants — prevents reassignment
- `@final` on methods — prevents override in subclasses
- `@final` on classes — prevents subclassing entirely
- Communicates **intent**: "this is not meant to be extended"

</v-click>

---

# Types: `@final` and `Final` — in aiida-core

Good: `@final` prevents subclassing config singletons

```python
# aiida/manage/configuration/settings.py
from typing import final

@final
class AiiDAConfigDir:
    """Singleton for setting and getting the path to configuration directory."""
    ...

@final
class AiiDAConfigPathResolver:
    """For resolving configuration directory, daemon dir, daemon log dir."""
    ...
```

<v-click>

Bad: 9 constants in the same file lack `Final`

```python
# aiida/manage/configuration/settings.py
DEFAULT_UMASK = 0o0077                       # should be Final
DEFAULT_AIIDA_PATH_VARIABLE = 'AIIDA_PATH'   # should be Final
DEFAULT_CONFIG_DIR_NAME = '.aiida'            # should be Final
DEFAULT_CONFIG_FILE_NAME = 'config.json'      # should be Final
DEFAULT_CONFIG_INDENT_SIZE = 4                # should be Final
# ... 4 more
```

Nothing prevents `settings.DEFAULT_UMASK = 0o0000` — a subtle security issue that `Final` would catch.

</v-click>

---

# Types: `TypeAlias` — readable names for complex types

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
- Especially valuable for callback signatures and nested generics

</v-click>

---

# Types: `TypeAlias` — in aiida-core

Good: graph traversal module uses aliases for complex types

```python
# aiida/tools/graph/age_entities.py
from typing_extensions import TypeAlias

_NodeOrGroupCls: TypeAlias = 'type[orm.Node] | type[orm.Group]'
_ContainerTypes: TypeAlias = 'list[Any] | tuple[Any, ...] | set[Any]'
_EdgeType: TypeAlias = 'type[LinkQuadruple] | type[GroupNodeEdge]'
_BasketKeys: TypeAlias = Literal['nodes', 'groups', 'nodes_nodes', 'groups_nodes']
```

<v-click>

Bad: `CalcJobNode` repeats the same complex type 5 times

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

- Enables **dependency inversion** — depend on abstractions, not concretions
- Makes code **testable** — pass a fake that matches the protocol
- Matches Python's duck-typing philosophy, but **checked statically**
- Similar to Go interfaces or Rust traits

**In aiida-core:** `_EntityMapper(Protocol)` in the query builder — defines the expected ORM model properties structurally, without forcing inheritance.

</v-click>

---

# Types: Protocols — in aiida-core

Good: `ProcessFunctionType` Protocol defines the full decorated function contract

```python
# aiida/engine/processes/functions.py
class ProcessFunctionType(Protocol, Generic[P, R_co, N]):
    def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ...
    def run(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ...
    def run_get_node(self, ...) -> tuple[dict[str, Any] | None, N]: ...
    is_process_function: bool
    node_class: Type[N]
    process_class: Type[Process]
```

<v-click>

Good: `NodeIterator` Protocol for group iteration

```python
# aiida/orm/implementation/groups.py
class NodeIterator(Protocol):
    def __iter__(self) -> 'NodeIterator': ...
    def __next__(self) -> BackendNode: ...
    def __getitem__(self, value: int | slice) -> ...: ...
    def __len__(self) -> int: ...
```

Both define structural contracts — any object matching the shape satisfies the Protocol, no inheritance needed.

</v-click>

---

# Types: Generics — preserving type information

Don't lose type precision through transformations

```python
# Without generics — accepts any list, but loses the element type
def first(items: list[Any]) -> Any:
    return items[0]


val = first([1, 2, 3])  # val: Any
val.upper()  # no error — but crashes at runtime!
```

<v-click>

```python
# With generics — return type tracks the input element type
from typing import TypeVar

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]


val = first([1, 2, 3])  # val: int
val.upper()  # error: "int" has no attribute "upper"
```

</v-click>

<v-click>

Generics keep the **type checker informed** across function boundaries. Especially important for container types, decorators, and utility functions.

**In aiida-core:** `NodeCollection(Generic[NodeType])` — the collection type preserves which node subclass it contains, so `CalcJobNode.collection.get()` returns `CalcJobNode`, not `Node`.

</v-click>

---

# Types: Generics — in aiida-core

Good: full generic entity hierarchy preserves types across layers

```python
# aiida/orm/implementation/entities.py
EntityType = TypeVar('EntityType', bound='BackendEntity')

class BackendCollection(Generic[EntityType]):
    ENTITY_CLASS: ClassVar[Type[EntityType]]

    def create(self, **kwargs: Any) -> EntityType:  # precise return type
        return self.ENTITY_CLASS(backend=self._backend, **kwargs)
```

<v-click>

Bad: `QueryBuilder` loses all type info through the query pipeline

```python
# aiida/orm/querybuilder.py
def all(self, ...) -> list[list[Any]]:  # always Any
    ...

# Caller must cast:
nodes: list[Node] = qb.all(flat=True)  # type: ignore
```

The entities are well-typed at the ORM layer, but as soon as they go through `QueryBuilder`, everything becomes `Any`. A generic `QueryBuilder[T]` could preserve the projected types.

</v-click>

---

# Types: `@overload` — precise return types per call signature

Tell the type checker *exactly* what comes back

```python
from typing import overload


@overload
def parse(raw: str, as_json: Literal[True]) -> dict: ...
@overload
def parse(raw: str, as_json: Literal[False] = ...) -> str: ...


def parse(raw: str, as_json: bool = False) -> dict | str:
    return json.loads(raw) if as_json else raw
```

<v-click>

```python
result = parse('{"a": 1}', as_json=True)  # type: dict  (not dict | str!)
result = parse("hello", as_json=False)  # type: str
```

</v-click>

<v-click>

- Without `@overload`, the caller always sees `dict | str` and must narrow manually
- With `@overload`, each call site gets the **precise** return type
- Useful for functions whose return type depends on a flag or input type

**In aiida-core:** `CalculationFactory(name, load=True)` → `Type[CalcJob]` vs `CalculationFactory(name, load=False)` → `EntryPoint` — return type depends on the `load` flag.

</v-click>

---

# Types: `@overload` — in aiida-core

Good: `QueryBuilder.first()` uses `@overload` to distinguish flat vs nested returns

```python
# aiida/orm/querybuilder.py
@overload
def first(self, flat: Literal[False] = False) -> list[Any] | None: ...

@overload
def first(self, flat: Literal[True]) -> Any | None: ...

def first(self, flat: bool = False) -> list[Any] | Any | None:
    ...
```

<v-click>

Bad: `QueryBuilder.all()` still returns `list[list[Any]]` — generic type info is lost through the query pipeline. The element types depend on what was projected, but the type system can't express that.

```python
def all(self, ..., flat: bool = False) -> list[list[Any]] | list[Any]:
    ...  # caller always gets Any — must cast manually
```

The `@overload` on `flat` helps, but the inner `Any` means the caller never knows the actual column types.

</v-click>

---

<!-- # Types: `@overload` — aiida-core: mypy vs pyright -->
<!---->
<!-- mypy and pyright don't always agree -->
<!---->
<!-- ```python -->
<!-- @overload -->
<!-- def get_jobs(self, ..., as_dict: Literal[False] = False) -> list[JobInfo]: ... -->
<!-- @overload -->
<!-- def get_jobs(self, ..., as_dict: Literal[True] = True) -> dict[str, JobInfo]: ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- **mypy**: passes. **pyright**: error — overlapping overloads with incompatible return types. -->
<!---->
<!-- Calling `get_jobs()` with no args matches **both** overloads. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # Fix: make as_dict keyword-only to avoid the overlap -->
<!-- @overload -->
<!-- def get_jobs(self, ..., *, as_dict: Literal[True]) -> dict[str, JobInfo]: ... -->
<!-- #                         ^ keyword-only, no default — no ambiguity -->
<!-- ``` -->
<!---->
<!-- But that's an API change — can't do it in a typing-only PR. -->
<!---->
<!-- **Lesson**: adding types to existing code surfaces API design decisions you never explicitly made. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

# Types: `@singledispatch` — dispatch based on input types

Runtime polymorphism, checked by types

```python
from functools import singledispatch

@singledispatch
def serialize(value: object) -> str:
    raise TypeError(f"Cannot serialize {type(value)}")

@serialize.register
def _(value: int) -> str:
    return str(value)

@serialize.register
def _(value: list) -> str:
    return "[" + ", ".join(serialize(v) for v in value) + "]"

@serialize.register
def _(value: datetime) -> str:
    return value.isoformat()
```

<v-click>

- Replaces chains of `isinstance` checks with **extensible dispatch**
- New types can register handlers **without modifying the original function**
- `@singledispatchmethod` for class methods

**vs `@overload`**: `@overload` is purely static — one function body, multiple type signatures. `@singledispatch` is runtime — separate function bodies, dispatched by the first argument's type.

</v-click>

---

# Types: `@singledispatch` — in aiida-core

Good: `@singledispatch` used to convert backend entities to ORM objects

```python
# aiida/orm/convert.py
@singledispatch
def get_orm_entity(backend_entity):
    raise TypeError(f"No conversion for {type(backend_entity)}")

@get_orm_entity.register(BackendComputer)
def _(backend_entity):
    return Computer.from_backend_entity(backend_entity)
```

<v-click>

Bad: isinstance chains that should use singledispatch

```python
# aiida/tools/dbimporters/plugins/oqmd.py
if not isinstance(values, str) and not isinstance(values, int):
    raise ValueError(
        f"incorrect value for keyword '{alias}' "
        "-- only strings and integers are accepted"
    )
```

A `@singledispatch` on the value type would be cleaner and extensible — new types could register handlers without modifying the function.

</v-click>

---

<!-- # Types: `TypeGuard` and `TypeIs` — custom type narrowing -->
<!---->
<!-- Teach the type checker about your validation logic -->
<!---->
<!-- ```python -->
<!-- from typing import TypeIs -->
<!---->
<!---->
<!-- def is_valid_user(value: object) -> TypeIs[User]: -->
<!--     return isinstance(value, dict) and "name" in value and "email" in value -->
<!---->
<!---->
<!-- def handle(data: object) -> None: -->
<!--     if is_valid_user(data): -->
<!--         print(data.name)  # type checker knows: data is User -->
<!--     else: -->
<!--         print("invalid")  # data is still object -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - `TypeGuard[T]` (3.10+) — narrows only in the `if` branch -->
<!-- - `TypeIs[T]` (3.13+) — narrows in **both** branches (more precise) -->
<!-- - Bridges the gap between runtime validation and static types -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

# Types: Builder pattern with `Self` — typed fluent APIs

Method chaining with correct types

```python
from typing import Self
from dataclasses import dataclass, field


@dataclass
class Query:
    _table: str = ""
    _columns: list[str] = field(default_factory=list)
    _conditions: list[str] = field(default_factory=list)
    _limit: int | None = None

    def select(self, *columns: str) -> Self:
        self._columns = list(columns)
        return self

    def where(self, condition: str) -> Self:
        self._conditions.append(condition)
        return self

    def limit(self, n: int) -> Self:
        self._limit = n
        return self
```

<v-click>

```python
query = Query(_table="users").select("name", "email").where("age > 18").limit(10)
```

`Self` ensures subclasses return **their own type**, not the parent.

</v-click>

---

# Types: `ParamSpec` — typed decorators that preserve signatures

Don't let decorators erase your function types

```python
from typing import ParamSpec, TypeVar, Callable
import functools

P = ParamSpec("P")
R = TypeVar("R")


def retry(times: int) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for _ in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    pass
            return func(*args, **kwargs)

        return wrapper

    return decorator
```

<v-click>

```python
@retry(3)
def fetch(url: str, timeout: float = 10.0) -> bytes: ...


fetch(42)  # error: int is not str — signature preserved!
```

</v-click>

---

# Types: `ParamSpec` — in aiida-core

Good: `@calcfunction` preserves signatures via `ParamSpec` + Protocol

```python
# aiida/engine/processes/functions.py
P = ParamSpec('P')
R_co = TypeVar('R_co', covariant=True)

class ProcessFunctionType(Protocol, Generic[P, R_co, N]):
    def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ...
    def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[...]: ...
    process_class: Type[Process]
```

<v-click>

Bad: the internal decorator erases types with `*args, **kwargs`

```python
# Same file, the actual decorator implementation
@functools.wraps(function)
def decorated_function(*args, **kwargs):  # signature erased!
    result, _ = run_get_node(*args, **kwargs)
    return result

decorated_function.run = decorated_function  # type: ignore[attr-defined]
decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined]
# ... 6 more type: ignore[attr-defined] lines
```

The `ProcessFunctionType` Protocol patches over this at the type level — but the runtime decorator still erases signatures.

</v-click>

---

# Types: Variance — why `list[Dog]` is not `list[Animal]`

A common source of type errors and confusion

```python
class Animal: ...


class Dog(Animal): ...


class Cat(Animal): ...


def add_cat(animals: list[Animal]) -> None:
    animals.append(Cat())


dogs: list[Dog] = [Dog(), Dog()]
add_cat(dogs)  # mypy error: list[Dog] is not list[Animal]
# If allowed: dogs would now contain a Cat — type violation
```

<v-click>

- `list` is **invariant** — `list[Dog]` is not a subtype of `list[Animal]`
- `Sequence` (read-only) is **covariant** — `Sequence[Dog]` *is* `Sequence[Animal]`
- Mutable containers must be invariant to prevent inserting wrong types

</v-click>

<v-click>

```python
from collections.abc import Sequence


def count_legs(animals: Sequence[Animal]) -> int:  # accepts Sequence[Dog]
    return sum(4 for _ in animals)
```

This is **Postel's Law** (the Robustness Principle, from TCP/RFC 761):

> "Be liberal in what you accept, be conservative in what you send."

Accept broad input types (`Sequence`, `Mapping`), return narrow/specific types (`list`, `dict`).

</v-click>

---

# Types: Variance — aiida-core: LSP violations

The process hierarchy overrides `define()` with **narrower** parameter types:

```python
# Process (base)
@classmethod
def define(cls, spec: ProcessSpec) -> None:  # type: ignore[override]
    super().define(spec)


# CalcJob (subclass) — wants a more specific spec type
@classmethod
def define(cls, spec: CalcJobProcessSpec) -> None:  # type: ignore[override]
    super().define(spec)
    spec.inputs.validator = validate_calc_job  # type: ignore[assignment]
```

<v-click>

Every level violates the [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) — a `CalcJob` can't be used everywhere a `Process` is expected if it demands a more specific spec.

</v-click>

<v-click>

**This is not a typing limitation — it's a design signal.** Consider generic classes:
`class Process(Generic[S]): def define(cls, spec: S)` — no override needed.

</v-click>

---

# Types: `type: ignore` — a taxonomy from aiida-core

What 366 ignores across a codebase tell you

| Category | Count | What it reveals |
|---|---|---|
| `attr-defined` | 646 | Heavy dynamic attribute access (ORM, mixins) |
| `arg-type` | 561 | API polymorphism that types can't express |
| `assignment` | 487 | Variables change type mid-function |
| `override` | 368 | Inheritance hierarchy violates LSP |
| `union-attr` | 229 | Missing type narrowing |
| `return-value` | 198 | Return type doesn't match annotation |

<v-click>

**Real examples from aiida-core:**

```python
# Dynamic ORM attribute access — fundamentally untypeable
node.base.repository.put_object(...)  # type: ignore[attr-defined]

# SQLAlchemy relationships assigned at runtime
DbNode.dbcomputer = sa_orm.relationship(...)  # type: ignore[attr-defined]

# dict interface violation — documented as a TODO
# TODO: We're in violation of the `dict` interface here
def __dir__(self) -> list[Any]:  # type: ignore[override]
```

</v-click>

<v-click>

Each `type: ignore` is a **decision**: fix the design, or document the debt.

</v-click>

