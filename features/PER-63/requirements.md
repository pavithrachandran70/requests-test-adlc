# Requirements — PER-63

**Story:** Raise FileNotFoundError when a client cert or key path is missing

**Epic:** (none — backlog story)

**Covers:** (no spec capabilities)

---

## Problem Statement

`HTTPAdapter.cert_verify` in `src/requests/adapters.py` raises `OSError` when a cert or key path does not exist. Callers cannot distinguish a missing file from other OS-level errors, and the exception lacks a `filename` attribute.

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

## Verification

1. **Unit test:** Missing cert path → `FileNotFoundError`, `filename` set, `OSError` catches it
2. **Unit test:** Missing key path → `FileNotFoundError`, `filename` set, `OSError` catches it
3. **Integration check:** Existing TLS tests pass or are updated only for type/message assertions
4. **Regression check:** Valid cert/key paths behave as before

---

## Open Questions

(none)

---

## Notes

- Scope: `src/requests/adapters.py` only — the two raises in `cert_verify` for missing cert and missing key paths
- Out of scope: CA-bundle raise, TLS handshake logic, other repos, framework/stack choices
- Backward compatibility: `FileNotFoundError` is a subclass of `OSError`; existing `except OSError` blocks remain valid
