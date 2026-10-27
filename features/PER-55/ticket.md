---
key: PER-55
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Allow chunk_size=None in the Response.iter_lines type annotation
---

Problem

`Response.iter_content()` in `src/requests/models.py` correctly declares `chunk_size` as `int | None`, but `Response.iter_lines()` declares it as `int` in all overloads and its implementation.

Since `iter_lines()` passes `chunk_size` directly to `iter_content()`, passing `None` works at runtime, but type checkers incorrectly reject the call.

### Expected Behavior

Update all `iter_lines()` overloads and the implementation to accept `chunk_size: int | None`, without changing runtime behavior.

### Scope

* Repository: `requests`
* File: `src/requests/models.py`
* Update the `chunk_size` annotation in every `iter_lines()` overload and its implementation.
* Do not change line-splitting behavior, the default `ITER_CHUNK_SIZE`, or `iter_content()`.

### Acceptance Criteria

* `typing.get_type_hints(Response.iter_lines)["chunk_size"]` resolves to `int | None`.
* `iter_lines(chunk_size=None)` on an already-consumed response produces the same lines as `iter_lines()` for a small body.
* Existing line-splitting behavior remains unchanged.
* `python -m pytest tests` passes for the relevant `test_response_iter_lines` tests.
