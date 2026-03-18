# Add `Final` to constants in `manage/configuration/settings.py`

## Problem

9 constants in `aiida/manage/configuration/settings.py` are plain assignments — nothing prevents accidental reassignment:

```python
DEFAULT_UMASK = 0o0077
DEFAULT_AIIDA_PATH_VARIABLE = 'AIIDA_PATH'
DEFAULT_CONFIG_DIR_NAME = '.aiida'
DEFAULT_CONFIG_FILE_NAME = 'config.json'
DEFAULT_CONFIG_INDENT_SIZE = 4
# ... 4 more
```

`settings.DEFAULT_UMASK = 0o0000` would silently succeed — a potential security issue.

## Proposed fix

```python
from typing import Final

DEFAULT_UMASK: Final = 0o0077
DEFAULT_AIIDA_PATH_VARIABLE: Final = 'AIIDA_PATH'
DEFAULT_CONFIG_DIR_NAME: Final = '.aiida'
DEFAULT_CONFIG_FILE_NAME: Final = 'config.json'
DEFAULT_CONFIG_INDENT_SIZE: Final = 4
# ... etc.
```

The type checker will flag any reassignment as an error.

This should also be applied to other constant-like module-level variables across the codebase (e.g., `EXIT_CODE_*` constants, `SCHEDULER_*` constants).

## Files affected

- `src/aiida/manage/configuration/settings.py`
- Audit other modules for module-level constants that should be `Final`
