# Requirements — PER-71

**Story:** Set Content-Length from UTF-8 bytes for a text StringIO body

**Source:** Ticket PER-71, review feedback (architect_review)

---

## Given/When/Then Acceptance Criteria

### AC-1: Text StringIO with non-ASCII characters calculates UTF-8 byte length

**Given** a text StringIO body containing non-ASCII characters (e.g., `"héllo"` — 5 characters, 6 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("héllo")).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (the UTF-8 byte count, not the character count)

**Verification:**
- Unit test calling `prepare()` with `data=StringIO("héllo")` asserts `headers['Content-Length'] == '6'`
- Test runs without opening a network connection

---

### AC-2: Binary BytesIO length remains unchanged

**Given** a binary BytesIO body containing UTF-8 encoded bytes (e.g., `BytesIO("héllo".encode())` — 6 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=BytesIO("héllo".encode())).prepare()` is called  
**Then** the prepared request has `Content-Length: 6` (unchanged behavior for binary streams)

**Verification:**
- Unit test calling `prepare()` with `data=BytesIO("héllo".encode())` asserts `headers['Content-Length'] == '6'`
- Test confirms binary stream handling remains unaffected

---

### AC-3: ASCII-only text StringIO calculates correct byte length

**Given** a text StringIO body containing only ASCII characters (e.g., `"hello"` — 5 characters, 5 UTF-8 bytes)  
**When** `requests.Request("PUT", "http://example.invalid/x", data=StringIO("hello")).prepare()` is called  
**Then** the prepared request has `Content-Length: 5` (character count equals byte count for ASCII)

**Verification:**
- Unit test calling `prepare()` with `data=StringIO("hello")` asserts `headers['Content-Length'] == '5'`
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

## Design Constraint

**Change site:** `super_len` in `src/requests/utils.py` **only**. Do **not** change `prepare_body` in `src/requests/models.py`.

When the stream is a text file object (`io.TextIOBase`, including `StringIO`), `Content-Length` must be the UTF-8 byte length, not the character count. Binary `BytesIO` must keep its current byte length. An explicit `Content-Length` from the caller must not be overwritten. Multipart and JSON bodies stay unchanged.

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

## Technical Notes

- **Focus area:** UTF-8 encoding edge case in `super_len` (`src/requests/utils.py`)
- **Constraint:** Binary streams must not change behavior; `prepare_body` in `src/requests/models.py` must not be modified
- **Test isolation:** All verification uses `prepare()` and does not open network connections
- **File scope:** `src/requests/utils.py`, plus test file in `tests/test_utils.py` or `tests/test_requests.py`
