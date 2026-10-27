# Breakdown — Keep a caller-supplied SSLContext when opening an HTTPS connection

**Ticket:** PER-57  
**Type:** Story  
**Repo:** demo-test  
**Status:** Planned

---

## Overview

This breakdown converts the requirements and HLD into one implementable story. The story modifies pool-key construction in `src/requests/adapters.py` so a custom `ssl_context` passed to `HTTPAdapter.init_poolmanager` is not overridden by `cert_reqs` logic, and adds unit tests to verify both custom-context and default-verification paths.

---

## Dependencies

**Prerequisites:**

- urllib3 library is already pinned in `pyproject.toml` (verify version compatibility during LLD).
- Existing adapter tests in `tests/test_adapters.py` pass.

**Blocked by:** None.

**Blocking:** None.

---

## Ordered Story List

### Story 1: Preserve custom SSLContext in HTTPAdapter pool-key construction

**Repo:** `demo-test`

**Description:**

Modify the pool-key construction logic in `src/requests/adapters.py` so that when a custom `ssl_context` is supplied to `HTTPAdapter.init_poolmanager`, its `verify_mode` is preserved in the connection pool. A normal request with `verify=True` and no custom context still sets `cert_reqs` to `CERT_REQUIRED`.

**Covers:**

- **AC-1:** After `init_poolmanager(ssl_context=ctx)` where `ctx.verify_mode` is `ssl.CERT_NONE`, the pool selected for `https://example.com/` still carries that same context, and `cert_reqs` does not force `CERT_REQUIRED` onto it.
- **AC-2:** A request with `verify=True` and no custom context still sets `cert_reqs` to `CERT_REQUIRED`.
- **AC-3:** `python -m pytest tests` passes for the adapter tests.

**Acceptance Criteria:**

1. **AC-1: Custom SSLContext preserved**  
   **Given** an `HTTPAdapter` initialized with `init_poolmanager(ssl_context=ctx)` where `ctx.verify_mode` is `ssl.CERT_NONE`  
   **When** the adapter selects a pool for `https://example.com/`  
   **Then** the pool carries that same context, and `cert_reqs` does not force `CERT_REQUIRED` onto it.

2. **AC-2: Default verify=True enforced**  
   **Given** a request with `verify=True` and no custom context  
   **When** the adapter builds a pool key  
   **Then** `cert_reqs` is set to `CERT_REQUIRED`.

3. **AC-3: Adapter tests pass**  
   **Given** the adapter logic is changed  
   **When** `python -m pytest tests` runs for the adapter tests  
   **Then** all tests pass.

**After:** None.

**Verification Approach:**

- **Test 1 (AC-1):** Unit test `test_custom_sslcontext_preserves_verify_mode` in `tests/test_adapters.py` initializes an `HTTPAdapter` with a custom `ssl.SSLContext` (e.g. `verify_mode=ssl.CERT_NONE`). Retrieves the pool for `https://example.com/` and asserts the pool's `cert_reqs` does not force `CERT_REQUIRED`. (1 test case)
- **Test 2 (AC-2):** Unit test `test_default_verify_enforces_cert_required` in `tests/test_adapters.py` initializes an `HTTPAdapter` with no custom context, retrieves the pool with `verify=True`, and asserts the pool's `cert_reqs` is `CERT_REQUIRED`. (1 test case)
- **Test 3 (AC-3):** Run `python -m pytest tests/test_adapters.py` and confirm all tests pass. (1 suite run)

**Total test count:** 2 new test cases + 1 suite run.

**Allowed Paths:**

- `src/requests/adapters.py` (modify `_urllib3_request_context` or `get_connection_with_tls_context`)
- `tests/test_adapters.py` (add `test_custom_sslcontext_preserves_verify_mode` and `test_default_verify_enforces_cert_required`)

**Technical Notes:**

- **Custom context detection:** Retrieve the `ssl_context` from the pool manager's `connection_pool_kw` attribute (or equivalent). Check if it is non-`None`.
- **Conditional logic:** In `_urllib3_request_context`, skip the `cert_reqs` override when a custom context is present. When no custom context is present and `verify=True`, set `cert_reqs` to `resolve_cert_reqs(None)` (which is `CERT_REQUIRED`).
- **Test isolation:** The unit tests must mock or inspect the pool manager's internal state without opening a real socket connection.
- **Scope constraint:** Do not change `verify=False`, CA-bundle paths, or client-cert file checks.

**Risks:**

- **R-1: urllib3 version compatibility** — The pool manager API or `cert_reqs` resolution may vary across urllib3 versions. Verify the change against the pinned version in `pyproject.toml`. (Likelihood: Low, Severity: High)
- **R-2: Existing adapter tests break** — The pool-key construction change may alter pool-manager behavior in ways that break existing tests. Run the full test suite during Dev. (Likelihood: Medium, Severity: Medium)
- **R-3: Custom context detection is fragile** — If the pool manager does not expose the `ssl_context` reference via `connection_pool_kw`, the detection logic will fail. Inspect the urllib3 `PoolManager` source during LLD. (Likelihood: Low, Severity: High)
- **R-4: Hostname check preservation unclear** — The ticket's expected behavior mentions "hostname checks", but the ACs do not test them. Clarify during LLD or tests step whether hostname checks are controlled by the pool-key construction or by the `ssl_context`. (Likelihood: Medium, Severity: Medium)

**Open Questions:**

1. **Q1:** Does the urllib3 `PoolManager` expose the `ssl_context` via `connection_pool_kw` or another attribute?  
   **Owner:** LLD step (designer)  
   **Deadline:** Before implementation.

2. **Q2:** Are hostname checks (`check_hostname`) controlled by the `ssl_context` or by pool-key parameters?  
   **Owner:** LLD step (designer) or Test Architect  
   **Deadline:** Before test design.

3. **Q3:** Does the existing test suite mock the pool manager in a way that allows inspection of `cert_reqs` without opening a socket?  
   **Owner:** LLD step (designer)  
   **Deadline:** Before test implementation.

---

## Consistency Check

- **Ticket ↔ Requirements ↔ HLD ↔ Breakdown:**
  - Same AC count: 3 ACs in ticket, requirements, HLD, and breakdown.
  - Same AC wording: "Custom SSLContext preserved", "Default verify=True enforced", "Adapter tests pass".
  - Same verification count: 2 new test cases + 1 suite run in HLD Verification section and breakdown Verification Approach.
  - Same file paths: `src/requests/adapters.py` and `tests/test_adapters.py` in HLD Components and breakdown Allowed Paths.

- **Acceptance criteria alignment:**
  - No new epic ACs invented. ACs are copied verbatim from the ticket and requirements.
  - Edge cases (e.g. hostname checks) are noted as open questions, not new ACs.

- **Test count alignment:**
  - Breakdown lists 2 new test cases; HLD Verification section lists 2 test cases.
  - Breakdown lists 1 suite run (AC-3); HLD Verification section confirms "run pytest and confirm all tests pass".

- **File paths alignment:**
  - Breakdown Allowed Paths cites `src/requests/adapters.py` and `tests/test_adapters.py`.
  - HLD Components and File Touch List cite the same paths.
  - Paths match the repo tree (`demo-test`).

- **Stack alignment:**
  - Breakdown uses Python, requests, urllib3, pytest (no invented language or framework).
  - Repo context confirms Python stack and `python -m pytest` test command.

---

## Notes

- **No new ACs:** All acceptance criteria are derived from the ticket and requirements. No new epic ACs were invented.
- **Concrete paths:** All file paths are from the repo tree (`src/requests/adapters.py`, `tests/test_adapters.py`).
- **Test count:** 2 new test cases + 1 suite run (matches HLD Verification section).
- **Scope:** Repo `demo-test` only. No changes to `verify=False`, CA-bundle paths, or client-cert file checks.
- **Deferred:** Hostname check preservation (open question Q2).

---

**End of Breakdown**
