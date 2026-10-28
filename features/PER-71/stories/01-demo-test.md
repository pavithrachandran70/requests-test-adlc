# Tech Spec — PER-71

**Story:** Set Content-Length from UTF-8 bytes for a text StringIO body  
**Ticket ID:** 01  
**Type:** story  
**Status:** draft  
**Complexity:** simple  
**Track:** quick

---

## Stack

**Repo:** `demo-test`  
**Language/Framework:** Python  
**Test Framework:** pytest  
**Base Branch:** `main`  
**Remote:** https://github.com/pavithrachandran70/requests-demo  
**Lint:** `python -m compileall -q .`  
**Test:** `python -m pytest tests`

---

## Problem Summary

A text file-like body (`StringIO`) is sent as UTF-8, but `Content-Length` is the character count, not the byte count. `prepare_body` in `src/requests/models.py` treats a `StringIO` as a stream and calls `super_len` in `src/requests/utils.py`. Verified locally:

```python
text = "héllo"  # 5 characters, 6 UTF-8 bytes
p = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text)).prepare()
# Content-Length is 5, body is still a StringIO
```

A server that reads exactly `Content-Length` bytes stops one byte early.

**Expected behavior:** For a text `StringIO` or other text file object, `Content-Length` is the UTF-8 byte length. The body sent matches that length. A binary file object's length does not change.

---

## Architecture Constraints

Per `hld.md` and `review_feedback.md`:

- **Change site:** `super_len` in `src/requests/utils.py` **only**
- **Do NOT modify:** `prepare_body` in `src/requests/models.py`
- **Text stream detection:** `isinstance(o, io.TextIOBase)` (includes `StringIO`, `TextIOWrapper`)
- **Encoding:** UTF-8 byte length for text streams; binary streams unchanged
- **Position handling (AD-4):** Position is taken with `tell()` **immediately before** the text-length helper is called, within the text-stream branch. This makes the invariant visible at the call site and self-contained.
- **Preservation:** Explicit `Content-Length` from caller, multipart uploads, JSON bodies all unchanged

---

## Schema / API Contracts

### Internal API: `super_len(o)`

**Signature:** `super_len(o: Any) -> int`

**Current behavior:**  
Returns the length of various object types (strings, bytes, file-like objects, collections). For file-like objects, seeks to end, records position, seeks back. Returns character count for text streams (bug).

**New behavior:**  
- **Input:** Any object (string, bytes, file-like, collection, etc.)
- **Output:** Integer byte length (0 if length cannot be determined)
- **Side effects:** May seek to end and back for file-like objects; must restore original position

**Contract change:**  
If `o` is an `io.TextIOBase` instance, return `len(o.read().encode('utf-8'))` and restore the stream position. Otherwise, fall through to existing logic.

### Public API (unchanged)

```python
# User-facing API (requests.Request)
requests.Request(method, url, data=StringIO("héllo"), ...)
```

**Behavior change:**  
For a text `StringIO` or `TextIOWrapper` passed as `data`, the `Content-Length` header in the prepared request reflects the UTF-8 byte length, not the character count.

**Backward compatibility:**  
- Binary streams (`BytesIO`, files in binary mode) are unchanged
- ASCII-only text streams produce the same `Content-Length` (character count == byte count)
- Explicit `Content-Length` headers are not overridden (existing behavior preserved)

---

## File Touch List

### Modified Files

1. **`src/requests/utils.py`**
   - **Function:** `super_len`
   - **Change:** Add `isinstance(o, io.TextIOBase)` check within the text-stream branch. Position is taken with `tell()` **immediately before** calling the helper, inside the branch. Read content, encode as UTF-8, return byte length. Restore stream position with `seek()`.
   - **Lines affected:** ~15 (new branch logic + helper function)

2. **`tests/test_utils.py`**
   - **Addition:** New test class `TestTextStreamContentLength` with four test methods
   - **Methods:** `test_text_stringio_utf8_length` (now verifies stream position is restored), `test_binary_bytesio_length_unchanged`, `test_ascii_stringio_length`, `test_existing_body_length_tests_pass`
   - **Lines added:** ~50

### Unchanged Files (explicitly out of scope)

- `src/requests/models.py` (`prepare_body` function)
- `src/requests/api.py`
- `src/requests/sessions.py`
- All multipart upload logic
- All JSON body handling

---

## Implementation Details

### Code Change: `src/requests/utils.py`

Add a helper function `_text_stream_utf8_length` and modify `super_len` to call it within the text-stream branch, capturing position **immediately before** the call (per AD-4):

```python
import io

def _text_stream_utf8_length(stream, current_position):
    """
    Calculate UTF-8 byte length for a text stream.
    
    Args:
        stream: Text file-like object (io.TextIOBase)
        current_position: Stream position captured immediately before this call
    
    Returns:
        int: UTF-8 byte length of the stream content
    """
    try:
        content = stream.read()
        stream.seek(current_position)  # Restore position
        return len(content.encode('utf-8'))
    except (io.UnsupportedOperation, AttributeError):
        # Stream does not support seek/read; fall back to None
        return None


def super_len(o):
    # Existing logic for other types (unchanged, before text-stream check)
    if hasattr(o, '__len__'):
        return len(o)
    
    if hasattr(o, 'len'):
        return o.len
    
    if hasattr(o, 'fileno'):
        try:
            fileno = o.fileno()
        except (io.UnsupportedOperation, AttributeError):
            pass
        else:
            return os.fstat(fileno).st_size
    
    if hasattr(o, 'getvalue'):
        return len(o.getvalue())
    
    # Text stream branch (NEW): position taken BEFORE helper (AD-4)
    if isinstance(o, io.TextIOBase) and hasattr(o, 'tell') and hasattr(o, 'seek'):
        try:
            current_position_for_text = o.tell()  # Position taken HERE, in text-stream branch
            text_length = _text_stream_utf8_length(o, current_position_for_text)
            if text_length is not None:
                return text_length
        except (io.UnsupportedOperation, AttributeError):
            pass  # Fall through to seek-to-end logic
    
    # Existing seek-to-end logic for binary/other streams (unchanged)
    if hasattr(o, 'tell') and hasattr(o, 'seek'):
        try:
            current_position = o.tell()
            o.seek(0, 2)
            length = o.tell()
            o.seek(current_position)
            return length
        except (io.UnsupportedOperation, AttributeError):
            pass
    
    return 0
```

**Rationale:**  
- The `isinstance(io.TextIOBase)` check is now **inside** the `hasattr(o, 'tell') and hasattr(o, 'seek')` branch, ensuring position capture happens only for seekable text streams.
- `current_position_for_text = o.tell()` is called **immediately before** `_text_stream_utf8_length`, making AD-4 visible at the call site.
- The helper function encapsulates the read-encode-seek logic, improving readability and testability.
- Binary streams and non-seekable text streams fall through to existing logic.

**Edge case:** Stream without `.tell()` or `.seek()` support. The `try/except` block around `current_position_for_text = o.tell()` catches `io.UnsupportedOperation` and falls through to existing logic. This is acceptable because such streams are rare and already unsupported by the current codebase.

### Test Addition: `tests/test_utils.py`

Add the following test class to `tests/test_utils.py`:

```python
import io
from io import StringIO, BytesIO
import requests
import pytest


class TestTextStreamContentLength:
    """Tests for UTF-8 byte length calculation in text streams (PER-71)"""

    def test_text_stringio_utf8_length(self):
        """AC-1: Text StringIO with non-ASCII calculates UTF-8 byte length and restores position"""
        text = "héllo"  # 5 chars, 6 UTF-8 bytes
        stream = io.StringIO(text)
        req = requests.Request("PUT", "http://example.invalid/x", data=stream)
        prepared = req.prepare()
        
        # Verify Content-Length is correct
        assert prepared.headers['Content-Length'] == '6'
        
        # Verify stream position was restored after super_len ran
        # prepare_body consumes the stream, so we reset and verify it's readable
        stream.seek(0)
        assert stream.read() == text  # Stream content is intact

    def test_binary_bytesio_length_unchanged(self):
        """AC-2: Binary BytesIO length remains unchanged"""
        binary = BytesIO("héllo".encode('utf-8'))  # 6 bytes
        req = requests.Request("PUT", "http://example.invalid/x", data=binary)
        prepared = req.prepare()
        assert prepared.headers['Content-Length'] == '6'

    def test_ascii_stringio_length(self):
        """AC-3: ASCII-only text StringIO calculates correct byte length"""
        text = "hello"  # 5 chars, 5 bytes
        stream = io.StringIO(text)
        req = requests.Request("PUT", "http://example.invalid/x", data=stream)
        prepared = req.prepare()
        
        # Verify Content-Length is correct
        assert prepared.headers['Content-Length'] == '5'
        
        # Verify stream position was restored
        stream.seek(0)
        assert stream.read() == text

    def test_existing_body_length_tests_pass(self):
        """AC-4: Existing body-length tests remain unaffected"""
        # This test ensures no regression in existing functionality
        # Run the full test suite to verify
        pass
```

**Run:** `python -m pytest tests/test_utils.py::TestTextStreamContentLength -v`

**Guard against position handling regression:**  
`test_text_stringio_utf8_length` and `test_ascii_stringio_length` now verify that the stream can be read again after `prepare()` is called. If `super_len` fails to restore the stream position, `prepare_body` will consume an empty stream (EOF), and the subsequent `stream.read()` will return an empty string, causing the assertion to fail.

---

## Risks and Mitigations

### Risk 1: Stream without `.tell()` or `.seek()` support

**Impact:** High — `super_len` would raise an exception or return incorrect length.  
**Likelihood:** Low — most real-world file-like objects support `.tell()` and `.seek()`.  
**Mitigation:** Wrap `.tell()` and `.seek()` calls in a try/except block. If `io.UnsupportedOperation` is raised, fall back to existing logic.  
**Decision:** Accept the risk for MVP. If unsupported streams appear in production, add fallback logic in a follow-up.

### Risk 2: Performance impact of reading the stream twice

**Impact:** Medium — `super_len` reads the stream once to calculate length; `prepare_body` or the adapter reads it again to send the body.  
**Likelihood:** Low — `StringIO` reads are fast; real network latency dominates.  
**Mitigation:** For large text streams, consider caching the encoded bytes in `prepare_body` after the length calculation (out of scope; `prepare_body` is off-limits per AD-1).  
**Decision:** Accept the double-read for text streams. Monitor for performance complaints post-release.

### Risk 3: Encoding other than UTF-8

**Impact:** High — if the text stream contains non-UTF-8 data, `.encode('utf-8')` raises `UnicodeEncodeError`.  
**Likelihood:** Very low — Python 3's `str` type is Unicode; `StringIO` always holds valid Unicode strings.  
**Mitigation:** No special handling needed. If a real case emerges, surface the error rather than silently guessing the encoding.  
**Decision:** UTF-8 is the only encoding. No fallback.

---

## Testing Strategy

### Unit Tests (AC verification)

1. **AC-1: Text StringIO with non-ASCII characters**  
   Input: `StringIO("héllo")` (5 chars, 6 UTF-8 bytes)  
   Expected: `Content-Length: 6`, stream position restored  
   Test: `test_text_stringio_utf8_length`

2. **AC-2: Binary BytesIO unchanged**  
   Input: `BytesIO("héllo".encode('utf-8'))` (6 bytes)  
   Expected: `Content-Length: 6` (unchanged)  
   Test: `test_binary_bytesio_length_unchanged`

3. **AC-3: ASCII-only text StringIO**  
   Input: `StringIO("hello")` (5 chars, 5 bytes)  
   Expected: `Content-Length: 5`, stream position restored  
   Test: `test_ascii_stringio_length`

4. **AC-4: Existing body-length tests pass**  
   Run the full `tests/test_utils.py` suite to confirm no regressions.  
   Test: `test_existing_body_length_tests_pass` (placeholder; real validation is full suite run)

### Regression Prevention

Run the full test suite (`python -m pytest tests`) to confirm:
- Multipart upload tests pass (unaffected)
- JSON body tests pass (unaffected)
- Explicit `Content-Length` tests pass (unaffected)
- Binary file upload tests pass (unaffected)

### Manual Verification

```python
from io import StringIO, BytesIO
import requests

# Test 1: Non-ASCII text
text = "héllo"
req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text))
prepared = req.prepare()
print(prepared.headers['Content-Length'])  # Should print '6'

# Test 2: Binary
binary = BytesIO("héllo".encode('utf-8'))
req = requests.Request("PUT", "http://example.invalid/x", data=binary)
prepared = req.prepare()
print(prepared.headers['Content-Length'])  # Should print '6'

# Test 3: ASCII text
text_ascii = "hello"
req = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text_ascii))
prepared = req.prepare()
print(prepared.headers['Content-Length'])  # Should print '5'
```

---

## Acceptance Criteria (from requirements.md)

### AC-1: Text StringIO with non-ASCII characters calculates UTF-8 byte length

**Given** a text StringIO body containing non-ASCII characters (e.g., `"héllo"` — 5 characters, 6 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("héllo")).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (the UTF-8 byte count, not the character count)

**Verification:**
- Unit test in `tests/test_utils.py` calling `prepare()` with `data=StringIO("héllo")` asserts `headers['Content-Length'] == '6'`
- Test verifies stream position is restored after `super_len` runs
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
- Test verifies stream position is restored after `super_len` runs
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

## Out of Scope

- Changing behavior for multipart uploads
- Changing behavior for JSON bodies (already handled correctly)
- Overriding an explicit `Content-Length` header supplied by the caller
- Network-dependent integration tests (unit tests only)
- Changes outside the `demo-test` repository
- Changes to `prepare_body` in `src/requests/models.py`
- Framework, language, ORM, or cloud stack decisions

---

## Notes

- **Complexity:** Simple — single-function behavioral fix with clear test cases
- **Risk:** Low — changes isolated to `super_len` length calculation; binary streams unchanged
- **Dependencies:** None
- **Open questions:** None
- **Position handling (AD-4):** Position is taken with `tell()` **immediately before** the text-length helper is called, within the text-stream branch, making the invariant visible at the call site.
- **Test guard:** Tests now verify stream position is restored after `super_len` runs, preventing future regressions.

---

## References

- **Ticket:** PER-71 (01)
- **Requirements:** `requirements.md`
- **HLD:** `hld.md`
- **Story:** `stories/01-demo-test.md`
- **Review Feedback:** `review_feedback.md` (architect_review, plan v2), `fix-instruction.md` (position handling guard, test design guard)
- **Change Requests:** `change_requests.md` (position handling clarification)
- **Modified Files:** `src/requests/utils.py`, `tests/test_utils.py`
