---
name: Python Type Annotation Style
description: User prefers PEP 604 union syntax — X | None instead of Optional[X], no Union[], no from __future__ import annotations
type: feedback
---

Use PEP 604 union syntax throughout the codebase.

**Why:** User explicitly requested "new Python style (PEP 604)" when they saw Optional[X] usage.

**How to apply:**
- Use `X | None` instead of `Optional[X]`
- Use `X | Y` instead of `Union[X, Y]`
- Do not add `from __future__ import annotations` (Python ≥ 3.10 is required)
- Import `Callable`, `Awaitable`, `Iterator`, etc. from `collections.abc`, not `typing` (PEP 585)
- Use built-in generics: `list[X]`, `dict[K, V]`, `tuple[X, ...]`, `set[X]` — not `List`, `Dict`, etc.
- The `typing` module should not be imported at all
