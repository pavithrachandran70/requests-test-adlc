# Requirements — PER-55

Allow chunk_size=None in the Response.iter_lines type annotation

---

## Meta

- **Ticket**: PER-55
- **Type**: story
- **Repository**: demo-test
- **Files**: `src/requests/models.py`

---

## Context

`Response.iter_content()` in `src/requests/models.py` correctly declares `chunk_size` as `int | None`, but `Response.iter_lines()` declares it as `int` in all overloads and its implementation.

Since `iter_lines()` passes `chunk_size` directly to `iter_content()`, passing `None` works at runtime, but type checkers incorrectly reject the call.

---

## Acceptance Criteria

### AC1: Type annotation allows `chunk_size=None`

**Given** the `Response` class in `src/requests/models.py`  
**When** `typing.get_type_hints(Response.iter_lines)["chunk_size"]` is evaluated  
**Then** it resolves to `int | None`

---

### AC2: Runtime behavior unchanged for `chunk_size=None`

**Given** a `Response` object with a small body that has already been consumed  
**When** `iter_lines(chunk_size=None)` is called  
**Then** it produces the same lines as `iter_lines()` with no arguments

---

### AC3: Existing line-splitting behavior preserved

**Given** any `Response` object  
**When** `iter_lines()` is called with various valid `chunk_size` values  
**Then** the line-splitting behavior remains unchanged from before the annotation fix

---

### AC4: Tests pass

**Given** the test suite in `tests/`  
**When** `python -m pytest tests` is run, filtering for `test_response_iter_lines` tests  
**Then** all relevant tests pass

---

## Out of Scope

- Changing the default `ITER_CHUNK_SIZE` value
- Modifying `iter_content()` type annotations or behavior
- Altering line-splitting logic or delimiter handling
- Framework, language, ORM, or cloud stack decisions

---

## Notes

This is a type annotation fix only. No runtime behavior changes are expected or permitted. All `iter_lines()` overloads and the implementation must be updated to reflect that `chunk_size` accepts `int | None`, consistent with `iter_content()`.
