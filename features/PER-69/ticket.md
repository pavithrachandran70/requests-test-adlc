---
key: PER-69
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Coerce non-string multipart field names instead of raising AttributeError
---

Problem. A non-string key in `data` is accepted for a urlencoded body, but the same key fails once `files` is set and the body becomes multipart. A non-string key in `files` fails the same way. Verified locally:

* `Request("POST", "http://x", data={1: "a"}).prepare()` succeeds and the body is `1=a`.
* `Request("POST", "http://x", data={1: "a"}, files={"f": b"x"}).prepare()` raises `AttributeError: 'int' object has no attribute 'translate'`.
* `Request("POST", "http://x", files={1: b"x"}).prepare()` raises the same `AttributeError`.

`Request._encode_files` in `src/requests/models.py` converts values with `str()`, then passes the field name through unchanged at `RequestField(name=k, ...)`.

Expected behavior. Both `prepare()` calls succeed. The multipart body uses the string `"1"` as the field name, matching the urlencoded path. No `AttributeError`.

Scope. Repo `requests` only. Change field-name handling in `_encode_files`. Add a unit test that calls `prepare()` and does not open a network connection. Do not change urlencoded encoding, file-tuple shapes, or public method signatures.
