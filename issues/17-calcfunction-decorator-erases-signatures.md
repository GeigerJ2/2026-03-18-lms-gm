# Fix `@calcfunction` / `@workfunction` decorator to preserve signatures at runtime

## Problem

The `ProcessFunctionType` Protocol correctly types the decorated function at the static analysis level, but the runtime implementation erases the signature with bare `*args, **kwargs` and monkey-patches attributes:

```python
# aiida/engine/processes/functions.py
@functools.wraps(function)
def decorated_function(*args, **kwargs):  # signature erased!
    result, _ = run_get_node(*args, **kwargs)
    return result

decorated_function.run = decorated_function  # type: ignore[attr-defined]
decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined]
# ... 6 more type: ignore[attr-defined] lines
```

At runtime, `inspect.signature(decorated_function)` shows `(*args, **kwargs)` instead of the original parameters. The 8 `type: ignore[attr-defined]` lines indicate the monkey-patching doesn't satisfy the type checker.

## Proposed fix

Use a proper wrapper class that implements the `ProcessFunctionType` Protocol directly:

```python
class ProcessFunction(Generic[P, R_co, N]):
    """Wrapper that preserves the original function's signature and adds process methods."""

    def __init__(self, func: Callable[P, R_co], node_class: type[N], process_class: type[Process]):
        functools.update_wrapper(self, func)
        self._func = func
        self.node_class = node_class
        self.process_class = process_class
        self.is_process_function = True

    def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co:
        result, _ = self.run_get_node(*args, **kwargs)
        return result

    def run(self, *args: P.args, **kwargs: P.kwargs) -> R_co:
        return self(*args, **kwargs)

    def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[dict[str, Any] | None, N]:
        ...

    def run_get_pk(self, *args: P.args, **kwargs: P.kwargs) -> tuple[dict[str, Any] | None, int]:
        ...
```

Benefits:
- `inspect.signature()` returns the original signature (via `update_wrapper`)
- No monkey-patching — all attributes are proper class members
- No `type: ignore` needed
- The `ParamSpec` typing flows through correctly

## Files affected

- `src/aiida/engine/processes/functions.py`
