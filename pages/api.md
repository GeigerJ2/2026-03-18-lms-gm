# API: `urllib` vs `requests`

Example from `urllib`, part of Python `stdlib`

<div class="grid grid-cols-[55%_45%] gap-4">
<div>

```python
import urllib

manager = urllib.request.HTTPPasswordMgrWithDefaultRealm()
manager.add_password(
    None, "https://httpbin.org/", "usr", "pwd",
)
handler = urllib.request.HTTPBasicAuthHandler(manager)
opener = urllib.request.build_opener(handler)
response = opener.open(
    "https://httpbin.org/basic-auth/usr/pwd",
)
print(response.status)
```

</div>
<div>

<v-click>

Could you write this from scratch?

- **3 concepts**: manager, handler, opener
- **2 absurd names**: `HTTPPasswordMgrWithDefaultRealm`, `HTTPBasicAuthHandler`
- A mystery `None` as first argument to `add_password`

</v-click>

</div>
</div>

<v-click>
A good API shouldn't require a tutorial for basic operations!
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

Same task, which code would you rather write? **API design matters.**

In the following slides, we'll explore what makes an API good, drawing on principles from Ben Hoyt's [*Designing Pythonic library APIs*](https://benhoyt.com/writings/python-api-design/).

We'll use a fictional `fishnchips` library as a running example, alongside real examples from `aiida-core`.

</v-click>

---

# API: Flat is better than nested

Expose a clean API for common functionality; file structure is an implementation detail

```python
# Bad — user must know internal file structure
from django.core.files.images import ImageFile  # 4 levels deep!
from django.contrib.auth.hashers import make_password
```

<v-click>

Re-exports in `aiida.orm` and `aiida.engine` ✅

```python
# aiida/orm/__init__.py — 90+ classes re-exported
from .authinfos import *
from .comments import *
from .nodes import *
from .querybuilder import *

# User code
from aiida.orm import Int, CalcJobNode, QueryBuilder  # all flat
```

</v-click>

<v-click>

In `aiida-core`, by convention, anything not importable from the second level is **private API**.

However, in Python it's not enforced and the line gets blurry between users, workflow devs, and plugin devs.

</v-click>

---

# API: Naming — short as possible, clear as necessary

The module name is already context — `import lib` then `lib.thing()`

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Bad — where does order() come from?
from fishnchips import order

def eat_takeaways():
    meal = order(chips=1)
    # ordering what? from where?
```

</div>
<div>

<v-click>

```python
# Good — immediately clear
import fishnchips

def eat_takeaways():
    meal = fishnchips.order(chips=1)
    # obviously fish & chips
```

</v-click>

</div>
</div>

<v-click>

Don't name your function `order_fishnchips` — `fishnchips.order()` reads naturally.

Same reason `requests.get()` works — you're not meant to `from requests import get`.

Think about how you want users to interact with your library?

@mbercx: _docs-driven_ development helps flesh out the API

</v-click>

---

# API: Naming — short as possible, clear as necessary

The same applies for classes — no need to repeat the class name in method names

<div class="grid grid-cols-2 gap-4">
<div>

```python
# aiida/orm/nodes/data/array/kpoints.py ❌
class KpointsData(ArrayData):
    def set_kpoints_mesh(self, mesh, ...): ...
    def get_kpoints_mesh(self, ...): ...
    def set_kpoints_mesh_from_density(self, ...): ...

# "kpoints" said three times in one line:
kpoints.set_kpoints_mesh([4, 4, 4])
```

</div>
<div>

```python
# let the class name provide context ✅
class KpointsData(ArrayData):
    def set_mesh(self, mesh, ...): ...
    def get_mesh(self, ...): ...
    def set_mesh_from_density(self, ...): ...

kpoints.set_mesh([4, 4, 4])
mesh = kpoints.get_mesh()
```

</div>
</div>

---

# API: Minimize mutable state and side effects

Prefer returning new values over mutating inputs

<div class="grid grid-cols-2 gap-4">
<div>

```python
# Surprising — mutates the input
def normalize(data: list[float]) -> list[float]:
    max_val = max(data)
    for i in range(len(data)):
        data[i] /= max_val
    return data  # same object, mutated

raw = [1.0, 2.0, 3.0]
normed = normalize(raw)
# raw is now [0.33, 0.66, 1.0] — surprise!
```

</div>
<div>

<v-click>

```python
# Better — returns a new list
def normalize(data: list[float]) -> list[float]:
    max_val = max(data)
    return [x / max_val for x in data]

raw = [1.0, 2.0, 3.0]
normed = normalize(raw)
# raw is still [1.0, 2.0, 3.0] ✅
```

</v-click>

</div>
</div>

<v-click>

Tools: `tuple` over `list`, `frozenset`, `frozen=True` dataclasses

</v-click>

<v-click>

**In aiida-core:** Slurm scheduler mutated the caller's job list by reference — fix: `list` → `frozenset` + defensive copy ([#7155](https://github.com/aiidateam/aiida-core/pull/7155) by @GeigerJ2 and @khsrali)

</v-click>

<div class="absolute bottom-4 left-12 text-xs opacity-50">#7155: <i>Fix: Avoid mutating JobsList._polling_jobs inside slurm scheduler</i></div>

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

- Safe defaults
- Unsafe options require explicit action
- Make wrong code look wrong: if misuse looks identical to correct use, the API has failed

</v-click>

---

# API: Progressive disclosure

<!-- TODO: Add the ssh configuration as a wrong example here -->

Simple things simple, complex things possible

```python
# Level 1 — one line, sensible defaults
response = requests.get("https://api.example.com/users")
```

<v-click>

```python
# Level 2 — add options as needed
response = requests.get(
    "https://api.example.com/users",
    params={"page": 2},
    timeout=5.0,
)
```

</v-click>

<v-click>

```python
# Level 3 — full control with a session
session = requests.Session()
session.auth = ("user", "pass")
session.headers.update({"Accept": "application/json"})
response = session.get("/users", params={"page": 2})
```

</v-click>

<v-click>

**Don't front-load complexity.** Let users discover features when they need them.

**Tools:** keyword arguments with defaults, module-level functions wrapping classes, layered APIs.

</v-click>

---

# API: Progressive disclosure — in aiida-core

`core.ssh` transport — 17 options upfront ❌

```bash
❯ verdi computer configure core.ssh daint
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

Better: `core.ssh_async` (@khsrali) — leverage the user's existing SSH config ✅

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

6 options instead of 17!

Transport reads the user's `~/.ssh/config` directly — host, key, proxy settings live where they belong:

```bash
# ~/.ssh/config — the user already has this
Host daint
    HostName daint.cscs.ch
    User geiger_j
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump ela.cscs.ch
```

`core.ssh_async` delegates to `OpenSSH` / `AsyncSSH` and only exposes what's specific to AiiDA.

</v-click>

---

# API: Choose the right input representation

Same problem, two approaches ([#7069](https://github.com/aiidateam/aiida-core/pull/7069) vs [#7116](https://github.com/aiidateam/aiida-core/pull/7116)): control what happens on unhandled process failure

<div class="grid grid-cols-2 gap-4">
<div>

**Two boolean flags** (PR #7069)

```python
# aiida/engine/processes/workchains/restart.py
spec.input(
    'pause_for_unknown_errors',
    valid_type=orm.Bool,
    default=lambda: orm.Bool(False),
)
spec.input(
    'restart_once_for_unknown_errors',
    valid_type=orm.Bool,
    default=lambda: orm.Bool(True),
)
```

</div>
<div>

**One string input** (PR #7116)

```python
# aiida/engine/processes/workchains/restart.py
spec.input(
    'on_unhandled_failure',
    valid_type=orm.Str,
    required=False,
    validator=validate_on_unhandled_failure,
)
# options: 'abort', 'pause',
#   'restart_once', 'restart_and_pause'
```

</div>
</div>

<v-click>

Two booleans → **2 extra levels of nesting** to resolve 4 combinations. One string → flat `if/elif` dispatch, each case self-contained.

</v-click>

<div class="absolute bottom-4 left-12 text-xs opacity-50">#7069: <i>Pause BaseRestartWorkChain on unhandled errors</i> · #7116: <i>Add on_unhandled_failure input</i></div>
