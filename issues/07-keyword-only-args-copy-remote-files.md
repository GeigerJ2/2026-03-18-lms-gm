# Use keyword-only arguments in `_copy_remote_files`

## Problem

`aiida/engine/daemon/execmanager.py` has an internal helper with 7 positional parameters:

```python
async def _copy_remote_files(
    logger, node, computer, transport,
    remote_copy_list, remote_symlink_list, workdir
):  # 7 positional args — adding one breaks all callers, easy to mix up order
```

The argument order is easy to get wrong (is it `node, computer, transport` or `computer, node, transport`?), and adding a parameter is a breaking change for all callers.

## Proposed fix

Make arguments keyword-only:

```python
async def _copy_remote_files(
    *,
    logger: logging.Logger,
    node: CalcJobNode,
    computer: Computer,
    transport: Transport,
    remote_copy_list: list[tuple[str, str, str]],
    remote_symlink_list: list[tuple[str, str, str]],
    workdir: PurePosixPath,
) -> None: ...
```

Alternatively, `logger`, `node`, `computer`, and `transport` always travel together — bundle them:

```python
@dataclass(frozen=True)
class RemoteJobContext:
    logger: logging.Logger
    node: CalcJobNode
    computer: Computer
    transport: Transport


async def _copy_remote_files(
    ctx: RemoteJobContext,
    *,
    remote_copy_list: list[tuple[str, str, str]],
    remote_symlink_list: list[tuple[str, str, str]],
    workdir: PurePosixPath,
) -> None: ...
```

## Files affected

- `src/aiida/engine/daemon/execmanager.py`
- All call sites of `_copy_remote_files`
