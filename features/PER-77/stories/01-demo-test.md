# Story PER-77 — Coerce non-string multipart field names

**Type:** bug  
**ID:** 77  
**Title:** Coerce non-string multipart field names instead of raising AttributeError  
**Epic:** _(none — backlog item)_  
**Status:** draft  
**Assignee:** _(unassigned)_

---

## Story

**As a** developer using the requests library  
**I want** non-string field names in multipart requests to be automatically coerced to strings  
**So that** I can use the same data dictionary for both urlencoded and multipart requests without crashes

---

## Background

Currently, `Request._encode_files` in `src/requests/models.py` converts field *values* with `str()` but passes field *names* through unchanged. When urllib3 processes these names, it calls `.translate()` on them, which raises `AttributeError` for non-string types like `int`. This inconsistency means `data={1: "a"}` works for urlencoded bodies but crashes when `files` is also set.

**Example of current behavior:**

```python
requests.Request("POST", "http://x", data={1: "a"}).prepare()
# works, body is 1=a

requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()
# AttributeError: 'int' object has no attribute 'translate'

requests.Request("POST", "http://x", files={1: b"x"}).prepare()
# AttributeError: 'int' object has no attribute 'translate'
```

---

## Acceptance Criteria

### AC1: Integer field names in `data` with `files` set

**Given** a POST request with `data={1: "a"}` and `files={"f": b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully  
**And** the multipart body contains the field name `"1"`

**Verification:**
```python
req = requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"})
prepared = req.prepare()
# Should not raise AttributeError
assert prepared.body is not None
assert b'name="1"' in prepared.body
```

---

### AC2: Integer field names in `files`

**Given** a POST request with `files={1: b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully  
**And** the multipart body contains the field name `"1"`

**Verification:**
```python
req = requests.Request("POST", "http://x", files={1: b"x"})
prepared = req.prepare()
# Should not raise AttributeError
assert prepared.body is not None
assert b'name="1"' in prepared.body
```

---

### AC3: Urlencoded behavior preserved

**Given** a POST request with `data={1: "a"}` and no `files`  
**When** `.prepare()` is called  
**Then** the request prepares successfully  
**And** the urlencoded body is `1=a` (integer key preserved, not stringified in the body)

**Verification:**
```python
req = requests.Request("POST", "http://x", data={1: "a"})
prepared = req.prepare()
assert prepared.body == b"1=a"
# Not b'"1"=a' — urlencoded encoding unchanged
```

---

### AC4: Test coverage

**Given** a new test `test_non_string_field_names` in `tests/test_requests.py`  
**When** `python -m pytest tests/test_requests.py::test_non_string_field_names` runs  
**Then** the test passes  
**And** when `python -m pytest tests` runs, all existing request-preparation tests pass

**Verification:**
```bash
python -m pytest tests/test_requests.py::test_non_string_field_names -v
# Exit code 0

python -m pytest tests
# Exit code 0, no regressions
```

---

## Technical Specification

### Change Scope

**File:** `src/requests/models.py`  
**Method:** `Request._encode_files`

**Current behavior:** Field names are passed through unchanged to urllib3. Field values are coerced with `str()`.

**Required change:** Coerce field names to strings before passing to urllib3.

**Implementation approach:**

1. Locate the section in `_encode_files` where field names are extracted from `data` and `files` dictionaries
2. Wrap each field name with `str()` before constructing the fields list passed to urllib3
3. Ensure this coercion happens for both `data` keys (when `files` is set) and `files` keys
4. Do NOT modify the urlencoded path — `_encode_params` should remain unchanged

**Example pseudo-code:**

```python
# In _encode_files, when iterating over data.items():
for k, v in data.items():
    field_name = str(k)  # <-- Add this coercion
    # ... rest of field construction

# Similarly for files.items():
for k, v in files.items():
    field_name = str(k)  # <-- Add this coercion
    # ... rest of field construction
```

---

### Test Requirements

**Test file:** `tests/test_requests.py`  
**Test function:** `test_non_string_field_names`

**Requirements:**

- Unit test using `.prepare()` — no network calls, no server fixture
- Cover three scenarios:
  1. Integer key in `data` with `files` set (AC1)
  2. Integer key in `files` (AC2)
  3. Integer key in `data` with no `files` (AC3 — urlencoded preserved)
- Assert no `AttributeError` raised
- Assert field names appear as strings in multipart bodies
- Assert urlencoded body unchanged
- Must not import or call any network-dependent fixtures

**Example test structure:**

```python
def test_non_string_field_names():
    """Non-string field names should be coerced to strings in multipart bodies."""
    
    # AC1: data with int key, files set
    req = requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"})
    prepared = req.prepare()
    assert prepared.body is not None
    assert b'name="1"' in prepared.body
    
    # AC2: files with int key
    req = requests.Request("POST", "http://x", files={1: b"x"})
    prepared = req.prepare()
    assert prepared.body is not None
    assert b'name="1"' in prepared.body
    
    # AC3: urlencoded preserved
    req = requests.Request("POST", "http://x", data={1: "a"})
    prepared = req.prepare()
    assert prepared.body == b"1=a"
```

---

### Constraints

1. **Do not modify urlencoded encoding:** `_encode_params` and the urlencoded path must remain unchanged. AC3 verifies this.
2. **Do not change file-tuple shapes:** The format of file tuples passed to `files` must remain backward-compatible.
3. **Do not alter public method signatures:** `Request.__init__`, `Request.prepare()`, and related public APIs must not change.
4. **Preserve existing behavior:** All existing request-preparation tests must continue to pass.

---

## Out of Scope

- Changing how urlencoded bodies serialize field names (they remain as-is)
- Modifying urllib3's internal field-name handling
- Altering public API contracts or method signatures
- Supporting non-stringifiable types (objects without `__str__` are out of scope)
- Changing behavior for string field names (no regression)

---

## Notes

- **Complexity:** Simple — single-method change with focused test
- **Risk:** Low — change is localized, well-constrained, and directly addresses the AttributeError
- **Verification approach:** Unit test only, no integration or network calls required
- **Regression risk:** Mitigated by preserving urlencoded path and running full test suite

---

## References

- Source ticket describes the bug and expected behavior
- `src/requests/models.py` contains `Request._encode_files`
- `tests/test_requests.py` is the standard location for request-preparation tests
