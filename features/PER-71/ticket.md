---
key: PER-71
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Set Content-Length from UTF-8 bytes for a text StringIO body
---

Problem. A text file-like body is sent as UTF-8, but `Content-Length` is the character count. `prepare_body` in `src/requests/models.py` treats a `StringIO` as a stream and calls `super_len` in `src/requests/utils.py`. Verified locally:

```
text = "héllo"  # 5 characters, 6 UTF-8 bytes
p = requests.Request("PUT", "http://example.invalid/x", data=StringIO(text)).prepare()
# Content-Length is 5, body is still a StringIO
```

A server that reads exactly `Content-Length` bytes stops one byte early.

Expected behavior. For a text `StringIO` or other text file object, `Content-Length` is the UTF-8 byte length. The body sent matches that length. A binary file object's length does not change.

Scope. Repo `requests` only. Fix length calculation for text streams in `super_len` or `prepare_body`. Add a unit test that calls `prepare()` and does not open a network connection. Do not change multipart uploads, JSON bodies, or an explicit `Content-Length` supplied by the caller.

Acceptance criteria.

* `data=StringIO("héllo")` prepares with `Content-Length: 6`.
* `data=BytesIO("héllo".encode())` still prepares with `Content-Length: 6`.
* An ASCII `StringIO("hello")` stays `Content-Length: 5`.
* `python -m pytest tests` passes for the body-length tests.
