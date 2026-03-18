# Consolidate global singletons into a single `AiiDAContext`

## Problem

Four module-level globals are scattered across the codebase, each using `global` statements:

```python
# aiida/manage/configuration/__init__.py
CONFIG: Optional['Config'] = None

# aiida/manage/manager.py
MANAGER: Optional['Manager'] = None

# aiida/common/progress_reporter.py
PROGRESS_REPORTER = ProgressReporterNull

# aiida/plugins/entry_point.py (or similar)
OBJECT_LOADER = ...
```

Each has its own `get_X()` / `set_X()` / `reset_X()` pattern with `global` statements (suppressed via `# noqa: PLW0603`). This makes state management fragmented and error-prone — resetting one without the others can leave the system in an inconsistent state.

## Proposed fix

Consolidate into a single context object, backed by `contextvars`:

```python
# aiida/manage/context.py
from contextvars import ContextVar
from dataclasses import dataclass, field


@dataclass
class AiiDAContext:
    config: Config | None = None
    manager: Manager | None = None
    progress_reporter: type[ProgressReporter] = ProgressReporterNull
    object_loader: ObjectLoader | None = None


_context: ContextVar[AiiDAContext] = ContextVar('_context', default_factory=AiiDAContext)


def get_context() -> AiiDAContext:
    return _context.get()


@contextmanager
def profile_context(profile: str | None = None):
    old = _context.get()
    new = AiiDAContext(config=load_config_for_profile(profile))
    token = _context.set(new)
    try:
        yield new
    finally:
        _context.reset(token)
```

Benefits:
- One place to manage all runtime state
- Resetting is atomic — no partial state
- Thread/async-safe via `contextvars`
- Easier to test — inject a fresh `AiiDAContext` per test

## Files affected

- `src/aiida/manage/configuration/__init__.py`
- `src/aiida/manage/manager.py`
- `src/aiida/common/progress_reporter.py`
- All call sites using `get_config()`, `get_manager()`, `set_progress_reporter()`
