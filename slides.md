---
theme: seriph
colorSchema: light
# background: https://cover.sli.dev
title: Why API and types matter
class: text-center
transition: slide-left
comark: true
---

# Why API and types matter

**Julian Geiger**

LMS Seminar · 18 Mar 2026

---

# Why should one care?

Your library's API is the **surface area** that every user touches

- A bad API leads to bugs, frustration, and misuse
- A good API makes the "right thing" easy and the "wrong thing" hard
- **You spend more time reading code than writing it** — API clarity pays off every day
- Code without types is implicit knowledge that lives in someone's head
- Types are the **machine-checkable** part of the API contract
- When a type is hard to write, it's often a signal that the design/API needs work

---

# Outline

<div class="grid grid-cols-2 gap-8">
<div>

**API Design**

1. `urllib` vs `requests`
2. Flat is better than nested
3. Naming — short as possible, clear as necessary
4. Minimize mutable state and side effects
5. The pit of success
6. Progressive disclosure
7. Choose the right input representation

</div>
<div>

**Types**

1. Function signatures are your API contract
2. `stubgen` — Python's "header files"
3. Types catch bugs and guide the API
4. Types reveal code smells
5. TypedDict and dataclasses over plain dicts
6. Literals and Enums over bare strings
7. `TypeAlias` — name your complex types
8. Protocols and abstract interfaces
9. Generics — preserving type information
10. `@overload` and `@singledispatch`
11. Variance & Postel's Law / LSP

</div>
</div>

---
layout: center
class: text-center
---

# API Design

What makes an API good or bad — and how to tell the difference.

---
src: ./pages/api.md
---

---
layout: center
class: text-center
---

# Types

A good API tells users *what* to pass and *what* to expect back.

**Types make that contract explicit and machine-checkable — if you actually run the type checker.**

---
src: ./pages/types.md
---

---
src: ./pages/footer.md
---

<!-- Backup slides — commented out, not shown in main deck -->
<!-- ---
src: ./pages/backup.md
--- -->
