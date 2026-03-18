# Putting it all together — API design

| **Principle** | **Benefit** |
|---|---|
| Flat imports | File structure is an implementation detail |
| Naming | Short as possible, clear as necessary |
| Immutability | Fewer surprises, thread safety |
| Pit of success | Safe defaults, explicit opt-out |
| Progressive disclosure | Simple things simple, complex things possible |
| Input representation | Right abstraction reduces nesting |

---

# Putting it all together — Type system

| **Tool** | **Benefit** |
|---|---|
| Typed signatures | Machine-checkable contracts |
| dataclass / TypedDict | Structured data, typo detection |
| Enums / Literals | Constrained inputs, exhaustiveness |
| Protocols | Flexible abstractions, testability |
| Generics | Preserve type info across boundaries |
| @overload / @singledispatch | Precise dispatch by type |
| Variance / LSP | Subtyping done right |

---
layout: center
class: text-center
---

# Key takeaway

<div class="text-2xl mt-4 opacity-80">

A good API is easy to type. Well-typed code is well-designed code.

They enforce each other — types surface design flaws, and clean design makes types simple.

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

# Alternative conclusion

<div class="text-2xl mt-4 opacity-80">

Time for `aiida-core v3`?!

</div>

---
layout: center
class: text-center
---

# References

<div class="text-left inline-block">

- Ben Hoyt — [*Designing Pythonic library APIs*](https://benhoyt.com/writings/python-api-design/) (2023)
- Reviewing typing PRs by Daniel Hollas
- `Claude Code`

## Relevant PEPs

- [PEP 484 — Type Hints](https://peps.python.org/pep-0484/) (`@overload`, generics, variance)
- [PEP 544 — Protocols](https://peps.python.org/pep-0544/) (structural subtyping)
- [PEP 567 — Context Variables](https://peps.python.org/pep-0567/) (`ContextVar`)
- [PEP 586 — Literal Types](https://peps.python.org/pep-0586/)
- [PEP 589 — TypedDict](https://peps.python.org/pep-0589/)
- [PEP 612 — ParamSpec](https://peps.python.org/pep-0612/) (decorator signatures)
- [PEP 613 — TypeAlias](https://peps.python.org/pep-0613/)

</div>

---
layout: center
class: text-center
---

# Thank you!

<PoweredBySlidev mt-10 />