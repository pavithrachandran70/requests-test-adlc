# Story PER-77 — Coerce non-string multipart field names instead of raising AttributeError

**Type:** bug  
**Track:** Quick  
**Epic:** (none)  
**ID:** 77  
**Title:** Coerce non-string multipart field names instead of raising AttributeError

---

## Story

As a developer using the requests library,  
I want to pass non-string keys in `data` or `files` when preparing multipart requests,  
so that I don't encounter an `AttributeError` when urllib3 attempts to encode field names.

---

## Context

Currently, `Request._encode_files` accepts non-string keys in `data` and `files` but does not coerce field names to strings before passing them to urllib3's multipart encoder. urllib3 expects string field names and calls `.translate()` on them, causing an `AttributeError` when an integer key is used. Urlencoded requests already handle non-string keys correctly (producing bodies like `1=a`), so multipart requests should behave consistently.

**Reproduction:**

```python
import requests

# Works — urlencoded body is "1=a"
requests.Request("POST", "http://x", data={1: "a"}).prepare()

# Fails — AttributeError: 'int' object has no attribute 'translate'
requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()

# Fails — AttributeError: 'int' object has no attribute 'translate'
requests.Request("POST", "http://x", files={1: b"x"}).prepare()
```

---

## Acceptance Criteria

### AC1: Integer keys in `data` succeed with multipart encoding

**Given** a POST request with `data={1: "a"}` and `files={"f": b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully without raising `AttributeError`  
**And** the multipart body includes the field name `"1"` with value `"a"`

**Verification:** Call `requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()` and assert no exception is raised. Inspect the prepared request's body to confirm field `"1"` is present.

---

### AC2: Integer keys in `files` succeed

**Given** a POST request with `files={1: b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully without raising `AttributeError`  
**And** the multipart body includes the field name `"1"`

**Verification:** Call `requests.Request("POST", "http://x", files={1: b"x"}).prepare()` and assert no exception is raised. Inspect the prepared request's body to confirm field `"1"` is present.

---

### AC3: Urlencoded behavior is preserved

**Given** a POST request with `data={1: "a"}` and no `files` parameter  
**When** `.prepare()` is called  
**Then** the request body is `1=a` (urlencoded format)  
**And** the behavior is identical to the current implementation

**Verification:** Call `requests.Request("POST", "http://x", data={1: "a"}).prepare()` and assert `prepared.body == b"1=a"` (or the exact urlencoded representation).

---

### AC4: Test coverage

**Given** a new unit test for non-string field names  
**When** the test runs via `python -m pytest tests`  
**Then** the test exercises:
- An integer key in `data` when `files` is set
- An integer key in `files`  
**And** the test does not open network connections  
**And** the test passes

**Verification:** Add `test_nonstring_multipart_field_names` in `tests/test_requests.py`. Run `python -m pytest tests/test_requests.py::test_nonstring_multipart_field_names -v` and confirm it passes.

---

### AC5: Existing tests pass

**Given** the existing test suite in `tests/`  
**When** `python -m pytest tests` runs  
**Then** all request-preparation tests pass  
**And** no regressions are introduced

**Verification:** Run `python -m pytest tests` and confirm zero failures in request-preparation tests (at minimum `tests/test_requests.py`, `tests/test_lowlevel.py`).

---

## Technical Specification

### Change Location

**File:** `src/requests/models.py`  
**Method:** `Request._encode_files`

### Required Implementation

In `Request._encode_files`, coerce **every multipart field name** to `str` before passing it to urllib3's multipart encoder. This ensures non-string keys (e.g., integers) do not cause `AttributeError` when urllib3 calls `.translate()` on field names.

**Implementation requirement:**
- When iterating over `data` items for multipart encoding, apply `str()` to each field name
- When iterating over `files` items for multipart encoding, apply `str()` to each field name
- An integer key like `1` must become the string field name `"1"` in the multipart body
- Do **not** modify urlencoded encoding — `data={1: "a"}` with no `files` must still produce body `1=a`
- Do **not** change file-tuple shapes or method signatures

**Pseudocode:**
```python
# In _encode_files, when building multipart fields:
for field_name, field_value in data.items():
    field_name = str(field_name)  # Coerce to string
    # ... rest of multipart encoding logic
    
for field_name, field_value in files.items():
    field_name = str(field_name)  # Coerce to string
    # ... rest of multipart encoding logic
```

### Test Requirements

**File:** `tests/test_requests.py`  
**Test name:** `test_nonstring_multipart_field_names`

**Test body:**
```python
def test_nonstring_multipart_field_names():
    """Non-string keys in data or files should be coerced to strings in multipart encoding."""
    
    # AC1: Integer key in data with files set
    req1 = requests.Request("POST", "http://example.com", data={1: "a"}, files={"f": b"x"})
    prepared1 = req1.prepare()
    assert prepared1 is not None  # No AttributeError raised
    assert b'name="1"' in prepared1.body  # Field name "1" appears in multipart body
    
    # AC2: Integer key in files
    req2 = requests.Request("POST", "http://example.com", files={1: b"x"})
    prepared2 = req2.prepare()
    assert prepared2 is not None  # No AttributeError raised
    assert b'name="1"' in prepared2.body  # Field name "1" appears in multipart body
    
    # AC3: Urlencoded behavior preserved (no files)
    req3 = requests.Request("POST", "http://example.com", data={1: "a"})
    prepared3 = req3.prepare()
    assert prepared3.body == b"1=a"  # Urlencoded body unchanged
```

**No network calls:** The test uses `Request(...).prepare()` directly, which does not open network connections.

### Scope

- **In scope:**
  - Modify `Request._encode_files` in `src/requests/models.py` to coerce field names to strings
  - Add `test_nonstring_multipart_field_names` in `tests/test_requests.py`
  - Verify AC1–AC5

- **Out of scope:**
  - Changes to urlencoded encoding logic
  - Changes to file-tuple parsing (e.g., `(filename, fileobj, content_type)` shapes)
  - Changes to public API signatures (`Request.__init__`, `request()`, `post()`, etc.)
  - Framework, ORM, or deployment considerations

### Verification Steps

1. **Unit test passes:**  
   ```bash
   python -m pytest tests/test_requests.py::test_nonstring_multipart_field_names -v
   ```
   Expected: `PASSED`

2. **Existing tests pass:**  
   ```bash
   python -m pytest tests
   ```
   Expected: No new failures in `tests/test_requests.py`, `tests/test_lowlevel.py`, or other request-preparation tests.

3. **Lint passes:**  
   ```bash
   python -m compileall -q .
   ```
   Expected: No syntax errors.

4. **Manual verification:**  
   ```python
   import requests
   
   # AC1 — data with files
   req1 = requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"})
   prepared1 = req1.prepare()
   print("AC1 passed — no AttributeError")
   
   # AC2 — files only
   req2 = requests.Request("POST", "http://x", files={1: b"x"})
   prepared2 = req2.prepare()
   print("AC2 passed — no AttributeError")
   
   # AC3 — urlencoded preserved
   req3 = requests.Request("POST", "http://x", data={1: "a"})
   prepared3 = req3.prepare()
   assert prepared3.body == b"1=a"
   print("AC3 passed — urlencoded body is '1=a'")
   ```
   Expected: All assertions pass, no exceptions.

---

## Dependencies

None.

---

## Open Questions

None.

---

## Implementation Notes

### Code Change Detail

The fix requires modifying `Request._encode_files` in `src/requests/models.py`. Locate the sections where the method iterates over `data` and `files` dictionaries to build multipart fields. Before passing field names to urllib3's encoder, wrap them with `str()`.

**Example locations to modify:**
- Any loop like `for k, v in data.items():` → change to use `str(k)` as the field name
- Any loop like `for k, v in files.items():` → change to use `str(k)` as the field name

The change is surgical: only the field name variable needs wrapping. Do not modify:
- The urlencoded path (which already handles non-string keys correctly)
- File value processing or tuple unpacking
- Any method signatures

### Test Implementation Detail

Add `test_nonstring_multipart_field_names` to `tests/test_requests.py`. The test must:
- Use `requests.Request(...).prepare()` (not `requests.post()` or any network call)
- Cover AC1 (int key in `data` when `files` is set)
- Cover AC2 (int key in `files`)
- Cover AC3 (urlencoded behavior unchanged)
- Assert no `AttributeError` is raised
- Assert the multipart body contains `name="1"` for the coerced field

### Why This Fix Works

urllib3's multipart encoder calls `.translate()` on field names, which exists on strings but not integers. By coercing field names to strings in `_encode_files`, we ensure urllib3 receives valid input. The urlencoded path is unaffected because it's a separate code path that already handles non-string keys (producing bodies like `1=a`).

---

**Critical: This story requires actual code changes.** The implementation must modify `src/requests/models.py` to coerce field names to strings in `_encode_files`, and add the test `test_nonstring_multipart_field_names` to `tests/test_requests.py`. A pull request with no file changes leaves the bug unfixed.
