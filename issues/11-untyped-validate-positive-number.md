# Add type annotations to `validate_positive_number` and similar validators

## Problem

`aiida/transports/transport.py` has untyped click validators:

```python
def validate_positive_number(ctx, param, value):  # no annotations!
    if not isinstance(value, (int, float)) or value < 0:
        from click import BadParameter
        raise BadParameter(f'{value} is not a valid positive number')
    return value
```

Without annotations, the type checker can't verify callers pass the right types, and the return type is invisible.

## Proposed fix

Add annotations:

```python
def validate_positive_number(
    ctx: click.Context, param: click.Parameter, value: float
) -> float:
    if not isinstance(value, (int, float)) or value < 0:
        raise click.BadParameter(f'{value} is not a valid positive number')
    return value
```

This should be done for all click validators in the transport module.

## Files affected

- `src/aiida/transports/transport.py`
