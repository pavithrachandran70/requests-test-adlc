# Requirements — Keep a caller-supplied SSLContext when opening an HTTPS connection

**Ticket:** PER-57  
**Type:** Story  
**Scope:** Preserve a custom `ssl_context` passed to `HTTPAdapter.init_poolmanager` so its `verify_mode` and hostname checks are not overridden by the pool-key construction logic in `src/requests/adapters.py`.

---

## Context

When a caller passes `ssl_context=ctx` to `HTTPAdapter.init_poolmanager(...)`, urllib3 stores the context on the pool manager. However, the next request does not use that context as-is. The pool-key construction in `get_connection_with_tls_context` (via `_urllib3_request_context`) always sets `cert_reqs` to `CERT_REQUIRED` when `verify=True`, and urllib3 then executes `context.verify_mode = resolve_cert_reqs(cert_reqs)` while connecting. The caller's context is rewritten.

**Verified locally:** A context created with `verify_mode` `CERT_NONE` (`0`) is still the pool's context, but the pool's `cert_reqs` is `CERT_REQUIRED` (`2`), so the connection enforces certificate verification despite the caller's intent.

**Expected behavior:** An `ssl_context` passed to `init_poolmanager` is the context used for that adapter's HTTPS connections, including its `verify_mode` and hostname checks. A normal request with `verify=True` and no custom context still verifies certificates.

**Constraint:** Repo `requests` only. Change pool-key construction in `src/requests/adapters.py` so a custom context is not overridden by `cert_reqs`. Add a unit test that builds the pool and does not open a socket. Do not change `verify=False`, CA-bundle paths, or client-cert file checks.

---

## Acceptance Criteria

### AC-1: Custom SSLContext preserved

**Given** an `HTTPAdapter` initialized with `init_poolmanager(ssl_context=ctx)` where `ctx.verify_mode` is `ssl.CERT_NONE`  
**When** the adapter selects a pool for `https://example.com/`  
**Then** the pool carries that same context, and `cert_reqs` does not force `CERT_REQUIRED` onto it.

---

### AC-2: Default verify=True enforced

**Given** a request with `verify=True` and no custom context  
**When** the adapter builds a pool key  
**Then** `cert_reqs` is set to `CERT_REQUIRED`.

---

### AC-3: Adapter tests pass

**Given** the adapter logic is changed  
**When** `python -m pytest tests` runs for the adapter tests  
**Then** all tests pass.

---

## Open Questions

None at this time. The scope is clear: preserve a custom context's `verify_mode` while leaving default verification unchanged.

---

## Notes

- The fix targets `_urllib3_request_context` and/or `get_connection_with_tls_context` in `src/requests/adapters.py`.
- The unit test must verify both the custom-context path (AC-1) and the default path (AC-2) without opening a socket.
- `verify=False`, CA-bundle paths, and client-cert file checks are out of scope for this ticket.
