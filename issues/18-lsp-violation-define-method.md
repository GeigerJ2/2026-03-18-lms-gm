# Fix LSP violation in `define()` method hierarchy

## Problem

The process hierarchy overrides `define()` with progressively **narrower** parameter types, violating the Liskov Substitution Principle:

```python
# Process (base)
@classmethod
def define(cls, spec: ProcessSpec) -> None:  # type: ignore[override]
    super().define(spec)

# CalcJob (subclass) — demands a more specific spec type
@classmethod
def define(cls, spec: CalcJobProcessSpec) -> None:  # type: ignore[override]
    super().define(spec)
    spec.inputs.validator = validate_calc_job  # type: ignore[assignment]
```

Every level has `# type: ignore[override]` because the type checker correctly identifies that a `CalcJob` cannot be used everywhere a `Process` is expected — it demands a more specific spec.

## Proposed fix

Make `Process` generic over its spec type:

```python
S = TypeVar('S', bound=ProcessSpec)


class Process(Generic[S]):
    @classmethod
    def define(cls, spec: S) -> None:
        spec.input('metadata', ...)


class CalcJob(Process[CalcJobProcessSpec]):
    @classmethod
    def define(cls, spec: CalcJobProcessSpec) -> None:  # no override violation
        super().define(spec)
        spec.inputs.validator = validate_calc_job
```

Each subclass declares what spec type it uses via the generic parameter. The `define()` signature is consistent at every level — no `type: ignore[override]` needed.

This pattern also applies to the `ProcessSpec` hierarchy itself if it has similar override issues.

## Files affected

- `src/aiida/engine/processes/process.py` (`Process`)
- `src/aiida/engine/processes/calcjobs/calcjob.py` (`CalcJob`)
- `src/aiida/engine/processes/workchains/workchain.py` (`WorkChain`)
- `src/aiida/engine/processes/process_spec.py` (`ProcessSpec`, `CalcJobProcessSpec`)
