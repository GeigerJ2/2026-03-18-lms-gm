# Replace bare `Exception` with proper AiiDA exception in `orm/logs.py`

## Problem

`aiida/orm/logs.py` raises a bare `Exception` instead of using the AiiDA exception hierarchy:

```python
# aiida/orm/logs.py
if not isinstance(entity, nodes.Node):
    raise Exception('Only node logs are stored')  # bare Exception!
```

Callers cannot catch this specifically without catching all exceptions.

## Proposed fix

Use an existing AiiDA exception:

```python
if not isinstance(entity, nodes.Node):
    raise exceptions.InvalidOperation('Only node logs are stored')
```

Or, if input validation is the intent:

```python
if not isinstance(entity, nodes.Node):
    raise TypeError(f'Expected a Node instance, got {type(entity).__name__}')
```

## Files affected

- `src/aiida/orm/logs.py`
