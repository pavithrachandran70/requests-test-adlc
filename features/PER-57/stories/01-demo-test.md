# Story 01-demo-test LLD: Preserve custom SSLContext in HTTPAdapter pool-key construction

**Repo:** `demo-test`  
**Story ID:** 01-demo-test  
**Type:** Story

---

## Acceptance Criteria

*(Copied verbatim from requirements and breakdown)*

- **AC-1:** After `init_poolmanager(ssl_context=ctx)` where `ctx.verify_mode` is `ssl.CERT_NONE`, the pool selected for `https://example.com/` still carries that same context, and `cert_reqs` does not force `CERT_REQUIRED` onto it.

- **AC-2:** A request with `verify=True` and no custom context still sets `cert_reqs` to `CERT_REQUIRED`.

- **AC-3:** `python -m pytest tests/test_adapters.py` passes for the adapter tests.

---

## File Plan

### 1. `src/requests/adapters.py`

**Change:** Modify the `_urllib3_request_context` method in the `HTTPAdapter` class.

**Current behavior:**  
The method unconditionally sets `cert_reqs` in the pool kwargs dict based on the `verify` parameter, overriding any custom `ssl_context.verify_mode` supplied to `init_poolmanager`.

**New behavior:**  
Before setting `cert_reqs`, check whether a custom `ssl_context` was supplied to `init_poolmanager`. If so, omit the `cert_reqs` key from the pool kwargs dict. If no custom context is present, preserve the existing logic:
- `verify=True` → `cert_reqs = ssl.CERT_REQUIRED` (int `2`)
- `verify=False` → `cert_reqs = ssl.CERT_NONE` (int `0`)

**Implementation notes:**

- The custom context is stored in `self.poolmanager.connection_pool_kw["ssl_context"]` when passed to `init_poolmanager`.
- The conditional check must verify:
  1. `self.poolmanager` is not `None`.
  2. `self.poolmanager.connection_pool_kw.get("ssl_context")` is truthy.
- When both conditions are met, do **not** set the `cert_reqs` key in the returned pool kwargs dict.
- When the custom context is absent, preserve the existing `cert_reqs` assignment logic.

**Pseudocode for the conditional branch:**

```python
def _urllib3_request_context(self, request, verify, cert):
    # ... existing logic to build pool_kwargs dict ...
    
    # NEW: Check for custom ssl_context
    custom_context = (
        self.poolmanager
        and hasattr(self.poolmanager, 'connection_pool_kw')
        and self.poolmanager.connection_pool_kw.get('ssl_context')
    )
    
    # Conditional cert_reqs assignment
    if not custom_context:
        if verify is True:
            pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED  # int 2
        elif verify is False:
            pool_kwargs['cert_reqs'] = ssl.CERT_NONE      # int 0
        # else: verify is a string (CA bundle path), existing logic handles it
    
    # ... rest of existing logic ...
    return pool_kwargs
```

**Lines affected:** Approximately 10–15 lines (conditional check + branch logic).

---

### 2. `tests/test_adapters.py`

**Change:** Add 2 new test functions to verify AC-1 and AC-2.

**Test case 1: `test_custom_sslcontext_preserves_verify_mode` (AC-1)**

- **Setup:**
  - Import `ssl` and `HTTPAdapter`.
  - Create a custom `ssl.SSLContext` with `verify_mode=ssl.CERT_NONE` and `check_hostname=False`.
  - Initialize an `HTTPAdapter` and call `init_poolmanager(connections=10, maxsize=10, ssl_context=ctx)`.
  - Mock a `PreparedRequest` for `https://example.com/`.

- **Action:**
  - Call `adapter._urllib3_request_context(request, verify=True, cert=None)`.

- **Assertion:**
  - Assert that the returned pool kwargs dict does **not** contain a `cert_reqs` key, or if it does, its value is **not** `ssl.CERT_REQUIRED` (int `2`).

**Test case 2: `test_default_verify_enforces_cert_required` (AC-2)**

- **Setup:**
  - Import `ssl` and `HTTPAdapter`.
  - Initialize an `HTTPAdapter` and call `init_poolmanager(connections=10, maxsize=10)` (no custom context).
  - Mock a `PreparedRequest` for `https://example.com/`.

- **Action:**
  - Call `adapter._urllib3_request_context(request, verify=True, cert=None)`.

- **Assertion:**
  - Assert that the returned pool kwargs dict contains `cert_reqs=ssl.CERT_REQUIRED` (int `2`).

**Mock strategy:**

- Use `unittest.mock.Mock` or `pytest` fixtures to create a `PreparedRequest` object with a `url` attribute set to `https://example.com/`.
- Do **not** open a network socket or make a real HTTP request.
- If the real `_urllib3_request_context` method requires additional attributes on the request object, mock those as well (e.g. `headers`, `body`).

**Lines affected:** Approximately 40–60 lines (2 test functions + setup + assertions + docstrings).

---

## Interfaces

### Internal: `_urllib3_request_context`

**Signature (existing):**

```python
def _urllib3_request_context(
    self,
    request: PreparedRequest,
    verify: bool | str | None,
    cert: tuple | str | None
) -> dict:
```

**Return value:** A dict with keys including:
- `url` (str)
- `headers` (dict)
- `cert_reqs` (int, optional) — **NEW: omitted when custom ssl_context is present**
- `ca_certs` (str, optional)
- `cert_file` (str, optional)
- `key_file` (str, optional)
- Other urllib3 connection pool kwargs

**Behavioral contract (updated):**

- When `self.poolmanager` has a custom `ssl_context` (non-`None` in `connection_pool_kw`), the returned dict **omits** the `cert_reqs` key.
- When no custom `ssl_context` is present and `verify=True`, the returned dict includes `cert_reqs=ssl.CERT_REQUIRED` (int `2`).
- When `verify=False`, the returned dict includes `cert_reqs=ssl.CERT_NONE` (int `0`).
- When `verify` is a string (CA bundle path), existing logic applies (not changed by this story).

---

### Test: `test_custom_sslcontext_preserves_verify_mode`

**Inputs:**

- An `HTTPAdapter` initialized with a custom `ssl.SSLContext` where `ctx.verify_mode=ssl.CERT_NONE`.
- A mocked `PreparedRequest` for `https://example.com/`.
- `verify=True`, `cert=None`.

**Expected output:**

- The pool kwargs dict returned by `_urllib3_request_context` does **not** contain `cert_reqs`, or its value is **not** `ssl.CERT_REQUIRED`.

**Assertion:**

```python
pool_kwargs = adapter._urllib3_request_context(request, verify=True, cert=None)
assert 'cert_reqs' not in pool_kwargs or pool_kwargs['cert_reqs'] != ssl.CERT_REQUIRED
```

---

### Test: `test_default_verify_enforces_cert_required`

**Inputs:**

- An `HTTPAdapter` initialized with `init_poolmanager(connections=10, maxsize=10)` (no custom context).
- A mocked `PreparedRequest` for `https://example.com/`.
- `verify=True`, `cert=None`.

**Expected output:**

- The pool kwargs dict returned by `_urllib3_request_context` contains `cert_reqs=ssl.CERT_REQUIRED` (int `2`).

**Assertion:**

```python
pool_kwargs = adapter._urllib3_request_context(request, verify=True, cert=None)
assert pool_kwargs.get('cert_reqs') == ssl.CERT_REQUIRED
```

---

## Algorithm Notes

### Custom context detection logic

**Goal:** Determine whether a custom `ssl_context` was supplied to `init_poolmanager`.

**Implementation:**

```python
custom_context = (
    self.poolmanager is not None
    and hasattr(self.poolmanager, 'connection_pool_kw')
    and self.poolmanager.connection_pool_kw.get('ssl_context') is not None
)
```

**Edge cases:**

- `self.poolmanager` is `None` → no custom context (adapter not initialized).
- `connection_pool_kw` attribute missing → no custom context (old urllib3 version or unexpected state).
- `ssl_context` key missing or `None` → no custom context (default behavior).

**Defensive check:** Use `hasattr` to avoid `AttributeError` if `connection_pool_kw` is missing.

---

### Conditional branch for `cert_reqs` assignment

**Goal:** Set `cert_reqs` only when no custom context is present.

**Implementation:**

```python
if not custom_context:
    if verify is True:
        pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED
    elif verify is False:
        pool_kwargs['cert_reqs'] = ssl.CERT_NONE
    # else: verify is a string (CA bundle path), handled by existing logic
# else: custom_context is present, do NOT set cert_reqs
```

**Rationale:**

- When a custom context is present, urllib3 will use the context's `verify_mode` directly. Setting `cert_reqs` would override it (via `context.verify_mode = resolve_cert_reqs(cert_reqs)` in urllib3).
- When no custom context is present, the existing behavior (default certificate verification) must be preserved.

---

### Mock strategy for unit tests

**Goal:** Test the pool kwargs dict without opening a socket.

**Implementation:**

```python
from unittest.mock import Mock
from requests.adapters import HTTPAdapter
import ssl

def test_custom_sslcontext_preserves_verify_mode():
    adapter = HTTPAdapter()
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    
    adapter.init_poolmanager(connections=10, maxsize=10, ssl_context=ctx)
    
    # Mock PreparedRequest
    request = Mock()
    request.url = 'https://example.com/'
    # Add other required attributes if _urllib3_request_context accesses them
    
    pool_kwargs = adapter._urllib3_request_context(request, verify=True, cert=None)
    
    # Assert cert_reqs is not CERT_REQUIRED
    assert 'cert_reqs' not in pool_kwargs or pool_kwargs['cert_reqs'] != ssl.CERT_REQUIRED
```

**Rationale:**

- The `_urllib3_request_context` method builds a dict; it does not open a connection.
- Mocking the `PreparedRequest` object is sufficient to test the conditional logic.
- No network I/O is required.

---

## Tests

*(Matching the breakdown Verification Approach count exactly: 2 test cases + 1 suite run)*

### Test 1: AC-1 — Custom SSLContext preserved

**File:** `tests/test_adapters.py`  
**Function:** `test_custom_sslcontext_preserves_verify_mode`

**Setup:**

- Create an `HTTPAdapter`.
- Create a custom `ssl.SSLContext` with `verify_mode=ssl.CERT_NONE` and `check_hostname=False`.
- Call `adapter.init_poolmanager(connections=10, maxsize=10, ssl_context=ctx)`.
- Mock a `PreparedRequest` for `https://example.com/`.

**Action:**

- Call `adapter._urllib3_request_context(request, verify=True, cert=None)`.

**Assertion:**

- The returned pool kwargs dict does **not** contain `cert_reqs`, or if it does, its value is **not** `ssl.CERT_REQUIRED`.

**Implementation:**

```python
def test_custom_sslcontext_preserves_verify_mode():
    """
    AC-1: After init_poolmanager(ssl_context=ctx) where ctx.verify_mode 
    is ssl.CERT_NONE, the pool's cert_reqs does not force CERT_REQUIRED.
    """
    from unittest.mock import Mock
    import ssl
    from requests.adapters import HTTPAdapter
    
    adapter = HTTPAdapter()
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    
    adapter.init_poolmanager(connections=10, maxsize=10, ssl_context=ctx)
    
    # Mock PreparedRequest
    request = Mock()
    request.url = 'https://example.com/'
    
    pool_kwargs = adapter._urllib3_request_context(request, verify=True, cert=None)
    
    # Assert cert_reqs is not CERT_REQUIRED
    assert 'cert_reqs' not in pool_kwargs or pool_kwargs['cert_reqs'] != ssl.CERT_REQUIRED
```

---

### Test 2: AC-2 — Default verify=True enforced

**File:** `tests/test_adapters.py`  
**Function:** `test_default_verify_enforces_cert_required`

**Setup:**

- Create an `HTTPAdapter`.
- Call `adapter.init_poolmanager(connections=10, maxsize=10)` (no custom context).
- Mock a `PreparedRequest` for `https://example.com/`.

**Action:**

- Call `adapter._urllib3_request_context(request, verify=True, cert=None)`.

**Assertion:**

- The returned pool kwargs dict contains `cert_reqs=ssl.CERT_REQUIRED` (int `2`).

**Implementation:**

```python
def test_default_verify_enforces_cert_required():
    """
    AC-2: A request with verify=True and no custom context sets 
    cert_reqs to CERT_REQUIRED.
    """
    from unittest.mock import Mock
    import ssl
    from requests.adapters import HTTPAdapter
    
    adapter = HTTPAdapter()
    adapter.init_poolmanager(connections=10, maxsize=10)
    
    # Mock PreparedRequest
    request = Mock()
    request.url = 'https://example.com/'
    
    pool_kwargs = adapter._urllib3_request_context(request, verify=True, cert=None)
    
    # Assert cert_reqs is CERT_REQUIRED
    assert pool_kwargs.get('cert_reqs') == ssl.CERT_REQUIRED
```

---

### Test 3: AC-3 — Adapter tests pass

**Verification method:** Run `python -m pytest tests/test_adapters.py` after implementation.

**Expected outcome:** All tests pass, including the newly added tests and all pre-existing tests in the file.

**Note:** This is not a standalone test function; it is verified by running the test suite.

---

## Open Questions

None at this time. The scope is clear, the file plan is concrete, and the algorithm is defined.

---

**End of Story 01-demo-test LLD**
