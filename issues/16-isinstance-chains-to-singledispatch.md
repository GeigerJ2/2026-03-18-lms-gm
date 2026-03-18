# Replace `isinstance` chains with `@singledispatch`

## Problem

Several places use `isinstance` chains instead of extensible dispatch:

```python
# aiida/tools/dbimporters/plugins/oqmd.py
if not isinstance(values, str) and not isinstance(values, int):
    raise ValueError(
        f"incorrect value for keyword '{alias}' "
        "-- only strings and integers are accepted"
    )
```

This pattern is closed — adding support for a new type requires modifying the original function.

## Proposed fix

Use `@singledispatch` for extensible, type-based dispatch:

```python
from functools import singledispatch


@singledispatch
def validate_keyword_value(value: object, alias: str) -> None:
    raise TypeError(
        f"Unsupported type {type(value).__name__} for keyword '{alias}' "
        "— only str and int are accepted"
    )


@validate_keyword_value.register(str)
@validate_keyword_value.register(int)
def _(value: str | int, alias: str) -> None:
    pass  # valid types, nothing to do
```

New types can register handlers without modifying the original function. This also applies to other `isinstance` chains in the codebase (e.g., in serialization, entity conversion).

## Files affected

- `src/aiida/tools/dbimporters/plugins/oqmd.py`
- Audit other `isinstance` chains across the codebase
