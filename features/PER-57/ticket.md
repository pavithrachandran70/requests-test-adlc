---
key: PER-57
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: 'High: custom SSLContext is forced back to certificate verification'
---

Title: Keep a caller-supplied SSLContext when opening an HTTPS connection

Problem. `HTTPAdapter.init_poolmanager(..., ssl_context=ctx)` stores the context on the urllib3 pool manager. The next request does not use that context as-is. `get_connection_with_tls_context` builds a new pool key through `_urllib3_request_context` in `src/requests/adapters.py`, and with `verify=True` that key always sets `cert_reqs` to `CERT_REQUIRED`. urllib3 then executes `context.verify_mode = resolve_cert_reqs(cert_reqs)` while connecting, so the caller's context is rewritten. Verified locally without a network connection: a context created with `verify_mode` `CERT_NONE` (`0`) is still the pool's context, but the pool's `cert_reqs` is `CERT_REQUIRED` (`2`).

Expected behavior. An `ssl_context` passed to `init_poolmanager` is the context used for that adapter's HTTPS connections, including its `verify_mode` and hostname checks. A normal request with `verify=True` and no custom context still verifies certificates.

Scope. Repo `requests` only. Change pool-key construction in `src/requests/adapters.py` so a custom context is not overridden by `cert_reqs`. Add a unit test that builds the pool and does not open a socket. Do not change `verify=False`, CA-bundle paths, or client-cert file checks.

Acceptance criteria.

* After `init_poolmanager(ssl_context=ctx)` where `ctx.verify_mode` is `ssl.CERT_NONE`, the pool selected for `https://example.com/` still carries that same context, and `cert_reqs` does not force `CERT_REQUIRED` onto it.
* A request with `verify=True` and no custom context still sets `cert_reqs` to `CERT_REQUIRED`.
* `python -m pytest tests` passes for the adapter tests.
