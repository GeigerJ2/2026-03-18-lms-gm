# Break circular import cycle between `orm` and `engine`

## Problem

`aiida.orm` and `aiida.engine` have a circular dependency that requires `TYPE_CHECKING` guards and delayed imports as workarounds:

```python
# aiida/engine/processes/process.py — imports orm at runtime
from aiida import orm
from aiida.orm.nodes.process.calculation.calcjob import CalcJobNode

# aiida/orm/nodes/process/process.py — can't import engine at runtime
if TYPE_CHECKING:
    from aiida.engine.processes import ExitCode, Process
    from aiida.engine.processes.builder import ProcessBuilder

# aiida/orm/utils/managers.py — delayed import inside __init__
class NodeLinksManager:
    def __init__(self, ...):
        from aiida.orm import Node  # "This import is here to avoid circular imports"
```

The cycle is: `engine.processes.process` imports `aiida.orm` (runtime) → `orm` re-exports `ProcessNode` → `ProcessNode` needs `ExitCode`/`Process` from `engine` → cycle.

## Proposed fix

Extract shared types into a leaf module that neither `orm` nor `engine` depends on:

```python
# aiida/common/process_types.py (new module — no orm/engine imports)
from typing import NamedTuple

class ExitCode(NamedTuple):
    status: int = 0
    message: str | None = None
    invalidates_cache: bool = False

# Also: ProcessSpec base, ProcessState enum, link type definitions, etc.
```

Both `orm` and `engine` import from `aiida.common.process_types` instead of from each other. The `TYPE_CHECKING` guards and delayed imports become unnecessary.

## Files affected

- `src/aiida/engine/processes/exit_code.py` → move `ExitCode` to `aiida.common.process_types`
- `src/aiida/orm/nodes/process/process.py` → remove `TYPE_CHECKING` block
- `src/aiida/orm/utils/managers.py` → remove delayed import
- `src/aiida/engine/processes/process.py` → import from new location
