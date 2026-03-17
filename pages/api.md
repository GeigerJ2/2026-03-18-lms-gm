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

By convention, anything not re-exported (e.g. `aiida.orm.nodes.caching.NodeCaching`) is **private API** — but the line gets blurry with users, workflow devs, plugin devs. Notably, `LinkPair` and `LinkTriple` are re-exported, but `LinkQuadruple` from the same `utils.links` module is not.

<v-click>

Bad: circular imports force workarounds

```python
# The cycle: orm → engine → orm
# 1. aiida.engine.processes.process imports at runtime:
from aiida import orm                              # pulls in all of orm
from aiida.orm.nodes.process.calculation.calcjob import CalcJobNode

# 2. aiida.orm.nodes.process.process needs engine types, but can't
#    import at runtime (orm is still loading!) — so:
if TYPE_CHECKING:
    from aiida.engine.processes import ExitCode, Process  # type-only

# 3. aiida.orm.utils.managers works around it with a delayed import:
class NodeLinksManager:
    def __init__(self, ...):
        from aiida.orm import Node  # "to avoid circular imports"
```

</v-click>

<v-click>

**How could this be resolved?** Extract shared types (`ExitCode`, `ProcessSpec`, link types) into a standalone `aiida.common.process_types` module that both `orm` and `engine` import from — breaking the cycle at the root rather than patching around it.

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

---

# API: Naming — in aiida-core

Bad: redundant naming in `KpointsData`

```python
# aiida/orm/nodes/data/array/kpoints.py
class KpointsData(ArrayData):
    def set_kpoints(self, kpoints, ...): ...
    def get_kpoints(self, ...): ...
    def set_kpoints_mesh(self, mesh, offset=None): ...
    def get_kpoints_mesh(self, print_list=False): ...
    def set_kpoints_mesh_from_density(self, distance, ...): ...
```

```python
# User code — "kpoints" everywhere
kpoints = KpointsData()
kpoints.set_kpoints_mesh([4, 4, 4])
mesh = kpoints.get_kpoints_mesh()
```

<v-click>

The class already says "Kpoints" — the methods shouldn't repeat it:

```python
# Better — let the class name provide context
kpoints = KpointsData()
kpoints.set_mesh([4, 4, 4])
mesh = kpoints.get_mesh()
```

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

</v-click>

---

# API: Avoid global state — in aiida-core

Bad: four `global` singletons scattered across the codebase

```python
# aiida/manage/configuration/__init__.py
CONFIG: Optional['Config'] = None
def get_config(create=False) -> 'Config':
    global CONFIG  # noqa: PLW0603
    ...

# aiida/manage/manager.py
MANAGER: Optional['Manager'] = None
def get_manager() -> 'Manager':
    global MANAGER  # noqa: PLW0603
    ...

# Also: PROGRESS_REPORTER, OBJECT_LOADER — same pattern
```

<v-click>

The saving grace: `profile_context()` and `enable_caching()` wrap the most dangerous state in context managers. But the globals remain — not thread-safe, not async-safe.

**Fix:** replace `global` with `contextvars.ContextVar` (PEP 567):

```python
from contextvars import ContextVar

_config: ContextVar[Config | None] = ContextVar('_config', default=None)

def get_config() -> Config:
    config = _config.get()          # no global keyword needed
    ...

@contextmanager
def profile_context(profile=None):
    token = _config.set(...)        # scoped, thread/async-safe
    try: yield
    finally: _config.reset(token)   # clean restore via token
```

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

<!-- # API: Backwards compatibility — keyword args are your friend -->
<!---->
<!-- How positional args break callers -->
<!---->
<!-- ```python -->
<!-- # v1.0 -->
<!-- def order(chips: int, fish: int) -> Receipt: ... -->
<!---->
<!-- order(2, 1)  # 2 chips, 1 fish -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # v1.1 — "fish should come first, it's fish & chips after all" -->
<!-- def order(fish: int, chips: int) -> Receipt: ... -->
<!---->
<!-- order(2, 1)  # now 2 fish, 1 chip — silently wrong! -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Backwards compatibility — keyword args are your friend -->
<!---->
<!-- Add features without breaking existing callers -->
<!---->
<!-- ```python -->
<!-- # v1.0 — simple -->
<!-- def order(chips: int = 0, fish: int = 0) -> Receipt: ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # v1.1 — added fish_type, fully backwards compatible -->
<!-- from enum import Enum -->
<!---->
<!---->
<!-- class FishType(Enum): -->
<!--     BATTERED = "battered" -->
<!--     CRUMBED = "crumbed" -->
<!---->
<!---->
<!-- def order( -->
<!--     chips: int = 0, fish: int = 0, fish_type: FishType = FishType.BATTERED -->
<!-- ) -> Receipt: ... -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Backwards compatibility — keyword args are your friend -->
<!---->
<!-- ```python -->
<!-- # v1.2 — added new foods, still backwards compatible -->
<!-- def order( -->
<!--     chips: int = 0, -->
<!--     fish: int = 0, -->
<!--     fish_type: FishType = FishType.BATTERED, -->
<!--     onion_rings: int = 0, -->
<!--     hotdogs: int = 0, -->
<!-- ) -> Receipt: ... -->
<!-- ``` -->
<!---->
<!-- Every old call still works. New keyword args with defaults = **no breaking changes**. -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Backwards compatibility — in aiida-core -->
<!---->
<!-- Good: `Process.__init__` — all parameters have defaults -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/processes/process.py -->
<!-- def __init__( -->
<!--     self, -->
<!--     inputs: Optional[Dict[str, Any]] = None, -->
<!--     logger: Optional[logging.Logger] = None, -->
<!--     runner: Optional['Runner'] = None, -->
<!--     parent_pid: Optional[int] = None, -->
<!--     enable_persistence: bool = True, -->
<!-- ) -> None: ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- Bad: internal helper with 7 positional parameters -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/daemon/execmanager.py -->
<!-- async def _copy_remote_files( -->
<!--     logger, node, computer, transport, -->
<!--     remote_copy_list, remote_symlink_list, workdir -->
<!-- ):  # 7 positional args — adding one breaks all callers -->
<!-- ``` -->
<!---->
<!-- Should use keyword-only arguments: `async def _copy_remote_files(*, logger, node, ...)`. Even for internal functions, keyword args prevent ordering mistakes. -->
<!---->
<!-- </v-click> -->
<!---->
---

# API: Minimize mutable state and side effects

Prefer returning new values over mutating inputs

```python
# Surprising — mutates the input
def normalize(data: list[float]) -> list[float]:
    max_val = max(data)
    for i in range(len(data)):
        data[i] /= max_val
    return data  # returns the SAME list, mutated


raw = [1.0, 2.0, 3.0]
normed = normalize(raw)
# raw is now [0.33, 0.66, 1.0] — surprise!
# raw IS normed — they're the same object
```

<v-click>

```python
# Better — returns a new list, input unchanged
def normalize(data: list[float]) -> list[float]:
    max_val = max(data)
    return [x / max_val for x in data]


raw = [1.0, 2.0, 3.0]
normed = normalize(raw)
# raw is still [1.0, 2.0, 3.0]
```

</v-click>

<v-click>

- `frozen=True` dataclasses are immutable by default
- `tuple` over `list` for fixed collections — signals "this won't change"
- Immutable data is easier to reason about, cache, and share across threads

</v-click>

---

# API: Minimize mutable state — in aiida-core

Bad: slurm scheduler mutated the caller's job list ([#7155](https://github.com/aiidateam/aiida-core/pull/7155))

```python
# aiida/engine/processes/calcjobs/manager.py (before fix)
class JobsList:
    def __init__(self):
        self._polling_jobs: list[str] = []            # mutable list

    async def _get_jobs_from_scheduler(self):
        self._polling_jobs = [str(j) for j in self._job_update_requests]
        scheduler.get_jobs(jobs=self._polling_jobs)   # passes list by reference

# aiida/schedulers/plugins/slurm.py
def _get_joblist_command(self, jobs=None, ...):
    joblist = jobs              # no copy — same object as _polling_jobs!
    joblist.append(jobs[0])    # mutates _polling_jobs → jobs appear twice
```

---

# API: Minimize mutable state — in aiida-core

Fix: make the data immutable so mutation is impossible

```python
# manager.py (after fix)
self._polling_jobs: frozenset[str] = frozenset()  # immutable

async def _get_jobs_from_scheduler(self):
    self._polling_jobs = frozenset(str(j) for j in self._job_update_requests)
    scheduler.get_jobs(jobs=self._polling_jobs)    # passes frozenset directly

# slurm.py (after fix)
def _get_joblist_command(self, jobs=None, ...):
    joblist = list(jobs)       # converts to list for local use — can't mutate caller
```

`frozenset` makes the bug **structurally impossible** — the scheduler can't append to a frozen set.

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

</v-click>

---

# API: Progressive disclosure — in aiida-core

Anti-pattern: `core.ssh` transport — 18 options upfront

```bash
❯ verdi -p dev computer configure core.ssh daint
Report: enter ? for help.
Report: enter ! to ignore the default and set no value.
User name [geiger_j]:
Port number [22]:
Look for keys [Y/n]:
SSH key file []: /home/geiger_j/.ssh/cscs/cscs-key
Connection timeout in s [60]:
Allow ssh agent [Y/n]:
SSH proxy jump []:
SSH proxy command []:
Compress file transfers [Y/n]:
GSS auth [False]:
GSS kex [False]:
GSS deleg_creds [False]:
GSS host [daint.alps.cscs.ch]:
Load system host keys [Y/n]:
Key policy (RejectPolicy, WarningPolicy, AutoAddPolicy) [RejectPolicy]:
Use login shell when executing command [Y/n]:
Connection cooldown time (s) [15.0]:
```


Most users (and developers) don't know what half of these mean — and most are just defaults.

---

# API: Progressive disclosure — in aiida-core

Better: `core.ssh_async` — leverage the user's existing SSH config

```bash
❯ verdi computer configure core.ssh_async daint
Host as in 'ssh <HOST>' (needs to be a password-less setup in your ssh config) [daint.alps.cscs.ch]: daint
Maximum number of concurrent I/O operations [8]:
Local script to run before opening connection (path) [None]:
Type of async backend to use, `asyncssh` or `openssh` [asyncssh]:
Use login shell when executing command [Y/n]:
Connection cooldown time (s) [15.0]:
```

<v-click>

6 options instead of 19 — and most are just Enter. The transport reads the user's `~/.ssh/config` directly — host, port, key, proxy settings live where they belong:

```bash
# ~/.ssh/config — the user already has this
Host daint
    HostName daint.cscs.ch
    User geiger_j
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump ela.cscs.ch
```

Instead of re-implementing SSH configuration, `core.ssh_async` delegates to OpenSSH and only exposes what's specific to AiiDA.

</v-click>

---

# API: Context managers as API

Resource management the Pythonic way

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Bad — easy to forget cleanup
conn = db.connect()
try:
    result = conn.execute("SELECT ...")
finally:
    conn.close()  # what if you forget?
```

</div>
<div>

<v-click>

```python
# Good — cleanup is automatic
with db.connect() as conn:
    result = conn.execute("SELECT ...")
# conn.close() called automatically
```

</v-click>

</div>
</div>

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

</v-click>

If your API involves **acquire/release**, **open/close**, or **setup/teardown**, make it a context manager. Users get correctness for free.

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

# API: Context managers — in aiida-core (cont.)

Bad: `get_container()` returned an unmanaged resource ([#6741](https://github.com/aiidateam/aiida-core/pull/6741))

```python
# aiida/storage/psql_dos/migrator.py (before fix)
def get_container(self) -> Container:
    return Container(get_filepath_container(self.profile))

# Callers forgot to close:
container = self.get_container()
container.init_container(clear=True)
# container is never closed → file descriptors leak
```

---

# API: Context managers — in aiida-core (cont.)

Fix: replace with a context manager, deprecate the old method

```python
# after fix
def get_container(self):
    raise DeprecatedError(
        'Do not use get_container as it might leave resources open. '
        'Use container_context() instead.'
    )

@contextmanager
def container_context(self) -> Generator[Container, None, None]:
    with Container(get_filepath_container(self.profile)) as container:
        yield container

# Callers now can't forget:
with self.container_context() as container:
    container.init_container(clear=True)
# container is closed automatically
```

---

<!-- # API: Async API mirrors -->
<!---->
<!-- Designing sync + async without duplicating everything -->
<!---->
<!-- ```python -->
<!-- import httpx -->
<!---->
<!-- # Sync client -->
<!-- resp = httpx.get("https://example.com") -->
<!---->
<!-- # Async client — same API shape -->
<!-- async with httpx.AsyncClient() as client: -->
<!--     resp = await client.get("https://example.com") -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- **Design considerations**: -->
<!---->
<!-- - Mirror the sync API as closely as possible — same method names, same parameters -->
<!-- - Use separate classes (`Client` / `AsyncClient`), not a flag -->
<!-- - Types help: `-> Response` vs `-> Awaitable[Response]` are distinct contracts -->
<!-- - Avoid `is_async` boolean parameters — they make types imprecise -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # Bad — one class, unclear types -->
<!-- client.get(url, async_mode=True)  # returns... what? -->
<!---->
<!---->
<!-- # Good — distinct types make the contract clear -->
<!-- def get(self, url: str) -> Response: ...  # Client -->
<!-- async def get(self, url: str) -> Response: ...  # AsyncClient -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

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

<!-- # API: Deprecation strategy -->
<!---->
<!-- Guide users from old API to new -->
<!---->
<!-- ```python -->
<!-- import warnings -->
<!-- from typing import overload -->
<!---->
<!---->
<!-- @overload -->
<!-- def connect(url: str) -> Connection: ... -->
<!-- @overload -->
<!-- def connect(host: str, port: int) -> Connection: ... -->
<!---->
<!---->
<!-- def connect(url_or_host: str, port: int | None = None) -> Connection: -->
<!--     if port is not None: -->
<!--         warnings.warn( -->
<!--             "connect(host, port) is deprecated, use connect(url) instead. " -->
<!--             "Will be removed in v3.0.", -->
<!--             DeprecationWarning, -->
<!--             stacklevel=2, -->
<!--         ) -->
<!--         url_or_host = f"https://{url_or_host}:{port}" -->
<!--     return Connection(url_or_host) -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - `@overload` keeps both old and new signatures typed during the transition -->
<!-- - `DeprecationWarning` is shown by test runners but hidden in production -->
<!-- - `stacklevel=2` points the warning at the **caller**, not the library -->
<!-- - Set a clear removal timeline (e.g., "removed in v3.0") -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Deprecation strategy — in aiida-core -->
<!---->
<!-- Good: centralized utility with configurable filtering -->
<!---->
<!-- ```python -->
<!-- # aiida/common/warnings.py -->
<!-- def warn_deprecation(message: str, version: int, stacklevel: int = 2) -> None: -->
<!--     from_config = get_config_option('warnings.showdeprecations') -->
<!--     from_environment = os.environ.get(f'AIIDA_WARN_v{version}') -->
<!---->
<!--     if from_config or from_environment: -->
<!--         message = f'{message} (this will be removed in v{version})' -->
<!--         warnings.warn(message, AiidaDeprecationWarning, stacklevel=stacklevel) -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- Bad: informal deprecation without concrete migration path -->
<!---->
<!-- ```python -->
<!-- # aiida/transports/plugins/ssh.py -->
<!-- def chdir(self, path): -->
<!--     """ -->
<!--     PLEASE DON'T USE `chdir()` IN NEW DEVELOPMENTS, INSTEAD -->
<!--     DIRECTLY PASS ABSOLUTE PATHS TO INTERFACE. -->
<!--     `chdir()` is DEPRECATED and will be removed in the next -->
<!--     major version. -->
<!--     """ -->
<!-- ``` -->
<!---->
<!-- "PLEASE DON'T USE" in a docstring is not a deprecation strategy. Which interface? What's the concrete replacement? A proper deprecation tells you *exactly* what to write instead. -->
<!---->
<!-- </v-click> -->

<!-- PR 7116 (on_unhandled_failure) added to types.md Literals/Enums section -->
