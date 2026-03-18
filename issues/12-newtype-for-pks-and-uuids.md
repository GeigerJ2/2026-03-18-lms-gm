# Use `NewType` for PKs and UUIDs to prevent wrong-entity bugs

## Problem

Node PKs, group PKs, UUIDs, and job IDs are all plain `int` or `str`. Nothing prevents passing a group PK where a node PK is expected:

```python
def load_node(pk: int) -> Node: ...
def load_group(pk: int) -> Group: ...

node = load_node(group.pk)  # no error — but wrong entity!
```

## Proposed fix

Use `NewType` to create distinct types at zero runtime cost:

```python
from typing import NewType

NodePk = NewType('NodePk', int)
GroupPk = NewType('GroupPk', int)
Uuid = NewType('Uuid', str)


def load_node(pk: NodePk) -> Node: ...
def load_group(pk: GroupPk) -> Group: ...

node = load_node(group.pk)  # type error: GroupPk is not NodePk
```

`NewType` is erased at runtime, so there is zero performance cost. The entity classes would type their `.pk` property accordingly:

```python
class Node:
    @property
    def pk(self) -> NodePk: ...

class Group:
    @property
    def pk(self) -> GroupPk: ...
```

## Files affected

- `src/aiida/orm/entities.py` (base `.pk` property)
- `src/aiida/orm/nodes/node.py`, `src/aiida/orm/groups.py` (typed `.pk`)
- `src/aiida/orm/utils/loaders.py` (`load_node`, `load_group`, etc.)
- New type definitions in `src/aiida/common/types.py` or similar
