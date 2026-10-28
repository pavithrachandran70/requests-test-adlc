# Requirements — PER-77

## Story

As a developer using the requests library,  
I want to pass non-string keys in `data` or `files` when preparing multipart requests,  
so that I don't encounter an `AttributeError` when urllib3 attempts to encode field names.

## Context

Currently, `Request._encode_files` accepts non-string keys in `data` and `files` but does not coerce field names to strings before passing them to urllib3's multipart encoder. urllib3 expects string field names and calls `.translate()` on them, causing an `AttributeError` when an integer key is used. Urlencoded requests already handle non-string keys correctly (producing bodies like `1=a`), so multipart requests should behave consistently.

## Acceptance Criteria

### AC1: Integer keys in `data` succeed with multipart encoding

**Given** a POST request with `data={1: "a"}` and `files={"f": b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully without raising `AttributeError`  
**And** the multipart body includes the field name `"1"` with value `"a"`

### AC2: Integer keys in `files` succeed

**Given** a POST request with `files={1: b"x"}`  
**When** `.prepare()` is called  
**Then** the request prepares successfully without raising `AttributeError`  
**And** the multipart body includes the field name `"1"`

### AC3: Urlencoded behavior is preserved

**Given** a POST request with `data={1: "a"}` and no `files` parameter  
**When** `.prepare()` is called  
**Then** the request body is `1=a` (urlencoded format)  
**And** the behavior is identical to the current implementation

### AC4: Test coverage

**Given** a new unit test for non-string field names  
**When** the test runs via `python -m pytest tests`  
**Then** the test exercises:
- An integer key in `data` when `files` is set
- An integer key in `files`  
**And** the test does not open network connections  
**And** the test passes

### AC5: Existing tests pass

**Given** the existing test suite in `tests/`  
**When** `python -m pytest tests` runs  
**Then** all request-preparation tests pass  
**And** no regressions are introduced

## Technical Notes

### Scope

- **Change location:** `Request._encode_files` method in `src/requests/models.py`
- **Change:** Coerce field names to strings (e.g., via `str()`) before passing to urllib3's multipart encoder
- **Preserve:** Urlencoded encoding logic, file-tuple shapes, public method signatures

### Test Requirements

- Add test in `tests/test_requests.py` or a new test file under `tests/`
- Use `Request(...).prepare()` directly; no `requests.post()` or network calls
- Cover both `data` and `files` with integer keys
- Verify no `AttributeError` is raised
- Verify urlencoded path still produces `1=a` body

### Out of Scope

- Changes to urlencoded encoding
- Changes to file-tuple parsing (e.g., `(filename, fileobj, content_type)` tuples)
- Changes to public API signatures
- Framework, ORM, or deployment considerations

## Verification

1. Run `python -m pytest tests/test_requests.py -k <new_test_name>` — new test passes
2. Run `python -m pytest tests` — all existing tests pass
3. Run `python -m compileall -q .` — no syntax errors
4. Manual check: `requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()` succeeds

## Dependencies

None.

## Open Questions

None.
