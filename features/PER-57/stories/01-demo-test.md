# Low-Level Design — Story 01-demo-test: Preserve custom SSLContext in HTTPAdapter pool-key construction

**Ticket:** PER-62  
**Story:** 01-demo-test  
**Repo:** demo-test  
**Type:** Story

---

## Overview

This story modifies the pool-key construction logic in `_urllib3_request_context` (in `src/requests/adapters.py`) to detect when a custom `ssl.SSLContext` has been supplied to `HTTPAdapter.init_poolmanager` and omit the `cert_reqs` override in that case. When no custom context is present and `verify=True`, the pool key must still set `cert_reqs` to `CERT_REQUIRED`. Unit tests verify both the custom-context path and the default path without opening a socket.

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
**When** `python -m pytest tests/test_adapters.py` runs for the adapter tests  
**Then** all tests pass.

---

## File Plan

### Modified Files

#### 1. `src/requests/adapters.py`

**Function:** `_urllib3_request_context(self, request, verify, cert=None, proxies=None)`

**Current behavior:**  
The function builds a dictionary of pool kwargs for urllib3. When `verify=True`, it always sets `cert_reqs` to `ssl.CERT_REQUIRED` (int value `2`). When `verify=False`, it sets `cert_reqs` to `ssl.CERT_NONE` (int value `0`).

**Change:**  
Add conditional logic to detect a custom `ssl_context` from `self.poolmanager.connection_pool_kw["ssl_context"]`. When a custom context is present, omit the `cert_reqs` key from the returned dict. When no custom context is present, preserve the existing behavior.

**Pseudocode:**

```python
def _urllib3_request_context(self, request, verify, cert=None, proxies=None):
    host_params, pool_kwargs = ..., {}
    
    # Detect custom ssl_context
    custom_context = None
    if self.poolmanager is not None:
        custom_context = getattr(self.poolmanager, 'connection_pool_kw', {}).get('ssl_context')
    
    # Conditional cert_reqs override
    if custom_context is not None:
        # Custom context present: omit cert_reqs to preserve context.verify_mode
        pass
    elif verify is True:
        # No custom context, verify=True: set cert_reqs to CERT_REQUIRED
        pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED
    elif verify is False:
        # verify=False: set cert_reqs to CERT_NONE (existing behavior)
        pool_kwargs['cert_reqs'] = ssl.CERT_NONE
    else:
        # verify is a CA bundle path (string): existing cert_reqs logic
        pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED
        pool_kwargs['ca_certs'] = verify
    
    # ... rest of pool_kwargs construction
    return host_params, pool_kwargs
```

**Exact location:**  
The `_urllib3_request_context` function is a private method of the `HTTPAdapter` class in `src/requests/adapters.py`. The conditional logic must be inserted after the function retrieves `verify` and before it returns the `pool_kwargs` dict. The existing `cert_reqs` assignment (approximately line 250–260 in the current codebase) will be replaced with the conditional branch above.

**Lines affected:** Approximately 15–20 lines (conditional branch + custom context retrieval).

**Existing imports:**  
The `ssl` module is already imported at the top of `src/requests/adapters.py`. No new imports are required.

**Edge cases:**

- `self.poolmanager` is `None`: The conditional check must guard against `AttributeError` when accessing `self.poolmanager.connection_pool_kw`.
- `connection_pool_kw` does not exist: Use `getattr(self.poolmanager, 'connection_pool_kw', {})` to return an empty dict if the attribute is missing.
- `ssl_context` key is missing: Use `.get('ssl_context')` to return `None` if the key is missing.

---

#### 2. `tests/test_adapters.py`

**Change:** Add two new test cases to verify AC-1 and AC-2.

**Test case 1: `test_custom_sslcontext_preserves_verify_mode`**

**Purpose:** Verify AC-1 — a custom SSLContext's `verify_mode` is preserved (pool kwargs dict omits `cert_reqs` key).

**Pseudocode:**

```python
def test_custom_sslcontext_preserves_verify_mode():
    """
    AC-1: After init_poolmanager(ssl_context=ctx) where ctx.verify_mode 
    is ssl.CERT_NONE, the pool kwargs dict omits cert_reqs key.
    """
    import ssl
    from unittest.mock import Mock
    from requests.adapters import HTTPAdapter
    
    # Create custom SSLContext with verify_mode=CERT_NONE
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    
    # Initialize adapter with custom context
    adapter = HTTPAdapter()
    adapter.init_poolmanager(connections=1, maxsize=1, ssl_context=ctx)
    
    # Mock a PreparedRequest for https://example.com/
    request = Mock()
    request.url = 'https://example.com/'
    request.headers = {}
    
    # Call _urllib3_request_context with verify=True
    host_params, pool_kwargs = adapter._urllib3_request_context(
        request, verify=True, cert=None, proxies=None
    )
    
    # Assert cert_reqs key is NOT in pool_kwargs
    assert 'cert_reqs' not in pool_kwargs, (
        "Expected pool_kwargs to omit cert_reqs when custom ssl_context is present"
    )
```

**Lines affected:** Approximately 20 lines.

---

**Test case 2: `test_default_verify_enforces_cert_required`**

**Purpose:** Verify AC-2 — a request with `verify=True` and no custom context sets `cert_reqs` to `CERT_REQUIRED`.

**Pseudocode:**

```python
def test_default_verify_enforces_cert_required():
    """
    AC-2: A request with verify=True and no custom context sets 
    cert_reqs to CERT_REQUIRED.
    """
    import ssl
    from unittest.mock import Mock
    from requests.adapters import HTTPAdapter
    
    # Initialize adapter without custom context
    adapter = HTTPAdapter()
    adapter.init_poolmanager(connections=1, maxsize=1)
    
    # Mock a PreparedRequest for https://example.com/
    request = Mock()
    request.url = 'https://example.com/'
    request.headers = {}
    
    # Call _urllib3_request_context with verify=True
    host_params, pool_kwargs = adapter._urllib3_request_context(
        request, verify=True, cert=None, proxies=None
    )
    
    # Assert cert_reqs is CERT_REQUIRED (int value 2)
    assert pool_kwargs.get('cert_reqs') == ssl.CERT_REQUIRED, (
        f"Expected pool_kwargs['cert_reqs'] to be CERT_REQUIRED (2), "
        f"got {pool_kwargs.get('cert_reqs')}"
    )
```

**Lines affected:** Approximately 20 lines.

---

**Existing imports:**  
The `ssl` module and `unittest.mock` are already available in the test environment. No new imports are required at the module level.

**Test isolation:**  
Both test cases use `unittest.mock.Mock` to create a fake `PreparedRequest` object. They do **not** open a network socket. They call `_urllib3_request_context` directly and assert on the returned `pool_kwargs` dict.

**Lines affected (total):** Approximately 40 lines for test cases 1 and 2.

---

### No Changes Required

- **`src/requests/__init__.py`**: No change (no public API surface change).
- **`src/requests/sessions.py`**: No change (adapter logic is isolated).
- **`tests/conftest.py`**: No change (no new fixtures required).
- **Other test files**: No change (existing adapter tests must pass per AC-3).

---

## Interfaces

### Internal Function Signature

**Modified:** `HTTPAdapter._urllib3_request_context(self, request, verify, cert=None, proxies=None)`

**Input:**

- `self` (HTTPAdapter instance): The adapter instance, which may or may not have a `poolmanager` with a custom `ssl_context`.
- `request` (PreparedRequest): The prepared request object with `url` and `headers`.
- `verify` (bool | str | None): Whether to verify SSL certificates. `True` means verify with default CA bundle, `False` means do not verify, a string path means use that CA bundle.
- `cert` (tuple | str | None, optional): Client certificate file paths. Not used in this change.
- `proxies` (dict | None, optional): Proxy configuration. Not used in this change.

**Output:**

- `host_params` (dict): Host-specific connection parameters (not modified by this change).
- `pool_kwargs` (dict): Pool-manager keyword arguments. The `cert_reqs` key is conditionally included based on the presence of a custom `ssl_context`.

**Behavioral contract:**

- **When a custom `ssl_context` is present** (detected via `self.poolmanager.connection_pool_kw.get('ssl_context')`):  
  The returned `pool_kwargs` dict does **not** contain a `cert_reqs` key. This allows urllib3 to use the custom context's `verify_mode` without override.

- **When no custom `ssl_context` is present and `verify=True`:**  
  The returned `pool_kwargs` dict contains `cert_reqs=ssl.CERT_REQUIRED` (int value `2`), preserving the existing default behavior.

- **When `verify=False`:**  
  The returned `pool_kwargs` dict contains `cert_reqs=ssl.CERT_NONE` (int value `0`), unchanged from existing logic.

- **When `verify` is a string (CA bundle path):**  
  The returned `pool_kwargs` dict contains `cert_reqs=ssl.CERT_REQUIRED` and `ca_certs=verify`, unchanged from existing logic.

**Invariant:**  
A custom context's `verify_mode` is never overridden by the pool-key construction logic. The default certificate-verification behavior (verify=True → CERT_REQUIRED) is preserved when no custom context is present.

---

## Algorithm Notes

### Custom Context Detection

**Step 1: Guard against `None` poolmanager**

```python
if self.poolmanager is not None:
    custom_context = getattr(self.poolmanager, 'connection_pool_kw', {}).get('ssl_context')
else:
    custom_context = None
```

**Rationale:** The `poolmanager` attribute may be `None` before `init_poolmanager` is called. Use `getattr` with a default empty dict to avoid `AttributeError`.

---

### Conditional `cert_reqs` Assignment

**Step 2: Branch on custom context presence**

```python
if custom_context is not None:
    # Custom context present: omit cert_reqs to preserve context.verify_mode
    pass
elif verify is True:
    # No custom context, verify=True: set cert_reqs to CERT_REQUIRED
    pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED
elif verify is False:
    # verify=False: set cert_reqs to CERT_NONE
    pool_kwargs['cert_reqs'] = ssl.CERT_NONE
else:
    # verify is a CA bundle path (string): set cert_reqs to CERT_REQUIRED and ca_certs
    pool_kwargs['cert_reqs'] = ssl.CERT_REQUIRED
    pool_kwargs['ca_certs'] = verify
```

**Rationale:** The conditional branch mirrors the existing `verify` logic but adds a guard for custom contexts. When a custom context is present, the `cert_reqs` key is omitted entirely, so urllib3 does not override the context's `verify_mode`.

---

### Existing Logic Preservation

**Step 3: Preserve CA bundle and client cert handling**

The existing logic for CA bundle paths (string `verify`) and client certificates (`cert` tuple) is **not modified**. The conditional branch only affects the `cert_reqs` assignment. All other pool kwargs (e.g. `ca_certs`, `cert_file`, `cert_data`) are built as before.

---

### Test Mocking Strategy

**Step 4: Mock PreparedRequest without opening a socket**

```python
from unittest.mock import Mock

request = Mock()
request.url = 'https://example.com/'
request.headers = {}
```

**Rationale:** The `_urllib3_request_context` function only reads `request.url` and `request.headers`. A `Mock` object with these attributes is sufficient for testing the pool-key construction logic. No network connection is opened.

**Step 5: Assert on pool kwargs dict**

```python
host_params, pool_kwargs = adapter._urllib3_request_context(
    request, verify=True, cert=None, proxies=None
)
assert 'cert_reqs' not in pool_kwargs  # AC-1
# OR
assert pool_kwargs.get('cert_reqs') == ssl.CERT_REQUIRED  # AC-2
```

**Rationale:** The pool kwargs dict is the contract between `requests` and `urllib3`. Asserting on its contents verifies the fix without opening a socket or creating a real connection pool.

---

## Tests

### Test 1: Custom SSLContext preserved (AC-1)

**File:** `tests/test_adapters.py`  
**Function:** `test_custom_sslcontext_preserves_verify_mode`

**Setup:**

1. Create a custom `ssl.SSLContext` with `verify_mode=ssl.CERT_NONE`.
2. Initialize an `HTTPAdapter` with `init_poolmanager(ssl_context=ctx)`.
3. Mock a `PreparedRequest` for `https://example.com/`.

**Action:**

4. Call `adapter._urllib3_request_context(request, verify=True, cert=None, proxies=None)`.

**Assertion:**

5. Assert the returned `pool_kwargs` dict does **not** contain a `cert_reqs` key.

**Expected outcome:**  
The test passes, confirming that a custom context's `verify_mode` is preserved (urllib3 will not override it).

---

### Test 2: Default verify=True enforced (AC-2)

**File:** `tests/test_adapters.py`  
**Function:** `test_default_verify_enforces_cert_required`

**Setup:**

1. Initialize an `HTTPAdapter` with `init_poolmanager(connections=1, maxsize=1)` (no custom context).
2. Mock a `PreparedRequest` for `https://example.com/`.

**Action:**

3. Call `adapter._urllib3_request_context(request, verify=True, cert=None, proxies=None)`.

**Assertion:**

4. Assert the returned `pool_kwargs` dict contains `cert_reqs=ssl.CERT_REQUIRED` (int value `2`).

**Expected outcome:**  
The test passes, confirming that the default certificate-verification behavior is preserved when no custom context is present.

---

### Test 3: Adapter tests pass (AC-3)

**File:** N/A (manual verification)  
**Command:** `python -m pytest tests/test_adapters.py`

**Action:**

1. Run the full adapter test suite after implementing the fix.

**Assertion:**

2. Confirm all tests pass, including the two new test cases and all pre-existing adapter tests.

**Expected outcome:**  
All tests pass. If any test fails, investigate whether the failure is due to:
  - A genuine regression (fix the code).
  - A test assumption that needs updating (update the test).

---

## Verification

- **AC-1:** Unit test `test_custom_sslcontext_preserves_verify_mode` asserts the pool kwargs dict omits `cert_reqs` key when a custom context with `verify_mode=ssl.CERT_NONE` is supplied. (1 test case)
- **AC-2:** Unit test `test_default_verify_enforces_cert_required` asserts the pool kwargs dict contains `cert_reqs=ssl.CERT_REQUIRED` when `verify=True` and no custom context is supplied. (1 test case)
- **AC-3:** Run `python -m pytest tests/test_adapters.py` and confirm all tests pass. (1 suite run)

**Total test count:** 2 test cases + 1 suite run (matches breakdown Verification Approach).

---

## Open Questions

None. All open questions from the tech spec have been resolved:

- **Q1 (pool manager attribute):** Confirmed that `self.poolmanager.connection_pool_kw` is the correct attribute to retrieve the custom `ssl_context`.
- **Q2 (hostname checks):** Out of scope for this ticket. The ACs do not test hostname checks, and the fix does not alter hostname-check logic.
- **Q3 (test mocking):** Confirmed that the unit tests will use `unittest.mock.Mock` to create a fake `PreparedRequest` object and assert on the pool kwargs dict without opening a socket.

---

**End of LLD**
