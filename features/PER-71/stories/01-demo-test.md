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
- **Change site:** `super_len` in `src/requests/utils.py` **only**
- **Constraint:** Do **not** change `prepare_body` in `src/requests/models.py`
- Fix length calculation for text streams in `super_len`
- Add unit tests in `tests/test_utils.py` that call `prepare()` and do not open a network connection
- Do not change multipart uploads, JSON bodies, or an explicit `Content-Length` supplied by the caller

---

## Acceptance Criteria

### AC-1: Text StringIO with non-ASCII characters calculates UTF-8 byte length

**Given** a text StringIO body containing non-ASCII characters (e.g., `"héllo"` — 5 characters, 6 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("héllo")).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (the UTF-8 byte count, not the character count)

**Verification:**
- Unit test in `tests/test_utils.py` calling `prepare()` with `data=StringIO("héllo")` asserts `headers['Content-Length'] == '6'`
- Test runs without opening a network connection

---

### AC-2: Binary BytesIO length remains unchanged

**Given** a binary BytesIO body containing UTF-8 encoded bytes (e.g., `BytesIO("héllo".encode())` — 6 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=BytesIO("héllo".encode())).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (unchanged behavior for binary streams)

**Verification:**
- Unit test in `tests/test_utils.py` calling `prepare()` with `data=BytesIO("héllo".encode())` asserts `headers['Content-Length'] == '6'`
- Test confirms binary stream handling remains unaffected

---

### AC-3: ASCII-only text StringIO calculates correct byte length

**Given** a text StringIO body containing only ASCII characters (e.g., `"hello"` — 5 characters, 5 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("hello")).prepare()` is called  
**Then** the prepared request has `Content-Length: 5` (character count equals byte count for ASCII)

**Verification:**
- Unit test in `tests/test_utils.py` calling `prepare()` with `data=StringIO("hello")` asserts `headers['Content-Length'] == '5'`
- Test confirms no regression for ASCII-only content

---

### AC-4: Test suite passes for body-length tests

**Given** the fix is implemented in `super_len` in `src/requests/utils.py`  
**When** `python -m pytest tests` is executed  
**Then** all body-length tests pass without failures

**Verification:**
- CI/local run of `python -m pytest tests` shows 0 failures for new and existing body-length tests
- Tests for multipart uploads, JSON bodies, and explicit `Content-Length` headers remain unaffected

---

## Technical Spec

### Implementation Approach

The fix targets `super_len` in `src/requests/utils.py` **only**. Do **not** modify `prepare_body` in `src/requests/models.py`.

When the stream is a text file object (`io.TextIOBase`, including `StringIO`), `Content-Length` must be the UTF-8 byte length, not the character count. Binary `BytesIO` must keep its current byte length. An explicit `Content-Length` from the caller must not be overwritten. Multipart and JSON bodies stay unchanged.

### Detection Strategy

Detect text file-like objects in `super_len`:

- Use `isinstance(body, io.TextIOBase)` to detect text streams (includes `StringIO`, `TextIOWrapper`)
- For objects without a clear type hint, check `hasattr(body, 'mode')` and `'b' not in body.mode`
- Fallback: if the body has a `read()` method and returns `str` (not `bytes`), treat as text

### Encoding Logic

For text streams in `super_len`:

```python
import io

def super_len(o):
    # ... existing logic for other types ...
    
    # Check if o is a text stream
    if isinstance(o, io.TextIOBase):
        # Read content, encode as UTF-8, return byte length
        current_position = o.tell()
        content = o.read()
        o.seek(current_position)  # Reset stream position
        return len(content.encode('utf-8'))
    
    # ... existing file-like object handling for binary streams ...
```

For binary streams (existing behavior unchanged):

```python
# Existing logic for BytesIO, file objects in binary mode, etc.
```

### Files to Modify

- **`src/requests/utils.py`** — Update `super_len` function to handle text streams
- **`tests/test_utils.py`** — Add unit tests for AC-1, AC-2, AC-3, AC-4

### Test Structure

Add to `tests/test_utils.py`:

```python
from io import StringIO, BytesIO
import requests
import pytest


class TestTextStreamContentLength:
    """Tests for UTF-8 byte length calculation in text streams (AC-1, AC-2, AC-3)"""

    def test_text_stringio_utf8_length(self):
        """AC-1: Text StringIO with non-ASCII calculates UTF-8 byte length"""
        text = "héllo"  # 5 chars, 6 UTF-8 bytes
        req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text))
        prepared = req.prepare()
        assert prepared.headers['Content-Length'] == '6'

    def test_binary_bytesio_length_unchanged(self):
        """AC-2: Binary BytesIO length remains unchanged"""
        binary = BytesIO("héllo".encode('utf-8'))  # 6 bytes
        req = requests.Request("PUT", "http://example.invalid/x", data=binary)
        prepared = req.prepare()
        assert prepared.headers['Content-Length'] == '6'

    def test_ascii_stringio_length(self):
        """AC-3: ASCII-only text StringIO calculates correct byte length"""
        text = "hello"  # 5 chars, 5 bytes
        req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text))
        prepared = req.prepare()
        assert prepared.headers['Content-Length'] == '5'

    def test_existing_body_length_tests_pass(self):
        """AC-4: Existing body-length tests remain unaffected"""
        # This test ensures no regression in existing functionality
        # Specific assertions depend on existing test suite structure
        pass
```

Run with: `python -m pytest tests/test_utils.py::TestTextStreamContentLength -v`

### Edge Cases

- **Stream already at EOF:** Reset with `.seek(current_position)` after reading
- **Stream without `.tell()` or `.seek()`:** Read once, cache length, replace body with `BytesIO` of encoded content
- **Explicit `Content-Length` header:** Do not override (existing behavior preserved in `prepare_body`)
- **Multipart uploads:** Not affected (handled separately in `prepare_body`)
- **JSON bodies:** Not affected (already encoded as bytes before reaching `super_len`)

---

## Out of Scope

- Changing behavior for multipart uploads
- Changing behavior for JSON bodies (already handled correctly)
- Overriding an explicit `Content-Length` header supplied by the caller
- Network-dependent integration tests (unit tests only)
- Changes outside the `demo-test` repository
- Changes to `prepare_body` in `src/requests/models.py`

---

## Notes

- **Complexity:** Simple — single-function behavioral fix with clear test cases
- **Risk:** Low — changes isolated to `super_len` length calculation; binary streams unchanged
- **Dependencies:** None
- **Open questions:** None

---

## References

- Epic: _(none — backlog story)_
- Ticket: PER-71
- Requirements: `requirements.md` (updated with design constraint)
- Review feedback: `review_feedback.md` (architect_review, plan v2)
- Related files: `src/requests/utils.py`, `tests/test_utils.py`
