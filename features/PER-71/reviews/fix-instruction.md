Fix these forge PR review comments (auto-route):

### demo-test comment 4399426037
**AC Gap: Position handling violates AD-4**

The helper `_text_stream_utf8_length` receives `current_position` as a parameter, but the caller (`super_len`) computes `current_position` from `o.tell()` at line 199 — **before** entering the text-stream branch. This violates AD-4: "Position must be captured with `.tell()` **before** reading content for length calculation." The current flow is:

1. Line 199: `current_position = o.tell()` (earliest capture, outside the text-stream branch)
2. Line 253: `text_length = _text_stream_utf8_length(o, current_position)` (text-stream branch entered much later)
3. Line 184 inside helper: `content = o.read()` (position already moved)

The fix works only because `current_position` was captured early. However, the logic split across two functions obscures the AD-4 requirement. If a future edit moves the `current_position = o.tell()` line (e.g., into a different branch), the text-stream path will break.

**Guard:** Move `current_position = o.tell()` **inside** the text-stream branch (line 253 region), immediately before calling `_text_stream_utf8_length`, and pass it explicitly:

```python
if hasattr(o, "seek") and total_length is None:
    try:
        # Text streams: position taken BEFORE read (AD-4)
        current_position_for_text = o.tell()
        text_length = _text_stream_utf8_length(o, current_position_for_text)
        if text_length is not None:
            return text_length
        # ... existing seek-to-end logic
```

This makes the AD-4 invariant visible at the call site and self-contained within the text-stream logic.

### demo-test comment 4399426090
**Test design: `TestTextStreamContentLength` does not verify stream position is restored**

The tests confirm `Content-Length` is correct, but they do not verify that the stream position is restored after `super_len` runs. AD-4 and the HLD require:

> Position is taken with `tell()` before the text-length helper. ... Restore the stream's original position with `seek(current_position)`.

If `super_len` leaves the stream at EOF instead of its original position, `prepare_body` will read zero bytes when it consumes the body, producing an empty request. The AC tests would pass (they only check `Content-Length`), but the fix would still be broken.

**Potential consequence:** A future refactor might remove the `seek()` call, and the AC tests would not catch it.

**Guard:** Add a verification step to one of the tests:

```python
def test_text_stringio_utf8_length(self):
    text = "héllo"
    stream = io.StringIO(text)
    req = requests.Request("PUT", "http://example.invalid/x", data=stream)
    prepared = req.prepare()
    assert prepared.headers["Content-Length"] == "6"
    # Verify stream was consumed (prepare_body reads it)
    # but if prepare_body or super_len left it at EOF, this would fail:
    assert stream.tell() == 0 or stream.read() == text  # Position or content intact
```

Alternatively, add a dedicated unit test for `_text_stream_utf8_length` that verifies position restoration directly.
