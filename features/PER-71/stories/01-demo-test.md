# Story — Set Content-Length from UTF-8 bytes for a text StringIO body

**ID:** PER-71  
**Type:** story  
**Status:** draft  
**Epic:** _(none — backlog story)_  
**Assignee:** _(unassigned)_  
**Points:** 2  
**Track:** quick

---

## Problem

A text file-like body is sent as UTF-8, but `Content-Length` is the character count. `prepare_body` in `src/requests/models.py` treats a `StringIO` as a stream and calls `super_len` in `src/requests/utils.py`. Verified locally:

```python
text = "héllo"  # 5 characters, 6 UTF-8 bytes
p = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text)).prepare()
# Content-Length is 5, body is still a StringIO
```

A server that reads exactly `Content-Length` bytes stops one byte early.

---

## Expected Behavior

For a text `StringIO` or other text file object, `Content-Length` is the UTF-8 byte length. The body sent matches that length. A binary file object's length does not change.

---

## Scope

- **Repo:** `demo-test` only
- **Focus:** UTF-8 encoding edge case in `super_len` (`src/requests/utils.py`) or `prepare_body` (`src/requests/models.py`)
- **Constraint:** Binary streams must not change behavior
- Fix length calculation for text streams in `super_len` or `prepare_body`
- Add a unit test that calls `prepare()` and does not open a network connection
- Do not change multipart uploads, JSON bodies, or an explicit `Content-Length` supplied by the caller

---

## Acceptance Criteria

### AC-1: Text StringIO with non-ASCII characters calculates UTF-8 byte length

**Given** a text StringIO body containing non-ASCII characters (e.g., `"héllo"` — 5 characters, 6 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("héllo")).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (the UTF-8 byte count, not the character count)

**Verification:**
- Unit test in `tests/test_requests.py` or `tests/test_utils.py` calling `prepare()` with `data=StringIO("héllo")` asserts `headers['Content-Length'] == '6'`
- Test runs without opening a network connection

---

### AC-2: Binary BytesIO length remains unchanged

**Given** a binary BytesIO body containing UTF-8 encoded bytes (e.g., `BytesIO("héllo".encode())` — 6 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=BytesIO("héllo".encode())).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (unchanged behavior for binary streams)

**Verification:**
- Unit test in `tests/test_requests.py` or `tests/test_utils.py` calling `prepare()` with `data=BytesIO("héllo".encode())` asserts `headers['Content-Length'] == '6'`
- Test confirms binary stream handling remains unaffected

---

### AC-3: ASCII-only text StringIO calculates correct byte length

**Given** a text StringIO body containing only ASCII characters (e.g., `"hello"` — 5 characters, 5 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("hello")).prepare()` is called  
**Then** the prepared request has `Content-Length: 5` (character count equals byte count for ASCII)

**Verification:**
- Unit test in `tests/test_requests.py` or `tests/test_utils.py` calling `prepare()` with `data=StringIO("hello")` asserts `headers['Content-Length'] == '5'`
- Test confirms no regression for ASCII-only content

---

### AC-4: Test suite passes for body-length tests

**Given** the fix is implemented in `super_len` (`src/requests/utils.py`) or `prepare_body` (`src/requests/models.py`)  
**When** `python -m pytest tests` is executed  
**Then** all body-length tests pass without failures

**Verification:**
- CI/local run of `python -m pytest tests` shows 0 failures for new and existing body-length tests
- Tests for multipart uploads, JSON bodies, and explicit `Content-Length` headers remain unaffected

---

## Technical Spec

### Implementation Approach

The fix targets the UTF-8 encoding edge case in either:

1. **`super_len` in `src/requests/utils.py`** — Detect text file-like objects (e.g., `StringIO`, `TextIOWrapper`) and encode their content as UTF-8 to compute byte length, or
2. **`prepare_body` in `src/requests/models.py`** — Before calling `super_len`, check if the body is a text stream and encode it to bytes for length calculation

**Preferred location:** `super_len` in `src/requests/utils.py`, as it centralizes length logic.

### Detection Strategy

- Use `hasattr(body, 'mode')` and `'b' not in body.mode` to detect text mode file objects
- Use `isinstance(body, io.StringIO)` for `StringIO` specifically
- Fallback: if the body has a `read()` method and no `mode`, attempt `.read().encode('utf-8')` to test if it is text

### Encoding Logic

For text streams:
```python
if is_text_stream(body):
    # Read content, encode as UTF-8, return byte length
    content = body.read()
    body.seek(0)  # Reset stream position
    return len(content.encode('utf-8'))
```

For binary streams (existing behavior):
```python
# Existing logic unchanged
```

### Files to Modify

- **`src/requests/utils.py`** — Update `super_len` function to handle text streams
- **`tests/test_utils.py`** or **`tests/test_requests.py`** — Add unit tests for AC-1, AC-2, AC-3

### Test Structure

```python
from io import StringIO, BytesIO
import requests

def test_text_stringio_utf8_length():
    """AC-1: Text StringIO with non-ASCII calculates UTF-8 byte length"""
    text = "héllo"  # 5 chars, 6 UTF-8 bytes
    req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text))
    prepared = req.prepare()
    assert prepared.headers['Content-Length'] == '6'

def test_binary_bytesio_length_unchanged():
    """AC-2: Binary BytesIO length remains unchanged"""
    binary = BytesIO("héllo".encode('utf-8'))  # 6 bytes
    req = requests.Request("PUT", "http://example.invalid/x", data=binary)
    prepared = req.prepare()
    assert prepared.headers['Content-Length'] == '6'

def test_ascii_stringio_length():
    """AC-3: ASCII-only text StringIO calculates correct byte length"""
    text = "hello"  # 5 chars, 5 bytes
    req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text))
    prepared = req.prepare()
    assert prepared.headers['Content-Length'] == '5'
```

### Edge Cases

- Stream already at EOF: Reset with `.seek(0)` after reading
- Stream without `.seek()`: Read once, cache length, and replace body with `BytesIO` of encoded content
- Explicit `Content-Length` header: Do not override (existing behavior preserved)

---

## Out of Scope

- Changing behavior for multipart uploads
- Changing behavior for JSON bodies (already handled correctly)
- Overriding an explicit `Content-Length` header supplied by the caller
- Network-dependent integration tests (unit tests only)
- Changes outside the `demo-test` repository

---

## Notes

- **Complexity:** Simple — single-repo behavioral fix with clear test cases
- **Risk:** Low — changes isolated to length calculation; binary streams unchanged
- **Dependencies:** None
- **Open questions:** None

---

## References

- Epic: _(none — backlog story)_
- Ticket: PER-71
- Related files: `src/requests/utils.py`, `src/requests/models.py`, `tests/test_utils.py`, `tests/test_requests.py`
