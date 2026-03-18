# Replace global `CONFIG` with `contextvars.ContextVar`

## Problem

`aiida/manage/configuration/__init__.py` uses a module-level global for the config singleton:

```python
CONFIG: Optional['Config'] = None

def get_config(create=False) -> 'Config':
    global CONFIG  # noqa: PLW0603
    if not CONFIG:
        CONFIG = load_config(create=create)
    return CONFIG

def reset_config():
    """.. warning:: This is experimental functionality and should for now
    be used only internally. If the reset is unclean weird unknown
    side-effects may occur that end up corrupting or destroying data."""
    global CONFIG  # noqa: PLW0603
    CONFIG = None
```

The `reset_config()` function has a warning about unknown side effects, and the global state is not thread-safe or async-safe.

## Proposed fix

Use `contextvars.ContextVar` (PEP 567):

```python
from contextvars import ContextVar
from contextlib import contextmanager

_config: ContextVar[Config | None] = ContextVar('_config', default=None)


def get_config(create: bool = False) -> Config:
    config = _config.get()
    if config is None:
        config = load_config(create=create)
        _config.set(config)
    return config


@contextmanager
def profile_context(profile: str | None = None):
    token = _config.set(load_config_for_profile(profile))
    try:
        yield
    finally:
        _config.reset(token)  # restores previous value cleanly
```

Benefits:
- No `global` keyword needed
- Thread-safe and async-safe — each context gets its own value
- `reset_config()` disappears — `token.reset()` restores the previous value with no side effects
- `profile_context()` becomes trivially correct
- The call-site API (`get_config()`, `profile_context()`) stays identical

## Files affected

- `src/aiida/manage/configuration/__init__.py`
