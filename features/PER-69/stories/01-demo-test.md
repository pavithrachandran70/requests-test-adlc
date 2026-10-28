# Story — Coerce non-string multipart field names instead of raising AttributeError

**ID:** PER-69  
**Type:** story  
**Status:** draft  
**Epic:** (none — backlog item)  
**Assignee:** (unassigned)  
**Complexity:** simple  
**Estimate:** 2 points

---

## Description

**As a** developer using the requests library  
**I want** non-string keys in `data` and `files` to be handled consistently  
**So that** multipart requests behave the same as urlencoded requests when field names are not strings

### Problem

A non-string key in `data` is accepted for a urlencoded body, but the same key fails once `files` is set and the body becomes multipart. A non-string key in `files` fails the same way. Verified locally:

- `Request("POST", "http://x", data={1: "a"}).prepare()` succeeds and the body is `1=a`.
- `Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()` raises `AttributeError: 'int' object has no attribute 'translate'`.
- `Request("POST", "http://x", files={1: b"x"}).prepare()` raises the same `AttributeError`.

`Request._encode_files` in `src/requests/models.py` converts values with `str()`, then passes the field name through unchanged at `RequestField(name=k, ...)`.

### Expected Behavior

Both `prepare()` calls succeed. The multipart body uses the string `"1"` as the field name, matching the urlencoded path. No `AttributeError`.

### Scope

Repo `requests` only. Change field-name handling in `_encode_files`. Add a unit test that calls `prepare()` and does not open a network connection. Do not change urlencoded encoding, file-tuple shapes, or public method signatures.

---

## Acceptance Criteria

### AC1: Non-string key in `data` with `files` present
**Given** a POST request with a non-string key (e.g., integer `1`) in `data` and at least one entry in `files`  
**When** `prepare()` is called  
**Then** the multipart body is built successfully using the stringified field name `"1"` and no `AttributeError` is raised

### AC2: Non-string key in `files`
**Given** a POST request with a non-string key (e.g., integer `1`) in `files`  
**When** `prepare()` is called  
**Then** the multipart body is built successfully using the stringified field name `"1"` and no `AttributeError` is raised

### AC3: Behavior parity with urlencoded encoding
**Given** the same non-string key (e.g., integer `1`) is used in `data` for both a urlencoded request (no `files`) and a multipart request (with `files`)  
**When** both requests call `prepare()`  
**Then** both succeed and use the string `"1"` as the field name in their respective body encodings

---

## Technical Specification

### File to Change

**Path:** `src/requests/models.py`  
**Method:** `Request._encode_files`

### Current Behavior

The method iterates over `data` and `files` dictionaries and passes field names (`k`) directly to `RequestField(name=k, ...)`. When `k` is not a string (e.g., an integer), `RequestField` or a downstream component calls `.translate()` on it, raising `AttributeError`.

### Required Change

**Coercion point:** Before constructing `RequestField`, coerce the field name to a string:

```python
# In _encode_files, at the point where field name is used:
# Current (problematic):
# rf = RequestField(name=k, ...)

# Fixed:
rf = RequestField(name=str(k), ...)
```

Apply this coercion in **both** the `data` iteration block and the `files` iteration block within `_encode_files`.

### Constraints

- Do **not** change urlencoded encoding logic (it already handles non-string keys correctly).
- Do **not** modify file-tuple shapes or public method signatures.
- Do **not** introduce dependencies or external utilities.

---

## Tests

### Unit Test: `test_nonstring_field_names_in_multipart`

**Location:** `tests/test_requests.py` (or create `tests/test_multipart_fields.py` if preferred)

**Setup:** No network; calls `Request(...).prepare()` directly.

**Test Cases:**

1. **Non-string key in `data` with `files` present**  
   ```python
   r = Request("POST", "http://example.com", data={1: "a"}, files={"f": b"x"})
   prepared = r.prepare()
   # Assert: no AttributeError raised
   # Assert: prepared.body contains field name "1" (inspect multipart boundary-separated body)
   ```

2. **Non-string key in `files`**  
   ```python
   r = Request("POST", "http://example.com", files={1: b"x"})
   prepared = r.prepare()
   # Assert: no AttributeError raised
   # Assert: prepared.body contains field name "1"
   ```

3. **Baseline: urlencoded with non-string key (no `files`)**  
   ```python
   r = Request("POST", "http://example.com", data={1: "a"})
   prepared = r.prepare()
   # Assert: prepared.body == b"1=a" (already passing; confirms parity)
   ```

**Assertions:**

- No `AttributeError` is raised during `prepare()`.
- Multipart body contains the stringified field name `"1"` (parsed from the `Content-Disposition` header in the multipart body).
- Urlencoded body contains `1=a` (baseline).

---

## Implementation Notes

- The fix is a **one-line change per iteration block** in `_encode_files`: wrap the field name with `str()` before passing it to `RequestField`.
- The unit test exercises **both `data` and `files`** with non-string keys, ensuring parity with urlencoded behavior.
- No refactoring, no new modules, no config changes.

---

## Out of Scope

- Changes to urlencoded encoding behavior (already correct).
- Changes to file-tuple structure (e.g., `(filename, fileobj, content_type, headers)`).
- Public API signature changes (e.g., `Request.__init__`, `Session.post`).
- Integration tests that open network connections.

---

## Verification

**How to verify AC1, AC2, AC3:**

Run the unit test `test_nonstring_field_names_in_multipart`. All three cases must pass without `AttributeError`. Inspect the prepared body to confirm field names are strings.

**Regression check:**

Run the full test suite (`pytest tests/`) to confirm no existing behavior is broken.

---

## Dependencies

None.

---

## Notes

- **Planner focus delivered:** The spec identifies the exact coercion point (`str(k)` before `RequestField(name=k, ...)`), confirms parity with urlencoded behavior (both use stringified field names), and specifies the unit test structure (three cases exercising `data`, `files`, and baseline urlencoded).
- **File paths match repo tree:** `src/requests/models.py` (exists), `tests/test_requests.py` (exists).
- **AC count matches ticket scope:** Three ACs cover the three failure scenarios described in the ticket (data+files, files-only, baseline).
