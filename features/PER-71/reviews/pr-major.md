**Bug: `_text_stream_utf8_length` is called before `current_position` is set, causing it to receive `0` instead of the stream's actual position.**

Line 253 calls `_text_stream_utf8_length(o, current_position)`, but `current_position` is still `0` (initialized at line 248). The correct position is retrieved at line 264 via `current_position = o.tell()`, which happens *after* the text-stream check.

**Expected order:**
1. Retrieve `current_position = o.tell()` first.
2. Call `_text_stream_utf8_length(o, current_position)`.

**Fix:** Move `current_position = o.tell()` to before the `_text_stream_utf8_length` call, or pass `o.tell()` directly as the second argument.

**Consequence if shipped:** Text streams that are already partially consumed will have their position incorrectly restored to `0`, breaking AC-1 for streams where `.tell() != 0`.
