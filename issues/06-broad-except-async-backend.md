# Replace broad `except Exception: pass` in async_backend.py

## Problem

`aiida/transports/plugins/async_backend.py` silently swallows all exceptions when checking the OpenSSH version:

```python
try:
    result = subprocess.run(['ssh', '-V'], capture_output=True, ...)
    match = re.search(r'OpenSSH_(\d+)', output)
    ...
except Exception:  # silently swallows ALL errors
    pass
```

This hides real errors (e.g. permissions issues, corrupted PATH) behind silent failure.

## Proposed fix

Catch only the expected exceptions and log them:

```python
try:
    result = subprocess.run(['ssh', '-V'], capture_output=True, ...)
    match = re.search(r'OpenSSH_(\d+)', output)
    ...
except (FileNotFoundError, subprocess.SubprocessError) as exc:
    logger.debug('Could not determine OpenSSH version: %s', exc)
```

## Files affected

- `src/aiida/transports/plugins/async_backend.py`
