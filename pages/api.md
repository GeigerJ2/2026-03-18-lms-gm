# API: `urllib` vs `requests`

Example from `urllib`, part of Python `stdlib`<br>(taken from: https://docs.python.org/3/howto/urllib2.html#id5)

```python
import urllib

# create a password manager
manager = urllib.request.HTTPPasswordMgrWithDefaultRealm()

# Add the username and password.
manager.add_password(None, "https://httpbin.org/", "usr", "pwd")

# Create authentication handler
handler = urllib.request.HTTPBasicAuthHandler(manager)

# create "opener" (OpenerDirector instance)
opener = urllib.request.build_opener(handler)

# use the opener to fetch a URL and print the response status
response = opener.open("https://httpbin.org/basic-auth/usr/pwd")

print(response.status)
```

On first glance, doesn't look too bad, no?

---

# API: `urllib` vs `requests`

The code reads fine with comments — but could you write it from scratch?

- **5 concepts** to learn: password manager, handler, opener, request, response
- **2 absurd names** to remember: `HTTPPasswordMgrWithDefaultRealm`, `HTTPBasicAuthHandler`
- A mystery `None` as first argument to `add_password`
<!-- - `build_opener`? `opener.open`? — Java-style factory patterns in Python -->

<v-click>

The comments make the flow clear **after the fact** — but when writing from scratch, you'd need to read the docs for every single step. A good API shouldn't require a tutorial to make a basic authenticated request.

</v-click>

---

# API: `urllib` vs `requests`

Well, how about that instead?

```python
import requests

response = requests.get("https://httpbin.org/basic-auth/usr/pwd", auth=("usr", "pwd"))
print(response.status_code)
```

<v-click>

Same task. Wildly different experience. **API design matters.**

In the following slides, we'll explore what makes an API good, drawing on principles from Ben Hoyt's [*Designing Pythonic library APIs*](https://benhoyt.com/writings/python-api-design/).

We'll use a fictional `fishnchips` library as a running example, alongside real examples from `aiida-core`.

</v-click>

---

# API: Flat is better than nested

Expose a clean API; file structure is an implementation detail

```python
# Bad — user must know internal file structure
from django.core.files.images import ImageFile  # 4 levels deep!
from fishnchips.api.v2.orders import create_order
```

<v-click>

```python
# Good — re-export everything in __init__.py
# fishnchips/__init__.py
from .api import order
from .shop import Shop

# User code — flat and simple
import fishnchips

fishnchips.order(chips=1, fish=2)
```

</v-click>

<v-click>

**In aiida-core:** `aiida.orm` re-exports 90+ classes (`Node`, `Int`, `CalcJobNode`, ...) from deeply nested modules. By convention, anything below a 2nd-level import (e.g. `aiida.orm.nodes.data.int`) is considered **private API** (but the line gets blurry with users, workflow devs, plugin devs).

</v-click>

---

# API: Flat imports — in aiida-core

Good: clean re-exports in `aiida.orm` and `aiida.engine`

```python
# aiida/orm/__init__.py — 90+ classes re-exported
from .authinfos import *
from .comments import *
from .nodes import *
from .querybuilder import *

# User code:
from aiida.orm import Int, CalcJobNode, QueryBuilder  # all flat
```

<v-click>

Bad: circular imports force workarounds

```python
# aiida/orm/nodes/process/process.py
if TYPE_CHECKING:
    from aiida.engine.processes import ExitCode, Process  # circular!

# aiida/orm/utils/managers.py
# This import is here to avoid circular imports
from aiida.orm import Node
```

The flat re-exports work well for users, but internally the circular dependencies between `orm` and `engine` require `TYPE_CHECKING` guards and delayed imports.

</v-click>

---

# API: `import lib` then `lib.Thing()`

The module name provides context — don't make users lose it

```python
# Bad — where does order() come from? Is it defined here? Imported?
from fishnchips import order


def eat_takeaways():
    meal = order(chips=1)  # ordering what? from where?
```

<v-click>

```python
# Good — immediately clear what library you're using
import fishnchips


def eat_takeaways():
    meal = fishnchips.order(chips=1)  # obviously fish & chips
```

</v-click>

<v-click>

This means: don't name your function `order_fishnchips` — the module name already provides that context. `fishnchips.order()` reads naturally, `fishnchips.order_fishnchips()` is redundant.

Same reason `requests.get()` works — you're not meant to `from requests import get`.

**In aiida-core:** `from aiida import orm` then `orm.Int(42)`, `orm.load_node(pk)` — always clear it's an AiiDA operation.

</v-click>

---

# API: Naming — short as possible, clear as necessary

The module name is already context — don't repeat it

```python
# Bad — redundant, buries the important word in the middle
fishnchips.order_fishnchips_food()
fishnchips.place_fish_and_chips_order_for_meal()
```

<v-click>

```python
# Good — short, the module name already provides context
fishnchips.order()
```

</v-click>

<v-click>

**Conventions from PEP 8:**

- Functions → **verbs**: `get`, `post`, `order`, `connect`
- Classes → **nouns**: `Session`, `Response`, `Shop`, `Router`
- Capitalisation distinguishes them: `shop` (function) vs `Shop` (class)
- `_private` is sufficient — `__extra_private` (name mangling) is almost never needed

</v-click>

---

# API: Naming — in aiida-core

Good: consistent `load_X` pattern across all loaders

```python
# aiida/orm/utils/loaders.py
def load_code(...) -> 'Code': ...
def load_computer(...) -> 'Computer': ...
def load_group(...) -> 'Group': ...
def load_node(...) -> 'Node': ...
```

<v-click>

Bad: redundant naming in kpoints utilities

```python
# aiida/tools/data/array/kpoints/main.py
def get_kpoints_path(structure, method='seekpath', **kwargs): ...
#   dispatches to:
#   _legacy_get_kpoints_path()
#   _seekpath_get_kpoints_path()

def get_explicit_kpoints_path(structure, method='seekpath', **kwargs): ...
#   dispatches to:
#   _legacy_get_explicit_kpoints_path()
#   _seekpath_get_explicit_kpoints_path()
```

The word "kpoints_path" appears in every function name — the module already provides that context.

</v-click>

---

# API: Avoid global configuration

Use good defaults and let the user override them

```python
# Bad — mutable global that affects ALL callers
DEFAULT_TIMEOUT = 10


def order(chips=None, fish=None, timeout=None):
    if timeout is None:
        timeout = DEFAULT_TIMEOUT  # someone else set this to 1...
```

<v-click>

```python
# Good — keyword argument with a sensible default
def order(chips=None, fish=None, timeout=10): ...


# User who wants a different default? functools.partial
import functools

order_fast = functools.partial(fishnchips.order, timeout=1)
```

</v-click>

<v-click>

Each caller gets exactly what they asked for. No spooky action at a distance.

**In aiida-core:** global `CONFIG` exists but is scoped via `profile_context()` — a context manager that loads a profile temporarily and restores the previous one on exit.

</v-click>

---

# API: Avoid global configuration — in aiida-core

Bad: global `CONFIG` with `reset_config()` that warns of side effects

```python
# aiida/manage/configuration/__init__.py
CONFIG: Optional['Config'] = None

def get_config(create=False) -> 'Config':
    global CONFIG  # noqa: PLW0603
    if not CONFIG:
        CONFIG = load_config(create=create)
    return CONFIG

def reset_config():
    """.. warning:: This is experimental functionality and should for now
    be used only internally. If the reset is unclean weird unknown
    side-effects may occur that end up corrupting or destroying data."""
    global CONFIG  # noqa: PLW0603
    CONFIG = None
```

<v-click>

Good: the same module wraps this danger in a context manager

```python
@contextmanager
def profile_context(profile=None, allow_switch=False) -> 'Profile':
    current_profile = manager.get_profile()
    yield manager.load_profile(profile, allow_switch)
    if current_profile is None:
        manager.unload_profile()
    else:
        manager.load_profile(current_profile, allow_switch=True)
```

The global state exists — but its **API surface** guides users toward safe, scoped access.

</v-click>

---

# API: Avoid global state — use a class

What if `fishnchips` tracked orders in a module-level list?

```python
# fishnchips.py — Bad: module-level state, only one order queue
_orders = []


def order(chips=0, fish=0):
    _orders.append({"chips": chips, "fish": fish})


# app.py — works fine... until you need TWO shops
fishnchips.order(chips=2, fish=1)
# Later: "I want separate queues for dine-in and takeaway"
# ...but there's only one _orders list. No way to separate them.
```

---

# API: Avoid global state — use a class

```python
# Good — state lives in a class instance
class Shop:
    def __init__(self, name: str):
        self.name = name
        self._orders: list[dict] = []

    def order(self, chips: int = 0, fish: int = 0) -> None:
        self._orders.append({"chips": chips, "fish": fish})


# Users can have as many shops as they want
dine_in = fishnchips.Shop("Dine-in")
takeaway = fishnchips.Shop("Takeaway")
```

<v-click>

When your library needs state between calls, a class is the right tool. See `requests.Session` — holds auth, headers, and reuses TCP connections.

**In aiida-core:** caching state uses module-level `_CONTEXT_CACHE`, but access is controlled via `enable_caching()` / `disable_caching()` context managers — never raw mutation.

</v-click>

---

# API: Avoid global state — in aiida-core

Bad: multiple global singletons scattered across the codebase

```python
# aiida/manage/manager.py
MANAGER: Optional['Manager'] = None

def get_manager() -> 'Manager':
    global MANAGER  # noqa: PLW0603
    if MANAGER is None:
        MANAGER = Manager()
    return MANAGER

# aiida/common/progress_reporter.py
PROGRESS_REPORTER = ProgressReporterNull

def set_progress_reporter(reporter=None) -> None:
    global PROGRESS_REPORTER  # noqa: PLW0603
    PROGRESS_REPORTER = reporter or ProgressReporterNull
```

<v-click>

Four `global` singletons: `CONFIG`, `MANAGER`, `PROGRESS_REPORTER`, `OBJECT_LOADER` — each is module-level mutable state. The `# noqa: PLW0603` suppresses the linter warning about `global` statements, but the design concern remains.

The saving grace: `profile_context()` and `enable_caching()` wrap the most dangerous state in context managers.

</v-click>

---

# API: Errors — raise exceptions, not error codes

Build a custom exception hierarchy

```python
# Bad — returning None or error codes, caller has to check manually
def order(chips=None, fish=None):
    response = _call_api(chips, fish)
    if response.status_code == 200:  # Response is OK, happy path
        return response.json()
    return None  # caller must remember to check for None!
```

---

# API: Errors — raise exceptions, not error codes

```python
# Good — custom exception hierarchy with useful attributes
class Error(Exception): ...


class OrderError(Error):
    def __init__(self, code, chef_name, message):
        self.code = code
        self.chef_name = chef_name
        self.message = message
```

```python
# User code — easy to catch everything, or be specific
try:
    fishnchips.order(chips=1, fish=2)
except fishnchips.OrderError as e:
    print(f"{e.chef_name} said: {e.message}")
except fishnchips.Error:
    print("Unexpected error")
```

**In aiida-core:** `AiidaException` → `NotExistent` → `NotExistentAttributeError(AttributeError, NotExistent)` — deep hierarchy with multiple inheritance to also satisfy standard Python exception types.

---

# API: Errors — in aiida-core

Good: semantic exception hierarchy with useful subclasses

```python
# aiida/common/exceptions.py
class MissingEntryPointError(EntryPointError):
    """Raised when the requested entry point is not registered."""

class MultipleEntryPointError(EntryPointError):
    """Raised when the entry point cannot uniquely be resolved."""

class LockedProfileError(AiidaException):
    """Raised if attempting to access a locked profile."""
```

<v-click>

Bad: bare `Exception` raise instead of a proper AiiDA exception

```python
# aiida/orm/logs.py
if not isinstance(entity, nodes.Node):
    raise Exception('Only node logs are stored')  # bare Exception!
```

Bad: overly broad `except Exception: pass`

```python
# aiida/transports/plugins/async_backend.py
try:
    result = subprocess.run(['ssh', '-V'], capture_output=True, ...)
    match = re.search(r'OpenSSH_(\d+)', output)
    ...
except Exception:  # silently swallows ALL errors
    pass
```

</v-click>

---

# API: Backwards compatibility — keyword args are your friend

Add features without breaking existing callers

```python
# v1.0 — simple
def order(chips: int = 0, fish: int = 0) -> Receipt: ...
```

<v-click>

```python
# v1.1 — added fish_type, fully backwards compatible
from enum import Enum


class FishType(Enum):
    BATTERED = "battered"
    CRUMBED = "crumbed"


def order(
    chips: int = 0, fish: int = 0, fish_type: FishType = FishType.BATTERED
) -> Receipt: ...
```

</v-click>

---

# API: Backwards compatibility — keyword args are your friend

```python
# v1.2 — added new foods, still backwards compatible
def order(
    chips: int = 0,
    fish: int = 0,
    fish_type: FishType = FishType.BATTERED,
    onion_rings: int = 0,
    hotdogs: int = 0,
) -> Receipt: ...
```

Every old call still works. New keyword args with defaults = **no breaking changes**.

---

# API: Backwards compatibility — in aiida-core

Good: `Process.__init__` — all parameters have defaults

```python
# aiida/engine/processes/process.py
def __init__(
    self,
    inputs: Optional[Dict[str, Any]] = None,
    logger: Optional[logging.Logger] = None,
    runner: Optional['Runner'] = None,
    parent_pid: Optional[int] = None,
    enable_persistence: bool = True,
) -> None: ...
```

<v-click>

Bad: internal helper with 7 positional parameters

```python
# aiida/engine/daemon/execmanager.py
async def _copy_remote_files(
    logger, node, computer, transport,
    remote_copy_list, remote_symlink_list, workdir
):  # 7 positional args — adding one breaks all callers
```

Should use keyword-only arguments: `async def _copy_remote_files(*, logger, node, ...)`. Even for internal functions, keyword args prevent ordering mistakes.

</v-click>

---

# API: Minimize mutable state and side effects

Prefer returning new values over mutating inputs

```python
# Surprising — mutates the input
def add_defaults(config: dict) -> dict:
    config["timeout"] = config.get("timeout", 30)
    config["retries"] = config.get("retries", 3)
    return config  # returns the SAME object, mutated


original = {"timeout": 10}
result = add_defaults(original)
# original is now {"timeout": 10, "retries": 3} — surprise!
```

<v-click>

```python
# Better — returns a new value, input unchanged
from dataclasses import dataclass


@dataclass(frozen=True)
class Config:
    timeout: float = 30.0
    retries: int = 3


def with_defaults(overrides: Config | None = None) -> Config:
    return overrides or Config()
```

</v-click>

<v-click>

- `frozen=True` dataclasses are immutable by default
- `tuple` over `list` for fixed collections — signals "this won't change"
- Immutable data is easier to reason about, cache, and share across threads

**In aiida-core:** `Node.base.attributes.all` returns a `copy.deepcopy()` for stored nodes — once persisted, attributes are effectively immutable.

</v-click>

---

# API: Minimize mutable state — in aiida-core

Good: `Node.__copy__` raises — copying is explicitly forbidden

```python
# aiida/orm/nodes/node.py
def __copy__(self) -> NoReturn:
    """Copying a Node is not supported in general, but only for Data."""
    raise exceptions.InvalidOperation('copying a base Node is not supported')

def __deepcopy__(self, memo: Any) -> NoReturn:
    raise exceptions.InvalidOperation('deep copying a base Node is not supported')
```

<v-click>

Good: `Data.__deepcopy__` creates an unstored clone — no shared mutable state

```python
# aiida/orm/nodes/data/data.py
def __deepcopy__(self, memo):
    return self.clone()

def clone(self):
    backend_clone = self.backend_entity.clone()
    clone = from_backend_entity(self.__class__, backend_clone)
    clone.base.attributes.reset(copy.deepcopy(self.base.attributes.all))
    return clone  # new unstored node, fully independent
```

The base `Node` forbids mutation; `Data` allows cloning with full isolation.

</v-click>

---

# API: The pit of success

Design so the easiest path is the correct one

```python
# requests verifies SSL by default — you have to opt OUT of safety
requests.get("https://api.example.com")  # SSL verified
requests.get("https://api.example.com", verify=False)  # explicit opt-out
```

<v-click>

**Anti-pattern**: making users opt IN to correctness

```python
# Hypothetical bad API — unsafe by default
db.query("SELECT ...", sanitize=True)  # easy to forget sanitize=True
```

</v-click>

<v-click>

**Pit of success principles**:

- **Safe defaults** — SSL on, escaping on, validation on
- **Unsafe options require explicit action** — `verify=False`, `trust_env=False`
- **Make wrong code look wrong** — if misuse looks identical to correct use, the API has failed
- Types help enforce this: `SafeQuery` vs `RawQuery` as distinct types

</v-click>

---

# API: Progressive disclosure

<!-- TODO: Add the ssh configuration as a wrong example here -->

Simple things simple, complex things possible

```python
# Level 1 — one line, sensible defaults
response = httpx.get("https://api.example.com/users")
```

<v-click>

```python
# Level 2 — add options as needed
response = httpx.get(
    "https://api.example.com/users",
    params={"page": 2},
    timeout=5.0,
)
```

</v-click>

<v-click>

```python
# Level 3 — full control with a client
client = httpx.Client(
    base_url="https://api.example.com",
    auth=("user", "pass"),
    timeout=httpx.Timeout(connect=5.0, read=30.0),
    transport=httpx.HTTPTransport(retries=3),
)
response = client.get("/users", params={"page": 2})
```

</v-click>

<v-click>

**Don't front-load complexity.** Let users discover features when they need them. Keyword arguments with defaults are the primary tool for this.

**In aiida-core:** Level 1: `@calcfunction` decorator — write a plain Python function. Level 2: subclass `CalcJob` — full control over inputs, outputs, scheduling, parsing.

</v-click>

---

# API: Context managers as API

Resource management the Pythonic way

```python
# Bad API — easy to forget cleanup
conn = db.connect()
try:
    result = conn.execute("SELECT ...")
finally:
    conn.close()  # what if you forget this?
```

<v-click>

```python
# Good API — cleanup is automatic
with db.connect() as conn:
    result = conn.execute("SELECT ...")
# conn.close() called automatically — even on exceptions
```

</v-click>

<v-click>

```python
# Typed context manager
from contextlib import contextmanager
from typing import Iterator


@contextmanager
def connect(url: str) -> Iterator[Connection]:
    conn = Connection(url)
    try:
        yield conn
    finally:
        conn.close()
```

If your API involves **acquire/release**, **open/close**, or **setup/teardown**, make it a context manager. Users get correctness for free.

**In aiida-core:** `profile_context()`, `enable_caching()`, `disable_caching()` — all context managers that scope global state and restore it on exit.

</v-click>

---

# API: Context managers — in aiida-core

Good: `Transport` supports nested context managers with reference counting

```python
# aiida/transports/transport.py
def __enter__(self) -> Self:
    if self._track_enter():  # reference counting for nested use
        self.open()
    return self

def __exit__(self, type_, value, traceback):
    if self._track_exit():
        self.close()

# Also supports async:
async def __aenter__(self): ...
async def __aexit__(self, ...): ...
```

<v-click>

This is excellent design — users can nest `with transport:` blocks safely, and both sync and async paths are supported. The transport connection is never leaked, even on exceptions.

</v-click>

---

# API: Async API mirrors

Designing sync + async without duplicating everything

```python
import httpx

# Sync client
resp = httpx.get("https://example.com")

# Async client — same API shape
async with httpx.AsyncClient() as client:
    resp = await client.get("https://example.com")
```

<v-click>

**Design considerations**:

- Mirror the sync API as closely as possible — same method names, same parameters
- Use separate classes (`Client` / `AsyncClient`), not a flag
- Types help: `-> Response` vs `-> Awaitable[Response]` are distinct contracts
- Avoid `is_async` boolean parameters — they make types imprecise

</v-click>

<v-click>

```python
# Bad — one class, unclear types
client.get(url, async_mode=True)  # returns... what?


# Good — distinct types make the contract clear
def get(self, url: str) -> Response: ...  # Client
async def get(self, url: str) -> Response: ...  # AsyncClient
```

</v-click>

---

# API: Don't overuse Python's expressiveness

Just because you *can* doesn't mean you *should*

```python
# Bad — operator overloading on a non-numeric type
class Order:
    def __add__(self, other):  # order1 + order2 merges orders??
        return merge_orders(self, other)

    def __mul__(self, n):  # order * 3 triples quantities??
        return scale_order(self, n)
```

<v-click>

```python
# Good — explicit methods that say what they do
class Order:
    def merge_with(self, other: "Order") -> "Order": ...
    def scale(self, factor: int) -> "Order": ...
```

</v-click>

<v-click>

**Rules of thumb:**

- Only override `+`, `*` etc. for number/vector types
- Only override `[]` for collections
- Property getters should be cheap — no I/O, no exceptions
- **If the type signature is too hard to write, the API might be a bad idea**

</v-click>

---

# API: Deprecation strategy

Guide users from old API to new

```python
import warnings
from typing import overload


@overload
def connect(url: str) -> Connection: ...
@overload
def connect(host: str, port: int) -> Connection: ...


def connect(url_or_host: str, port: int | None = None) -> Connection:
    if port is not None:
        warnings.warn(
            "connect(host, port) is deprecated, use connect(url) instead. "
            "Will be removed in v3.0.",
            DeprecationWarning,
            stacklevel=2,
        )
        url_or_host = f"https://{url_or_host}:{port}"
    return Connection(url_or_host)
```

<v-click>

- `@overload` keeps both old and new signatures typed during the transition
- `DeprecationWarning` is shown by test runners but hidden in production
- `stacklevel=2` points the warning at the **caller**, not the library
- Set a clear removal timeline (e.g., "removed in v3.0")

**In aiida-core:** custom `AiidaDeprecationWarning` (does *not* inherit `DeprecationWarning` to avoid default filtering) + `warn_deprecation(message, version=3)` that appends "this will be removed in v3".

</v-click>

---

# API: Deprecation strategy — in aiida-core

Good: centralized utility with configurable filtering

```python
# aiida/common/warnings.py
def warn_deprecation(message: str, version: int, stacklevel: int = 2) -> None:
    from_config = get_config_option('warnings.showdeprecations')
    from_environment = os.environ.get(f'AIIDA_WARN_v{version}')

    if from_config or from_environment:
        message = f'{message} (this will be removed in v{version})'
        warnings.warn(message, AiidaDeprecationWarning, stacklevel=stacklevel)
```

<v-click>

Bad: informal deprecation without concrete migration path

```python
# aiida/transports/plugins/ssh.py
def chdir(self, path):
    """
    PLEASE DON'T USE `chdir()` IN NEW DEVELOPMENTS, INSTEAD
    DIRECTLY PASS ABSOLUTE PATHS TO INTERFACE.
    `chdir()` is DEPRECATED and will be removed in the next
    major version.
    """
```

"PLEASE DON'T USE" in a docstring is not a deprecation strategy. Which interface? What's the concrete replacement? A proper deprecation tells you *exactly* what to write instead.

</v-click>
