# Replace `CalcInfo(DefaultFieldsAttributeDict)` with a dataclass

## Problem

`CalcInfo` inherits from `DefaultFieldsAttributeDict`, a dict subclass that makes all fields implicitly `Optional` with no constructor signature:

```python
# aiida/common/datastructures.py
class CalcInfo(DefaultFieldsAttributeDict):
    _default_fields = (
        'job_environment', 'email', 'email_on_started', 'uuid',
        'prepend_text', 'append_text', 'num_machines',
        'num_mpiprocs_per_machine', 'codes_info', ...
    )
    # No constructor signature — anything goes, no IDE autocomplete, no type checking
```

Users constructing a `CalcInfo` get no autocomplete, no type checking, and any typo in a field name silently creates a new key instead of raising an error.

## Proposed fix

Replace with a dataclass (or pydantic model):

```python
from dataclasses import dataclass, field


@dataclass
class CalcInfo:
    uuid: str
    codes_info: list[CodeInfo]
    num_machines: int = 1
    num_mpiprocs_per_machine: int | None = None
    prepend_text: str = ''
    append_text: str = ''
    email: str | None = None
    email_on_started: bool = False
    job_environment: dict[str, str] = field(default_factory=dict)
    # ... all fields explicitly typed with defaults where appropriate
```

This gives IDE autocomplete, type checking, and typo detection. Fields that are truly optional get explicit `| None`, required fields have no default.

The same applies to `JobInfo(DefaultFieldsAttributeDict)` — same pattern, same fix.

## Files affected

- `src/aiida/common/datastructures.py` (`CalcInfo`, `CodeInfo`, `CodeRunMode`)
- `src/aiida/schedulers/datastructures.py` (`JobInfo`, `JobTemplate`)
- `src/aiida/common/extendeddicts.py` (`DefaultFieldsAttributeDict` — can be removed once no longer used)
- All code constructing `CalcInfo` / `JobInfo` instances
