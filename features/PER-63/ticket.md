---
key: PER-63
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: 3.Missing TLS files raise OSError instead of FileNotFoundError
---

Title: Raise FileNotFoundError when a client cert or key path is missing

Problem. `HTTPAdapter.cert_verify` in `src/requests/adapters.py` raises `OSError` when `cert` points at a file that does not exist. Verified locally: `requests.get("https://example.com", cert="/tmp/does-not-exist-adlc-cert.pem", timeout=2)` raises `OSError: Could not find the TLS certificate file, invalid path: /tmp/does-not-exist-adlc-cert.pem`. The same pattern is used for a missing key file. Callers cannot catch a missing file separately from other `OSError`s, and the exception has no `filename` attribute.

Expected behavior. A missing certificate path and a missing key path raise `FileNotFoundError`, which remains a subclass of `OSError`. The exception's `filename` is the missing path.

Scope. Repo `requests` only. Update the two raises in `cert_verify`. Add a unit test that does not perform a TLS handshake. Do not change behavior when the files exist, and do not change CA-bundle lookup.

Acceptance criteria.

* A missing cert path raises `FileNotFoundError` and `except OSError` still catches it.
* A missing key path does the same.
* `filename` on the exception is the path that was passed in.
* Existing tests that expect `OSError` still pass, or are updated only where they assert the exact type or message.
* `python -m pytest tests` passes for the TLS-cert tests.
