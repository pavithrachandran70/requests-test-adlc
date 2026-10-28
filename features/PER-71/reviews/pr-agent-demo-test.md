```json
[
  {
    "path": "src/requests/utils.py",
    "line": 160,
    "body": "**Bug: `_is_text_stream` fallback reads sample at current position, not start**\n\nThe `_is_text_stream` helper calls `read(0)` to probe for text vs. binary. `read(0)` may raise or return empty bytes on some file-like objects, but it **does not reset the stream position**. If the stream is at EOF when this probe runs, `read(0)` may still return `b''` (bytes) for a binary stream or `''` (str) for a text stream, but the stream remains at EOF. The subsequent `_text_stream_utf8_length` call then reads from EOF → empty content → zero length.\n\n**Expected:** The probe should not move the stream position, or the position should be captured and restored around the probe.\n\n**AC violated:** AC-1 (UTF-8 length for text streams) — if the stream is not at position 0 when `super_len` is called, the length may be incorrect.\n\n**Fix:** Remove the `read(0)` probe entirely and rely on `isinstance(o, io.TextIOBase)` alone (the primary detection per AD-2). If the probe is kept for non-stdlib streams, capture position before `read(0)` and restore it after."
  },
  {
    "path": "src/requests/utils.py",
    "line": 183,
    "body": "**Bug: `_text_stream_utf8_length` returns `None` when `content` is not `str`, causing incorrect fallback**\n\nIf `o.read()` returns something other than `str` (e.g., a broken text stream that returns bytes), `_text_stream_utf8_length` returns `None`. The caller in `super_len` then treats this as \"not a text stream\" and falls through to the binary seek-to-end logic, which may return the wrong length or fail.\n\n**Expected:** If `_is_text_stream(o)` returns `True` but `o.read()` does not return `str`, raise an exception or return 0 — do not silently fall back to binary logic.\n\n**AC violated:** AC-1 (UTF-8 length for text streams) — a text stream that returns non-str content should fail explicitly, not return a binary length.\n\n**Fix:** In `_text_stream_utf8_length`, after `content = o.read()`, check `if not isinstance(content, str): raise TypeError(\"Text stream read() returned non-str\")` or return 0. Do not return `None` and allow fallback."
  },
  {
    "path": "src/requests/utils.py",
    "line": 259,
    "body": "**Bug: `current_position_for_text` is captured inside the `try` block, violating AD-4**\n\nAD-4 (per `hld.md` and `change_requests.md`) requires: *\"Position is captured with `.tell()` **immediately before** the text-length helper is called, within the text-stream branch.\"* The current code captures `current_position_for_text` inside the `try` block that also contains the seek-to-end logic for binary streams. If the text-stream branch is not taken (e.g., `_text_stream_utf8_length` returns `None` due to the bug above), the position was captured too early — before determining whether the stream is text or binary.\n\n**Expected:** Move `current_position_for_text = o.tell()` **outside** the `try` block, immediately before the call to `_text_stream_utf8_length`, so it is clear that the position is taken for the text-stream branch alone.\n\n**AC violated:** None directly, but violates architectural constraint AD-4.\n\n**Fix:** Move the `o.tell()` call for text streams to just before `text_length = _text_stream_utf8_length(...)`, outside the `try` block that handles binary streams."
  },
  {
    "path": "tests/test_utils.py",
    "line": 170,
    "body": "**Weak assertion: `stream.tell() == 0 or stream.read() == text` does not verify position restoration correctly**\n\nThe test attempts to verify that `super_len` restores the stream position (per AC-1 and AD-4). However, the assertion `stream.tell() == 0 or stream.read() == text` is always true if the stream is at position 0, even if `super_len` left it at EOF. If the stream is at EOF, `stream.read()` returns `''`, and the assertion `'' == text` fails — but the assertion is guarded by `stream.tell() == 0`, so the EOF case is never tested.\n\n**Expected:** Assert `stream.tell() == 0` (the position is restored to the start), then assert `stream.read() == text` (reading from that position yields the full content).\n\n**AC violated:** AC-1 (\"restores position\") — the test does not actually verify position restoration in all cases.\n\n**Fix:** Change the assertion to:\n```python\nassert stream.tell() == 0, \"Stream position not restored to start\"\nassert stream.read() == text, \"Stream content not readable after super_len\"\n```"
  },
  {
    "path": "tests/test_utils.py",
    "line": 181,
    "body": "**Weak assertion: `stream.tell() == 0 or stream.read() == text` does not verify position restoration correctly**\n\nSame issue as AC-1 test (line 170). The assertion does not verify that the stream position is restored to the start after `super_len` is called.\n\n**Expected:** Assert `stream.tell() == 0`, then assert `stream.read() == text`.\n\n**AC violated:** AC-3 (\"restores position\") — the test does not actually verify position restoration in all cases.\n\n**Fix:** Change the assertion to:\n```python\nassert stream.tell() == 0, \"Stream position not restored to start\"\nassert stream.read() == text, \"Stream content not readable after super_len\"\n```"
  },
  {
    "path": "tests/test_utils.py",
    "line": 187,
    "body": "**Missing test: AC-4 requires running the full test suite, not a single assertion**\n\nAC-4 states: *\"CI/local run of `python -m pytest tests` shows 0 failures for new and existing body-length tests.\"* The current test method `test_existing_body_length_tests_pass` contains two assertions on `super_len` directly, but does not verify that the **entire** test suite passes.\n\n**Expected:** This test method should be removed or renamed to clarify it is not the full AC-4 validation. The real validation is the CI run of `pytest tests`, which is not represented in this diff.\n\n**AC violated:** AC-4 (full test suite passes) — the test does not verify the full suite, only two calls to `super_len`.\n\n**Fix:** Remove this test method or rename it to `test_super_len_examples` and add a comment: `# Full AC-4 validation is the CI run of pytest tests`."
  },
  {
    "path": "src/requests/utils.py",
    "line": 265,
    "body": "**Missing edge case: `_text_stream_utf8_length` may raise if `o.seek()` fails**\n\nIf the text stream does not support `seek()` (e.g., a wrapped socket or pipe), `o.seek(current_position or 0)` inside `_text_stream_utf8_length` will raise `io.UnsupportedOperation` or `AttributeError`. The exception is not caught, so `super_len` will propagate it to the caller.\n\n**Expected:** Per the HLD edge case note, *\"If `tell()` or `seek()` raises `io.UnsupportedOperation`, the function falls back to existing logic (which may return 0 or raise an error).\"* The code should catch seek failures and return `None` (allowing fallback to existing logic) or return 0.\n\n**AC violated:** None directly (edge case is documented as \"acceptable\" in HLD), but the exception may break callers that expect `super_len` to return an int.\n\n**Fix:** Wrap `o.seek(current_position or 0)` in a `try/except (io.UnsupportedOperation, AttributeError, OSError)` block. On exception, return `None` or 0."
  }
]
```
