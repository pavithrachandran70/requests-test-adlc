---
key: PER-77
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Coerce non-string multipart field names instead of raising AttributeError
---

A non-string key in `data` works for a urlencoded body, but the same key crashes once `files` is set. `Request._encode_files` converts values with `str()` and passes the field name through unchanged, so urllib3 raises `AttributeError: 'int' object has no attribute 'translate'`.

```python
requests.Request("POST", "http://x", data={1: "a"}).prepare()
# works, body is 1=a
requests.Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()
# AttributeError
requests.Request("POST", "http://x", files={1: b"x"}).prepare()
# AttributeError
```

**Expected behavior.** Both failing `prepare()` calls succeed. The multipart body uses the field name `"1"`. The urlencoded call still produces `1=a`.

**Scope.** Change field-name handling in `_encode_files` only. Add a unit test that calls `prepare()` and does not open a network connection. Do not change urlencoded encoding, file-tuple shapes, or public method signatures.

**Acceptance criteria.**

* The two failing calls above succeed.
* `data={1: "a"}` with no files still produces the body `1=a`.
* A new test covers an int key in `data` when `files` is set, and an int key in `files`.
* `python -m pytest tests` passes for that test and the existing request-preparation tests.
