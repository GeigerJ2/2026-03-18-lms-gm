# Replace informal `chdir()` deprecation with proper warning machinery

## Problem

`aiida/transports/plugins/ssh.py` uses a docstring shout instead of actual deprecation:

```python
def chdir(self, path):
    """
    PLEASE DON'T USE `chdir()` IN NEW DEVELOPMENTS, INSTEAD
    DIRECTLY PASS ABSOLUTE PATHS TO INTERFACE.
    `chdir()` is DEPRECATED and will be removed in the next
    major version.
    """
```

This is invisible to callers at runtime — no warning is emitted, and there's no concrete migration path ("DIRECTLY PASS ABSOLUTE PATHS TO INTERFACE" — which interface? which methods?).

## Proposed fix

Use AiiDA's existing deprecation machinery and document the concrete replacement:

```python
def chdir(self, path):
    """Change directory on the remote host.

    .. deprecated:: 2.x
        Use absolute paths with :meth:`put_object_from_filelike`,
        :meth:`get_object_content`, etc. instead.
    """
    warn_deprecation(
        'Transport.chdir() is deprecated. Pass absolute paths to '
        'put_object_from_filelike(), get_object_content(), listdir(), etc. instead.',
        version=3,
    )
    self._cwd = path
```

## Files affected

- `src/aiida/transports/plugins/ssh.py`
- Possibly `src/aiida/transports/transport.py` (base class)
