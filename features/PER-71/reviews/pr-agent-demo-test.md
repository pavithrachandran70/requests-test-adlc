```json
[
  {
    "path": "src/requests/utils.py",
    "line": 184,
    "body": "**AC gap: `_text_stream_utf8_length` does not restore position correctly when `current_position` is non-zero.**\n\nThe function calls `o.seek(current_position or 0)`, which means if `current_position` is `0`, it restores to position `0`. However, `current_position` is passed from the caller (`super_len`), which retrieves it via `o.tell()` before any read. If the stream was already partially consumed (e.g., `tell()` returns `3`), the function should restore to position `3`, not `0`.\n\nThe `or 0` short-circuit is incorrect. When `current_position` is `0`, `0 or 0` evaluates to `0` (correct), but the logic reads as if it's guarding against `None`, which is not the intent here.\n\n**Fix:** Change `o.seek(current_position or 0)` to `o.seek(current_position)`. The caller (`super_len`) ensures `current_position` is always an integer (line 244: `current_position = o.tell()`).\n\n**Consequence if shipped:** A stream that was already partially read would be reset to position `0` rather than its original position, causing `prepare_body` to re-read from the start and produce incorrect behavior downstream."
  },
  {
    "path": "src/requests/utils.py",
    "line": 253,
    "body": "**Bug: `_text_stream_utf8_length` is called before `current_position` is set, causing it to receive `0` instead of the stream's actual position.**\n\nLine 253 calls `_text_stream_utf8_length(o, current_position)`, but `current_position` is still `0` (initialized at line 248). The correct position is retrieved at line 264 via `current_position = o.tell()`, which happens *after* the text-stream check.\n\n**Expected order:**\n1. Retrieve `current_position = o.tell()` first.\n2. Call `_text_stream_utf8_length(o, current_position)`.\n\n**Fix:** Move `current_position = o.tell()` to before the `_text_stream_utf8_length` call, or pass `o.tell()` directly as the second argument.\n\n**Consequence if shipped:** Text streams that are already partially consumed will have their position incorrectly restored to `0`, breaking AC-1 for streams where `.tell() != 0`."
  },
  {
    "path": "src/requests/utils.py",
    "line": 189,
    "body": "**Unnecessary conditional: `if isinstance(content, str)` is redundant.**\n\nThe function `_text_stream_utf8_length` only runs when `_is_text_stream(o)` returns `True` (line 187). A text stream's `.read()` always returns `str` by definition (that's what makes it a text stream). The `isinstance(content, str)` check at line 189 is defensive but logically unreachable when the function's precondition holds.\n\n**Options:**\n1. Remove the check and always encode `content` (assumes `_is_text_stream` is correct).\n2. Keep the check and add an `else: return 0` or `raise` branch to handle the \"impossible\" case.\n\nThe current code returns `None` when `content` is not `str`, which would cause `super_len` to fall through to the binary seek logic (lines 259–262). This is safe but circuitous.\n\n**Recommendation:** Document that this is a defensive fallback, or remove it and assert the precondition."
  },
  {
    "path": "tests/test_utils.py",
    "line": 171,
    "body": "**AC-4 verification gap: No test confirms existing body-length tests still pass.**\n\nAC-4 states: \"all body-length tests pass without failures\" and \"Tests for multipart uploads, JSON bodies, and explicit `Content-Length` headers remain unaffected.\"\n\nThe new test class `TestTextStreamContentLength` adds three tests (AC-1, AC-2, AC-3) but does not verify that *existing* tests in the suite remain green. AC-4's verification requires running `python -m pytest tests` and confirming 0 failures for *all* body-length tests, not just the new ones.\n\n**Missing evidence:** The PR does not include:\n- A test run output showing existing tests pass.\n- A comment or assertion that references existing test coverage (e.g., `test_prepare_body_with_explicit_content_length`, `test_json_body_length`).\n\n**Recommendation:** Add a comment in the test file or PR description confirming that the full suite was run locally and passed, or add a regression test that imports and re-runs a known body-length test from another module."
  }
]
```
