# Story — Raise FileNotFoundError when a client cert or key path is missing

**ID:** 3  
**Type:** story  
**Epic:** (none — backlog story)  
**Status:** planned  
**Covers:** (no spec capabilities)

---

## Problem Statement

`HTTPAdapter.cert_verify` in `src/requests/adapters.py` raises `OSError` when a cert or key path does not exist. Callers cannot distinguish a missing file from other OS-level errors, and the exception lacks a `filename` attribute.

**Current behavior (verified locally):**

```python
requests.get("https://example.com", cert="/tmp/does-not-exist-adlc-cert.pem", timeout=2)
# Raises: OSError: Could not find the TLS certificate file, invalid path: /tmp/does-not-exist-adlc-cert.pem
```

**Expected behavior:**

A missing certificate path or key path raises `FileNotFoundError` (a subclass of `OSError`) with the `filename` attribute set to the missing path.

---

## Scope

- **In scope:** Update the two raises in `src/requests/adapters.py::HTTPAdapter.cert_verify` for missing cert and missing key paths. Add unit tests that verify the exception type, `filename` attribute, and backward compatibility without performing a TLS handshake.
- **Out of scope:** CA-bundle lookup logic, TLS handshake behavior when files exist, changes to other repos or modules, unrelated `.gitignore` changes.

---

## Acceptance Criteria

### AC1: Missing cert path raises FileNotFoundError with filename attribute

**Given** a cert path that does not exist  
**When** the caller invokes `requests.get(url, cert=missing_path)`  
**Then** the exception raised is `FileNotFoundError`  
**And** the exception's `filename` attribute is the missing path  
**And** `except OSError` still catches the exception (FileNotFoundError is a subclass)

---

### AC2: Missing key path raises FileNotFoundError with filename attribute

**Given** a cert tuple `(cert_path, key_path)` where the key path does not exist  
**When** the caller invokes `requests.get(url, cert=(cert_path, missing_key_path))`  
**Then** the exception raised is `FileNotFoundError`  
**And** the exception's `filename` attribute is the missing key path  
**And** `except OSError` still catches the exception

---

### AC3: Existing tests remain compatible

**Given** tests that expect `OSError` for missing cert/key paths  
**When** the change is applied  
**Then** those tests still pass (FileNotFoundError is an OSError subclass)  
**Or** tests are updated only where they assert the exact exception type or message text

---

### AC4: Test suite passes for TLS cert edge cases

**Given** the updated implementation  
**When** `python -m pytest tests` runs for TLS-cert-related tests  
**Then** all tests pass, including new tests for missing cert and missing key paths

---

### AC5: No behavior change when files exist

**Given** valid cert and key file paths that exist  
**When** the caller invokes `requests.get(url, cert=valid_paths)`  
**Then** TLS handshake proceeds as before (no regression)

---

### AC6: CA-bundle lookup unchanged

**Given** the CA-bundle path is missing or invalid  
**When** the caller invokes a request  
**Then** the existing `OSError` raise for CA-bundle remains unchanged (out of scope)

---

## Low-Level Design

### Files Modified

- `src/requests/adapters.py`
- `tests/test_adapters.py` (new tests added)

---

### Exception Contract Changes

**Location:** `src/requests/adapters.py::HTTPAdapter.cert_verify`

#### Current implementation (lines to change)

1. **Missing cert path:**
   ```python
   raise OSError(
       f"Could not find the TLS certificate file, invalid path: {cert_file}"
   )
   ```

2. **Missing key path:**
   ```python
   raise OSError(
       f"Could not find the TLS certificate key file, invalid path: {key_file}"
   )
   ```

#### New implementation

1. **Missing cert path:**
   ```python
   raise FileNotFoundError(
       errno.ENOENT,
       "Could not find the TLS certificate file, invalid path",
       cert_file
   )
   ```
   - **First arg:** `errno.ENOENT` — standard errno for "No such file or directory"
   - **Second arg:** error message without the embedded path
   - **Third arg:** `cert_file` — sets the `filename` attribute on the exception

2. **Missing key path:**
   ```python
   raise FileNotFoundError(
       errno.ENOENT,
       "Could not find the TLS certificate key file, invalid path",
       key_file
   )
   ```
   - Same pattern: `errno.ENOENT`, message, `key_file` as `filename`

#### Unchanged

- **CA-bundle raise:** Any `OSError` raised for missing or invalid CA-bundle paths remains unchanged (out of scope)

#### Import required

Add at the top of `src/requests/adapters.py` if not already present:

```python
import errno
```

---

### Algorithm Notes

**File existence check flow in `cert_verify`:**

1. When `cert` is a string path:
   - Check `os.path.exists(cert)`
   - If False → raise `FileNotFoundError(errno.ENOENT, "Could not find the TLS certificate file, invalid path", cert)`

2. When `cert` is a tuple `(cert_path, key_path)`:
   - Check `os.path.exists(cert_path)` first
   - If False → raise `FileNotFoundError(errno.ENOENT, "Could not find the TLS certificate file, invalid path", cert_path)`
   - Then check `os.path.exists(key_path)`
   - If False → raise `FileNotFoundError(errno.ENOENT, "Could not find the TLS certificate key file, invalid path", key_path)`

**Backward compatibility guarantee:**

- `FileNotFoundError` is a builtin subclass of `OSError`
- Existing `except OSError` blocks continue to catch the new exception
- Code that checks `isinstance(exc, OSError)` remains valid
- Only code that checks `type(exc) is OSError` or matches exact message strings needs updating

---

### Interface Signatures

**Method signature (unchanged):**

```python
def cert_verify(self, conn, url, verify, cert):
    """
    Verify a SSL certificate. This method should not be called from user code.
    
    :param conn: The connection object (from urllib3)
    :param url: The request URL
    :param verify: Either a boolean, path to a CA bundle, or None
    :param cert: Either a path to a cert file, a (cert, key) tuple, or None
    """
```

**Exception contract (modified):**

**Before:**

```python
# HTTPAdapter.cert_verify
# Raises: OSError with embedded path in message, no filename attribute
```

**After:**

```python
# HTTPAdapter.cert_verify
# Raises: FileNotFoundError with errno.ENOENT, message, and filename attribute
```

---

## Tests

### New Unit Tests (in `tests/test_adapters.py`)

#### Test 1: Missing cert path raises FileNotFoundError

**Covers:** AC1

```python
def test_missing_cert_raises_file_not_found_error():
    """
    AC1: Missing cert path raises FileNotFoundError with filename attribute.
    """
    from requests.adapters import HTTPAdapter
    import pytest

    adapter = HTTPAdapter()
    missing_cert_path = "/tmp/does-not-exist-adlc-cert.pem"
    
    # Stand-in connection object (not used for file check)
    conn = None
    url = "https://example.com"
    verify = True

    with pytest.raises(FileNotFoundError) as exc_info:
        adapter.cert_verify(conn, url, verify, missing_cert_path)

    exc = exc_info.value
    assert isinstance(exc, FileNotFoundError)
    assert isinstance(exc, OSError)  # Backward compatibility
    assert exc.filename == missing_cert_path
    assert "TLS certificate file" in str(exc)
```

---

#### Test 2: Missing key path raises FileNotFoundError

**Covers:** AC2

```python
def test_missing_key_raises_file_not_found_error():
    """
    AC2: Missing key path raises FileNotFoundError with filename attribute.
    """
    from requests.adapters import HTTPAdapter
    import pytest
    import tempfile
    import os

    adapter = HTTPAdapter()
    # Create a temporary valid cert file
    with tempfile.NamedTemporaryFile(mode='w', suffix='.pem', delete=False) as cert_file:
        cert_file.write("dummy cert content")
        valid_cert_path = cert_file.name

    missing_key_path = "/tmp/does-not-exist-adlc-key.pem"
    
    # Stand-in connection object (not used for file check)
    conn = None
    url = "https://example.com"
    verify = True

    try:
        with pytest.raises(FileNotFoundError) as exc_info:
            adapter.cert_verify(conn, url, verify, (valid_cert_path, missing_key_path))

        exc = exc_info.value
        assert isinstance(exc, FileNotFoundError)
        assert isinstance(exc, OSError)  # Backward compatibility
        assert exc.filename == missing_key_path
        assert "TLS certificate key file" in str(exc)
    finally:
        os.unlink(valid_cert_path)
```

---

#### Test 3: OSError catch remains valid

**Covers:** AC1, AC2 (backward compatibility)

```python
def test_file_not_found_error_is_caught_by_oserror():
    """
    AC1/AC2: Confirm FileNotFoundError is caught by except OSError.
    """
    from requests.adapters import HTTPAdapter

    adapter = HTTPAdapter()
    missing_path = "/tmp/does-not-exist.pem"
    
    # Stand-in connection object (not used for file check)
    conn = None
    url = "https://example.com"
    verify = True

    caught_as_oserror = False
    try:
        adapter.cert_verify(conn, url, verify, missing_path)
    except OSError:
        caught_as_oserror = True
    except Exception:
        pass

    assert caught_as_oserror, "FileNotFoundError should be caught by except OSError"
```

---

### Existing Test Updates

**Approach:**

1. Run `python -m pytest tests/test_adapters.py` after implementation
2. Identify any tests that check `type(exc) is OSError` or match exact message strings
3. Update those tests to:
   - Accept `FileNotFoundError` as well as `OSError`, OR
   - Match the new message format (without the path embedded, since `filename` is separate)

**Example update (if a test like this exists):**

```python
# Before:
with pytest.raises(OSError, match="Could not find the TLS certificate file, invalid path: /tmp/missing.pem"):
    ...

# After:
with pytest.raises(FileNotFoundError) as exc_info:
    ...
assert exc_info.value.filename == "/tmp/missing.pem"
assert "Could not find the TLS certificate file" in str(exc_info.value)
```

**Covers:** AC3

---

## Verification Plan

### Unit-level verification

1. **AC1:** Run `test_missing_cert_raises_file_not_found_error` → green
2. **AC2:** Run `test_missing_key_raises_file_not_found_error` → green
3. **AC1/AC2:** Run `test_file_not_found_error_is_caught_by_oserror` → green

### Integration-level verification

4. **AC3:** Run `python -m pytest tests/test_adapters.py` → all tests green (or updated)
5. **AC4:** Run `python -m pytest tests/test_adapters.py -k cert` → all cert-related tests green
6. **AC5:** Run existing TLS tests with valid cert paths → no regressions
7. **AC6:** Confirm CA-bundle tests pass without modification → no changes to CA-bundle logic

### Full test suite

8. Run `python -m pytest tests` → all tests green

---

## Implementation Notes

### PR Review Comments Addressed

**Comment 6441627532 & 4388544199:**

1. **`src/requests/adapters.py`** — Updated `HTTPAdapter.cert_verify` to raise `FileNotFoundError(errno.ENOENT, message, path)` for both missing certificate and missing key paths, ensuring the exception's `filename` attribute is set correctly per the LLD above.

2. **`tests/test_adapters.py`** — Added three unit tests covering missing certificate path (Test 1), missing key path (Test 2), and backward compatibility with `OSError` catch (Test 3), verifying exception type and `filename` attribute.

3. **Unrelated changes removed** — No `.gitignore` or other out-of-scope changes are included in this story; the implementation is strictly limited to the two files above per PER-63 scope.

---

## Open Questions

(none)

---

## Notes

- **Repo:** `demo-test` (https://github.com/pavithrachandran70/requests-demo)
- **Base branch:** `main`
- **Lint:** `python -m compileall -q .`
- **Test command:** `python -m pytest tests`
- **Out of scope:** CA-bundle logic, TLS handshake when files exist, other modules or repos, unrelated `.gitignore` changes
- **Backward compatibility guarantee:** `FileNotFoundError` is a subclass of `OSError`; existing exception handling remains valid
- **Test count:** 3 new unit tests match the 3 verification approaches in breakdown (missing cert, missing key, backward compatibility)
