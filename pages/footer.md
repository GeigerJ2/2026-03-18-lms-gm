# Putting it all together

API design principles

| Principle | Benefit |
|---|---|
| Flat imports | File structure is an implementation detail |
| Naming | Short as possible, clear as necessary |
| Avoid global state | Use instances, `ContextVar` for thread safety |
| Pit of success | Safe defaults, explicit opt-out for danger |
| Progressive disclosure | Simple things simple, complex things possible |
| Immutability | Fewer surprises, thread safety |
| Context managers | Automatic resource cleanup |
| Errors | Custom exception hierarchy, never bare `Exception` |

---

# Putting it all together (cont.)

Type system tools

| Principle | Benefit |
|---|---|
| Typed signatures | Self-documenting, machine-checkable contracts |
| dataclass / TypedDict | Structured data with typo detection |
| Enums / Literals | Constrained inputs, exhaustiveness checking |
| NewType | Prevent value mix-ups at zero cost |
| Protocols | Flexible abstractions, testability |
| Generics | Preserve type info across boundaries |
| @overload | Precise return types per call signature |
| @singledispatch | Extensible runtime dispatch by type |
| ParamSpec | Decorators that preserve signatures |
| Final / @final | Lock down constants and methods |
| Variance | Read-only → covariant, mutable → invariant |

<v-click>

**Types are not just for catching bugs** — they shape how you think about your API.

When a type is hard to write, it's a signal that the design needs work.

</v-click>

---
layout: center
class: text-center
---

# Key takeaway

<div class="text-2xl mt-4 opacity-80">

Well-typed code is well-designed code.

</div>

---
layout: center
class: text-center
---

# Conclusion

<div class="text-2xl mt-4 opacity-80">

Rewrite it in Rust

🦀

</div>

---
layout: center
class: text-center
---

# References

<div class="text-left inline-block">

- Ben Hoyt — [*Designing Pythonic library APIs*](https://benhoyt.com/writings/python-api-design/) (2023)
- [PEP 544 — Protocols: Structural subtyping](https://peps.python.org/pep-0544/)
- [PEP 589 — TypedDict](https://peps.python.org/pep-0589/)
- [PEP 484 — Type Hints](https://peps.python.org/pep-0484/)
- [PEP 612 — ParamSpec](https://peps.python.org/pep-0612/)
- [PEP 742 — TypeIs](https://peps.python.org/pep-0742/)
- [Python docs — `typing` module](https://docs.python.org/3/library/typing.html)
- [Pyright](https://github.com/microsoft/pyright) / [mypy](https://mypy-lang.org/) — static type checkers

</div>

<PoweredBySlidev mt-10 />
