# Story — Allow chunk_size=None in Response.iter_lines type annotation

---

## Meta

- **ID**: PER-55
- **Type**: story
- **Track**: Quick
- **Repository**: demo-test
- **Epic**: (standalone)
- **Status**: draft
- **Assignee**: (unassigned)

---

## Problem

`Response.iter_content()` in `src/requests/models.py` correctly declares `chunk_size` as `int | None`, but `Response.iter_lines()` declares it as `int` in all overloads and its implementation.

Since `iter_lines()` passes `chunk_size` directly to `iter_content()`, passing `None` works at runtime, but type checkers incorrectly reject the call.

---

## What Success Looks Like

All `iter_lines()` overloads and the implementation accept `chunk_size: int | None` in their type annotations, making `iter_lines(chunk_size=None)` valid for type checkers. No runtime behavior changes occur.

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

## Technical Specification

### Scope

**Repository**: `demo-test`  
**File**: `src/requests/models.py`

Update the `chunk_size` parameter annotation in all `iter_lines()` overloads and the implementation from `int` to `int | None`.

### Changes Required

1. **Locate all `iter_lines()` signatures** in `src/requests/models.py`:
   - All `@overload` decorated variants
   - The implementation method

2. **Update each signature** to change:
   ```python
   chunk_size: int
   ```
   to:
   ```python
   chunk_size: int | None
   ```

3. **Verify no runtime changes**:
   - The implementation already passes `chunk_size` directly to `iter_content(chunk_size=chunk_size, ...)`
   - `iter_content()` already accepts `int | None`
   - No new logic is needed; this is purely a type annotation fix

### Implementation Notes

- **Do not modify**: The default `ITER_CHUNK_SIZE` constant value
- **Do not modify**: `iter_content()` method or its annotations
- **Do not modify**: Line-splitting logic, delimiter handling, or buffer behavior
- **Only change**: Type annotations on the `chunk_size` parameter in `iter_lines()`

### Files to Modify

- `src/requests/models.py` — Update all `iter_lines()` overload and implementation signatures

### Verification

Run the following to verify the change:

```python
import typing
from requests.models import Response

hints = typing.get_type_hints(Response.iter_lines)
assert hints["chunk_size"] == int | None
```

Run existing tests:

```bash
python -m pytest tests -k test_response_iter_lines
```

All tests must pass with no behavior changes.

---

## Out of Scope

- Changing the default `ITER_CHUNK_SIZE` value
- Modifying `iter_content()` type annotations or behavior
- Altering line-splitting logic or delimiter handling
- Any other parameter or method changes in `models.py`

---

## Tests

### Test: AC1 — Type annotation allows `chunk_size=None`

**Verification approach**: Introspect `Response.iter_lines` type hints and assert `chunk_size` parameter is `int | None`.

**Implementation**:
```python
import typing
from requests.models import Response

def test_iter_lines_chunk_size_type_annotation():
    hints = typing.get_type_hints(Response.iter_lines)
    assert hints["chunk_size"] == int | None
```

---

### Test: AC2 — Runtime behavior unchanged for `chunk_size=None`

**Verification approach**: Create a `Response` with a small body, consume it, then call `iter_lines(chunk_size=None)` and compare output to `iter_lines()`.

**Implementation**:
```python
from requests.models import Response

def test_iter_lines_chunk_size_none_behavior():
    r = Response()
    r._content = b"line1\nline2\nline3"
    
    lines_default = list(r.iter_lines())
    
    r2 = Response()
    r2._content = b"line1\nline2\nline3"
    lines_none = list(r2.iter_lines(chunk_size=None))
    
    assert lines_default == lines_none
```

---

### Test: AC3 — Existing line-splitting behavior preserved

**Verification approach**: Call `iter_lines()` with various valid `chunk_size` values and verify behavior matches expectations before the change.

**Implementation**: Existing tests in `tests/test_requests.py` cover this (`test_response_iter_lines*` test methods). Run:

```bash
python -m pytest tests/test_requests.py -k test_response_iter_lines
```

All must pass.

---

### Test: AC4 — Tests pass

**Verification approach**: Run the full test suite, filtering for `test_response_iter_lines`.

**Implementation**:
```bash
python -m pytest tests -k test_response_iter_lines
```

Expected: All tests pass with exit code 0.

---

## Notes

- This is a type annotation fix only.
- No runtime behavior changes are expected or permitted.
- All `iter_lines()` overloads and the implementation must be updated consistently.
- The change aligns `iter_lines()` with `iter_content()`, which already accepts `int | None`.

---

## References

- Ticket: PER-55
- Repository: demo-test
- File: `src/requests/models.py`
