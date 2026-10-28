# Requirements — PER-69

## Story

**As a** developer using the requests library  
**I want** non-string keys in `data` and `files` to be handled consistently  
**So that** multipart requests behave the same as urlencoded requests when field names are not strings

---

## Given/When/Then Acceptance Criteria

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

## Tests

### Unit Test: `test_nonstring_field_names_in_multipart`
- **Scope:** Unit test in `tests/test_requests.py` or equivalent test module
- **Setup:** No network; calls `Request(...).prepare()` directly
- **Cases:**
  1. `data={1: "a"}`, `files={"f": b"x"}` — verify `prepare()` succeeds and multipart body contains field name `"1"`
  2. `files={1: b"x"}` — verify `prepare()` succeeds and multipart body contains field name `"1"`
  3. `data={1: "a"}` (no files) — verify urlencoded body is `1=a` (baseline; may already pass)
- **Assertions:** No `AttributeError`; field names in prepared body are strings

---

## Implementation Notes

- **File:** `src/requests/models.py`, method `Request._encode_files`
- **Change:** Coerce field name `k` to string before passing to `RequestField(name=k, ...)`
- **Constraint:** Do not change urlencoded encoding, file-tuple shapes, or public method signatures

---

## Out of Scope

- Framework, language, ORM, or cloud stack choices
- Changes to urlencoded encoding behavior
- Changes to file-tuple structure or public API signatures
- Network-based integration tests
