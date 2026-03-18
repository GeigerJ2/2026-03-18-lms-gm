# `create_entry_from_record` should raise instead of returning `None`

## Problem

`aiida/orm/logs.py` silently returns `None` when a `LogRecord` is missing `dbnode_id`:

```python
def create_entry_from_record(self, record: LogRecord) -> Optional['Log']:
    dbnode_id = record.__dict__.get('dbnode_id', None)

    if dbnode_id is None:
        return None  # caller must remember to check!
```

Callers that forget to check for `None` will get `AttributeError` later, far from the actual cause.

## Proposed fix

Raise an exception — the caller should know immediately that the record is invalid:

```python
def create_entry_from_record(self, record: LogRecord) -> 'Log':
    dbnode_id = record.__dict__.get('dbnode_id', None)

    if dbnode_id is None:
        raise ValueError(
            f'LogRecord is missing required "dbnode_id" attribute: {record}'
        )
    ...
```

The return type simplifies from `Optional['Log']` to `'Log'` — callers no longer need `None` checks.

## Files affected

- `src/aiida/orm/logs.py`
- Callers of `create_entry_from_record` (verify they don't rely on the `None` return)
