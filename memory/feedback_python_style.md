---
name: Python Type Annotation Style
description: User prefers PEP 604 union syntax — X | None instead of Optional[X], no Union[], no from __future__ import annotations
type: feedback
originSessionId: 40881ff9-65ee-4a5c-9a0e-41dd681b2bb2
---
Use PEP 604 union syntax throughout the codebase.

**Why:** User explicitly requested "new Python style (PEP 604)" when they saw Optional[X] usage.

**How to apply:**
- Use `X | None` instead of `Optional[X]`
- Use `X | Y` instead of `Union[X, Y]`
- Do not add `from __future__ import annotations` (Python ≥ 3.10 is required)
- Import `Callable`, `Awaitable`, `Iterator`, etc. from `collections.abc`, not `typing` (PEP 585)
- Use built-in generics: `list[X]`, `dict[K, V]`, `tuple[X, ...]`, `set[X]` — not `List`, `Dict`, etc.
- The `typing` module should not be imported at all (except `Literal` — see below)
- Use `Literal["A", "B"]` (from `typing`) for fields with a known fixed set of valid string values; e.g. `num1: Literal["User", "Handset"]`. Apply in both model field declarations and method signatures.
