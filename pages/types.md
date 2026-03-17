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

# Types: `stubgen` — Python's header files

Generate `.pyi` stub files that show only the API surface

```bash
# Generate stubs for your package
stubgen -p mypackage -o stubs/
```

<v-click>

- `.pyi` files are like C header files — types and signatures, no implementation
- `stubgen` (from mypy) or `pyright --createstub` auto-generates them
- Great for reviewing your public API at a glance — **if the stub looks messy, the API is messy**
- Friendlier to LLM context than the full implementation 😉

</v-click>

---

# Types: `stubgen` — aiida-core example

`stubs/aiida/engine/processes/exit_code.pyi` — generated, nothing hand-written:

```python
from aiida.common.extendeddicts import AttributeDict
from typing import NamedTuple

class ExitCode(NamedTuple):
    status: int = ...
    message: str | None = ...
    invalidates_cache: bool = ...
    def format(self, **kwargs: str) -> ExitCode: ...

class ExitCodesNamespace(AttributeDict):
    def __call__(self, identifier: int | str) -> ExitCode: ...
```

<v-click>

The full implementation is ~80 lines. The stub tells you everything a caller needs in 10.

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

# Types: Types catch bugs — in aiida-core

Real bugs found by adding types to 5 scheduler plugins ([#7156](https://github.com/aiidateam/aiida-core/pull/7156) by `danielhollas`)

<v-click>

```python
# 1. Wrong return type — annotation said dict, code returned str
def _get_detailed_job_info_command(self, job_id: str) -> dict[str, Any]:  # wrong!
    return f"bjobs -l {escape_for_bash(job_id)}"  # actually returns str
```

</v-click>

<v-click>

```python
# 2. Wrong value type — str assigned where int expected
this_job.num_mpiprocs = str(element_child.data).strip()  # str, not int!
# Fix:
this_job.num_mpiprocs = int(str(element_child.data).strip())
```

</v-click>

<v-click>

```python
# 3. Silent None return — callers didn't handle it
def _parse_time_string(self, string: str, fmt: str = "...") -> datetime:
    if string == "-":
        return None  # type error! None is not datetime


# Fix: raise ValueError instead, handle at call site
```

</v-click>

---

# Types: Types reveal design smells

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

If you struggle to write the return type, **the function is doing too many things**.

</v-click>

---

# Types: Types reveal design smells — aiida-core

Breaking the dict contract: `None` instead of `KeyError` ([#7136](https://github.com/aiidateam/aiida-core/pull/7136))

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
    this_job.allocated_machines is not None
    and this_job.num_machines is not None
):  # type: ignore[redundant-expr]
    ...
```

</v-click>

</div>
</div>

---

# Types: TypedDict and dataclasses over plain dicts

Structure your data, don't just bag it

```python
# The "stringly typed" approach — no safety, no autocomplete
user = {"name": "Alice", "age": 30, "emial": "alice@example.com"}
#                                    ^^^^^ typo goes unnoticed
```

<v-click>

<div class="grid grid-cols-2 gap-4">
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
<div>

```python
# dataclass: type safety + methods + immutability option
from dataclasses import dataclass


@dataclass(frozen=True)
class User:
    name: str
    age: int
    email: str
```

</div>
</div>

</v-click>

<v-click>

`NamedTuple` sits between the two: immutable like a frozen dataclass, but also tuple-compatible (unpackable, positionally accessible). Used in aiida-core for `ExitCode`.

</v-click>

---

<!-- # Types: TypedDict and dataclasses — aiida-core -->
<!---->
<!-- What it looks like when done right: -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/processes/exit_code.py -->
<!-- class ExitCode(NamedTuple): -->
<!--     status: int = 0 -->
<!--     message: Optional[str] = None -->
<!--     invalidates_cache: bool = False -->
<!---->
<!--     def format(self, **kwargs: str) -> 'ExitCode': -->
<!--         message = self.message.format(**kwargs) -->
<!--         return ExitCode(self.status, message, self.invalidates_cache) -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # aiida/tools/_dumping/utils.py -->
<!-- @dataclass(frozen=True) -->
<!-- class DumpTimes: -->
<!--     current: datetime = field(default_factory=lambda: timezone.now()) -->
<!--     last: Optional[datetime] = None -->
<!-- ``` -->
<!---->
<!-- `NamedTuple` for lightweight value objects, `@dataclass(frozen=True)` when you need immutability. Both give you type safety, autocomplete, and clear contracts — unlike dict subclasses. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

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

# Types: Literals and Enums — in aiida-core

Enums used extensively for constrained state ✅

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

# Types: Literals and Enums — in aiida-core (cont.)

Same problem, two approaches ([#7069](https://github.com/aiidateam/aiida-core/pull/7069) vs [#7116](https://github.com/aiidateam/aiida-core/pull/7116)): control what happens on unhandled process failure

<div class="grid grid-cols-2 gap-4">
<div>

**Two boolean flags** (PR #7069)

```python
spec.input('pause_for_unknown_errors',
           valid_type=orm.Bool,
           default=lambda: orm.Bool(False))
spec.input('restart_once_for_unknown_errors',
           valid_type=orm.Bool,
           default=lambda: orm.Bool(True))
```

</div>
<div>

**One string input** (PR #7116)

```python
spec.input('on_unhandled_failure',
           valid_type=orm.Str,
           required=False,
           validator=validate_on_unhandled_failure)
# options: 'abort', 'pause',
#   'restart_once', 'restart_and_pause'
```

</div>
</div>

---

# Types: Literals and Enums — in aiida-core (cont.)

Two booleans → deeply nested dispatch

```python
if node.is_failed and not last_report:
    if self.inputs.restart_once_for_unknown_errors.value:       # restart enabled?
        if self.ctx.unhandled_failure:                           #   second failure?
            if self.inputs.pause_for_unknown_errors.value:      #     pause enabled?
                self.ctx.paused_by_handler = True
                self.ctx.unhandled_failure = False
                self.pause()
                return None
            else:                                               #     pause disabled
                return self.exit_codes.ERROR_SECOND_CONSECUTIVE_UNHANDLED_FAILURE
        self.ctx.unhandled_failure = True                       #   first failure
    else:                                                       # restart disabled
        if self.inputs.pause_for_unknown_errors.value:          #   pause enabled?
            self.ctx.paused_by_handler = True
            self.pause()
        else:                                                   #   pause disabled
            return self.exit_codes.ERROR_UNHANDLED_FAILURE
```

<v-click>

4 combinations of 2 booleans → **3 levels of nesting**, hard to follow

</v-click>

---

# Types: Literals and Enums — in aiida-core (cont.)

One string → flat dispatch

```python
if node.is_failed and not last_report:
    action = self.inputs.get('on_unhandled_failure', None)
    action = action.value if action is not None else 'abort'

    if action == 'abort':
        return self.exit_codes.ERROR_UNHANDLED_FAILURE
    elif action == 'pause':
        self.pause(f"Paused for inspection, see: 'verdi process report {self.node.pk}'")
        return None
    elif action == 'restart_once':
        if self.ctx.unhandled_failure:
            return self.exit_codes.ERROR_UNHANDLED_FAILURE
        self.ctx.unhandled_failure = True
        return None
    elif action == 'restart_and_pause':
        if self.ctx.unhandled_failure:
            self.ctx.unhandled_failure = False
            self.pause(f"Paused for inspection, see: 'verdi process report {self.node.pk}'")
            return None
        self.ctx.unhandled_failure = True
        return None
```

<v-click>

Same behavior, **1 level of nesting** — each case is self-contained and readable.

</v-click>

<!-- --- -->
<!---->
<!-- # Types: NewType — preventing value mix-ups -->
<!---->
<!-- Semantically different values deserve different types -->
<!---->
<!-- ```python -->
<!-- from typing import NewType -->
<!---->
<!-- UserId = NewType("UserId", int) -->
<!-- OrderId = NewType("OrderId", int) -->
<!---->
<!---->
<!-- def get_order(order_id: OrderId) -> Order: ... -->
<!---->
<!---->
<!-- user_id = UserId(42) -->
<!-- get_order(user_id)  # error: UserId is not compatible with OrderId -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - **Zero runtime cost** — `NewType` is erased at runtime -->
<!-- - Prevents an entire class of "wrong ID" bugs -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # Types: `@final` and `Final` — locking things down -->
<!---->
<!-- Prevent accidental overrides and mutations -->
<!---->
<!-- ```python -->
<!-- from typing import final, Final -->
<!---->
<!-- MAX_RETRIES: Final = 3 -->
<!-- MAX_RETRIES = 5  # error: cannot assign to Final variable -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- from typing import final -->
<!---->
<!---->
<!-- class BaseProcessor: -->
<!--     @final -->
<!--     def validate(self, data: bytes) -> bool: -->
<!--         """Critical validation — subclasses must not override.""" -->
<!--         return len(data) > 0 and data[0:4] == b"MAGIC" -->
<!---->
<!-- class CustomProcessor(BaseProcessor): -->
<!--     def validate(self, data: bytes) -> bool:  # error: cannot override final -->
<!--         return True  # oops — security bypass prevented -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - `Final` for constants — prevents reassignment -->
<!-- - `@final` on methods — prevents override in subclasses -->
<!-- - `@final` on classes — prevents subclassing entirely -->
<!-- - Communicates **intent**: "this is not meant to be extended" -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # Types: `@final` and `Final` — in aiida-core -->
<!---->
<!-- `@final` prevents subclassing config singletons ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/manage/configuration/settings.py -->
<!-- from typing import final -->
<!---->
<!-- @final -->
<!-- class AiiDAConfigDir: -->
<!--     """Singleton for setting and getting the path to configuration directory.""" -->
<!--     ... -->
<!---->
<!-- @final -->
<!-- class AiiDAConfigPathResolver: -->
<!--     """For resolving configuration directory, daemon dir, daemon log dir.""" -->
<!--     ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- 9 constants in the same file lack `Final` ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/manage/configuration/settings.py -->
<!-- DEFAULT_UMASK = 0o0077                       # should be Final -->
<!-- DEFAULT_AIIDA_PATH_VARIABLE = 'AIIDA_PATH'   # should be Final -->
<!-- DEFAULT_CONFIG_DIR_NAME = '.aiida'            # should be Final -->
<!-- DEFAULT_CONFIG_FILE_NAME = 'config.json'      # should be Final -->
<!-- DEFAULT_CONFIG_INDENT_SIZE = 4                # should be Final -->
<!-- # ... 4 more -->
<!-- ``` -->
<!---->
<!-- Nothing prevents `settings.DEFAULT_UMASK = 0o0000` — a subtle security issue that `Final` would catch. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

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

</v-click>

---

# Types: Protocols — in aiida-core

`ProcessFunctionType` Protocol defines the full decorated function contract ✅

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

`NodeIterator` Protocol for group iteration <span class="opacity-50">✅</span>

```python
# aiida/orm/implementation/groups.py
class NodeIterator(Protocol):
    def __iter__(self) -> 'NodeIterator': ...
    def __next__(self) -> BackendNode: ...
    def __getitem__(self, value: int | slice) -> ...: ...
    def __len__(self) -> int: ...
```

Both define structural contracts — any object matching the shape satisfies the Protocol, **no inheritance needed**.

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

```python
# With generics — return type tracks input
from typing import TypeVar

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]


val = first([1, 2, 3])  # val: int
val.upper()  # error: int has no .upper()
```

</div>
</div>

<v-click>

Generics keep the **type checker informed** across function boundaries.

</v-click>

---

<!-- # Types: Generics — in aiida-core -->
<!---->
<!-- full generic entity hierarchy preserves types across layers ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/orm/implementation/entities.py -->
<!-- EntityType = TypeVar('EntityType', bound='BackendEntity') -->
<!---->
<!-- class BackendCollection(Generic[EntityType]): -->
<!--     ENTITY_CLASS: ClassVar[Type[EntityType]] -->
<!---->
<!--     def create(self, **kwargs: Any) -> EntityType:  # precise return type -->
<!--         return self.ENTITY_CLASS(backend=self._backend, **kwargs) -->
<!-- ``` -->
<!---->
<!-- --- -->

# Types: `@overload` — narrow return types

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

</v-click>

---

# Types: `@overload` — in aiida-core

`QueryBuilder.first()` and `serialize()` both use `@overload` ✅

```python
# aiida/orm/querybuilder.py — flat flag changes return shape
@overload
def first(self, flat: Literal[False] = False) -> list[Any] | None: ...
@overload
def first(self, flat: Literal[True]) -> Any | None: ...
def first(self, flat: bool = False) -> list[Any] | Any | None: ...
```

<v-click>

```python
# aiida/orm/utils/serialize.py — encoding flag changes return type
@overload
def serialize(data: Any, encoding: None = None) -> str: ...
@overload
def serialize(data: Any, encoding: str) -> bytes: ...
def serialize(data: Any, encoding: str | None = None) -> str | bytes: ...
```

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

# Types: `@singledispatch` — type-based dispatch

Runtime polymorphism, checked by types

<div class="grid grid-cols-2 gap-4">
<div>

```python
from functools import singledispatch

@singledispatch
def serialize(value: object) -> str:
    raise TypeError(f"Cannot serialize {type(value)}")
```

</div>
<div>

```python
@serialize.register
def _(value: int) -> str:
    return str(value)

@serialize.register
def _(value: list) -> str:
    items = ", ".join(serialize(v) for v in value)
    return f"[{items}]"

@serialize.register
def _(value: datetime) -> str:
    return value.isoformat()
```

</div>
</div>

<v-click>

- Replaces chains of `isinstance` checks with **extensible dispatch**
- New types can register handlers **without modifying the original function**
- `@singledispatchmethod` for class methods

**vs `@overload`**: `@overload` is purely static — one function body, multiple type signatures. `@singledispatch` is runtime — separate function bodies, dispatched by the first argument's type.

</v-click>

---

# Types: `@singledispatch` — in aiida-core

`@singledispatch` used to convert backend entities to ORM objects ✅

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

isinstance chains that should use singledispatch ❌

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

# Types: `ParamSpec` — preserve decorator signatures

Don't let decorators erase your function types

```python
from typing import ParamSpec, TypeVar, Callable

P = ParamSpec("P")
R = TypeVar("R")


def log(func: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```

<v-click>

```python
@log
def fetch(url: str, timeout: float = 10.0) -> bytes: ...

fetch(42)        # error: int is not str — signature preserved!
fetch("https://example.com", timeout="slow")  # error: str is not float
```

</v-click>

---

<!-- # Types: `ParamSpec` — in aiida-core -->
<!---->
<!-- `@calcfunction` preserves signatures via `ParamSpec` + Protocol ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/processes/functions.py -->
<!-- P = ParamSpec('P') -->
<!-- R_co = TypeVar('R_co', covariant=True) -->
<!---->
<!-- class ProcessFunctionType(Protocol, Generic[P, R_co, N]): -->
<!--     def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ... -->
<!--     def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[...]: ... -->
<!--     process_class: Type[Process] -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- the internal decorator erases types with `*args, **kwargs` ❌ -->
<!---->
<!-- ```python -->
<!-- # Same file, the actual decorator implementation -->
<!-- @functools.wraps(function) -->
<!-- def decorated_function(*args, **kwargs):  # signature erased! -->
<!--     result, _ = run_get_node(*args, **kwargs) -->
<!--     return result -->
<!---->
<!-- decorated_function.run = decorated_function  # type: ignore[attr-defined] -->
<!-- decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined] -->
<!-- # ... 6 more type: ignore[attr-defined] lines -->
<!-- ``` -->
<!---->
<!-- The `ProcessFunctionType` Protocol patches over this at the type level — but the runtime decorator still erases signatures. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->

# Types: Variance

Why `list[Dog]` is not `list[Animal]` — and when that's the right call

<div class="grid grid-cols-2 gap-4">
<div>

```python
class Animal: ...
class Dog(Animal): ...
class Cat(Animal): ...


def add_cat(animals: list[Animal]) -> None:
    animals.append(Cat())


dogs: list[Dog] = [Dog(), Dog()]
add_cat(dogs)  # error: list[Dog] is not list[Animal]
# if allowed: dogs would contain a Cat!
```

</div>
<div>

<v-click>

- `list` is **invariant**: because it's mutable, the type checker must be strict — accepting `list[Dog]` where `list[Animal]` is expected would allow inserting a `Cat`
- `Sequence` is **covariant**: it's read-only (no `append`, no `__setitem__`), so no wrong type can ever be inserted — subtyping is safe
- The rule: **more capability → stricter variance**

</v-click>

</div>
</div>

---

# Types: Variance — prefer broad inputs, narrow outputs

<v-click>

```python
from collections.abc import Sequence

# accepts Sequence[Dog] ✅ — read-only, so covariance is safe
def count_legs(animals: Sequence[Animal]) -> int:
    return sum(4 for _ in animals)
```

</v-click>

<v-click>

This is **Postel's Law** (Robustness Principle, RFC 761):

> "Be liberal in what you accept, be conservative in what you send."

Accept broad input types (`Sequence`, `Mapping`), return narrow/specific types (`list`, `dict`). This maps directly to variance: covariant (read-only) inputs, invariant (mutable) outputs.

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

(Still) 343 `type: ignore` across the codebase 

| Category | Count | What it reveals |
|---|---|---|
| `attr-defined` | 89 | Heavy dynamic attribute access (ORM, mixins) |
| `arg-type` | 42 | API polymorphism that types can't express |
| `union-attr` | 30 | Missing type narrowing |
| `return-value` | 27 | Return type doesn't match annotation |
| `assignment` | 24 | Variables change type mid-function |
| `override` | 18 | Inheritance hierarchy violates LSP |

<!-- --- -->

<!-- **Real examples from aiida-core:** -->
<!---->
<!-- ```python -->
<!-- # Dynamic ORM attribute access — fundamentally untypeable -->
<!-- node.base.repository.put_object(...)  # type: ignore[attr-defined] -->
<!---->
<!-- # SQLAlchemy relationships assigned at runtime -->
<!-- DbNode.dbcomputer = sa_orm.relationship(...)  # type: ignore[attr-defined] -->
<!---->
<!-- # dict interface violation — documented as a TODO -->
<!-- # TODO: We're in violation of the `dict` interface here -->
<!-- def __dir__(self) -> list[Any]:  # type: ignore[override] -->
<!-- ``` -->

Each `type: ignore` is a **decision**: fix the design, or document the debt.
