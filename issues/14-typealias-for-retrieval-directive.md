# Extract `TypeAlias` for repeated `Sequence[Union[str, Tuple[str, str, str]]]` in CalcJobNode

## Problem

`aiida/orm/nodes/process/calculation/calcjob.py` repeats the same complex type 5+ times:

```python
def _validate_retrieval_directive(
    directives: Sequence[Union[str, Tuple[str, str, str]]]) -> None: ...

def set_retrieve_list(
    self, retrieve_list: Sequence[Union[str, Tuple[str, str, str]]]) -> None: ...

def get_retrieve_list(
    self) -> Optional[Sequence[Union[str, Tuple[str, str, str]]]]: ...

# ... repeated in set_retrieve_temporary_list, get_retrieve_temporary_list, etc.
```

## Proposed fix

Define a `TypeAlias` once:

```python
from typing import TypeAlias

RetrievalDirective: TypeAlias = str | tuple[str, str, str]
RetrievalList: TypeAlias = Sequence[RetrievalDirective]


def _validate_retrieval_directive(directives: RetrievalList) -> None: ...
def set_retrieve_list(self, retrieve_list: RetrievalList) -> None: ...
def get_retrieve_list(self) -> RetrievalList | None: ...
```

The alias names the concept ("retrieval directive"), making the code self-documenting.

## Files affected

- `src/aiida/orm/nodes/process/calculation/calcjob.py`
- Type alias definition in `src/aiida/common/datastructures.py` or a new `src/aiida/common/types.py`
