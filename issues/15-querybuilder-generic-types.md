# Make `QueryBuilder` generic to preserve projected types

## Problem

`QueryBuilder.all()` and `QueryBuilder.first()` return `list[list[Any]]` and `Any` respectively — all type information is lost through the query pipeline:

```python
# aiida/orm/querybuilder.py
def all(self, ...) -> list[list[Any]]:  # always Any
    ...

def first(self, flat: bool = False) -> list[Any] | Any | None:
    ...
```

Callers must cast manually:

```python
nodes: list[Node] = qb.all(flat=True)  # type: ignore
```

The `@overload` on `first(flat=...)` helps distinguish flat vs nested returns, but the element types are still `Any`.

## Proposed fix

Make `QueryBuilder` generic over the projected entity type:

```python
EntityT = TypeVar('EntityT')


class QueryBuilder(Generic[EntityT]):
    @overload
    def all(self, flat: Literal[True]) -> list[EntityT]: ...
    @overload
    def all(self, flat: Literal[False] = False) -> list[list[EntityT]]: ...

    @overload
    def first(self, flat: Literal[True]) -> EntityT | None: ...
    @overload
    def first(self, flat: Literal[False] = False) -> list[EntityT] | None: ...
```

With a factory or `append` method that narrows the type:

```python
qb = QueryBuilder().append(Node, tag='node')  # QueryBuilder[Node]
nodes = qb.all(flat=True)  # list[Node] — no cast needed
```

For multi-projection queries (multiple `append` calls), this gets harder — a pragmatic first step is to support the single-entity case, which covers the majority of usage.

## Files affected

- `src/aiida/orm/querybuilder.py`
