# High-Level Design — Preserve caller-supplied SSLContext verify_mode in HTTPAdapter connection pools

**Ticket:** PER-57  
**Type:** Story  
**Repo:** demo-test  
**Scope:** Modify pool-key construction in `src/requests/adapters.py` so a custom `ssl_context` passed to `HTTPAdapter.init_poolmanager` is not overridden by `cert_reqs` logic.

---

## Overview

When a caller supplies a custom `ssl.SSLContext` to `HTTPAdapter.init_poolmanager(ssl_context=ctx)`, urllib3 stores the context on the pool manager. However, the pool-key construction in `get_connection_with_tls_context` (via `_urllib3_request_context`) always sets `cert_reqs` to `CERT_REQUIRED` when `verify=True`, and urllib3 then executes `context.verify_mode = resolve_cert_reqs(cert_reqs)` while connecting. The caller's context is rewritten.

**Behavioral outcome:** After this change, a custom context's `verify_mode` (e.g. `ssl.CERT_NONE`) is preserved in the pool. A normal request with `verify=True` and no custom context still sets `cert_reqs` to `CERT_REQUIRED`. Existing adapter tests continue to pass.

---

## Components

### 1. `src/requests/adapters.py`

**Existing:**

- `HTTPAdapter.init_poolmanager(...)` accepts an optional `ssl_context` parameter and stores it on the pool manager.
- `get_connection_with_tls_context(...)` builds a pool key through `_urllib3_request_context(...)`.
- `_urllib3_request_context(...)` always sets `cert_reqs` to `resolve_cert_reqs(None)` (which is `CERT_REQUIRED`) when `verify=True`.

**Change:**

- Detect whether a custom `ssl_context` was supplied to the pool manager.
- When a custom context is present, skip the `cert_reqs` override in the pool-key construction.
- When no custom context is present, preserve the existing behavior: set `cert_reqs` to `CERT_REQUIRED` for `verify=True`.

**Rationale:** The pool-key construction must distinguish between a caller-supplied context and the default certificate-verification path. The pool manager already holds the `ssl_context` reference; the pool-key logic must consult it.

---

### 2. Unit test in `tests/test_adapters.py`

**Purpose:** Verify both the custom-context path (AC-1) and the default path (AC-2) without opening a socket.

**Structure:**

- Test case 1: Initialize an `HTTPAdapter` with a custom `ssl.SSLContext` (e.g. `verify_mode=ssl.CERT_NONE`). Call the pool-manager initialization and retrieve the pool for `https://example.com/`. Assert the pool's `cert_reqs` does not force `CERT_REQUIRED`.
- Test case 2: Initialize an `HTTPAdapter` with no custom context. Call the pool-manager initialization with `verify=True`. Assert the pool's `cert_reqs` is `CERT_REQUIRED`.

**Note:** The test must mock or patch the pool manager to avoid opening a real socket connection.

---

## Contracts

### Adapter pool-key construction

**Input:**

- `verify` (boolean): whether certificate verification is requested.
- `ssl_context` (optional `ssl.SSLContext`): the custom context supplied by the caller, or `None`.

**Output:**

- Pool key with `cert_reqs` set to:
  - The custom context's `verify_mode` when a custom context is present.
  - `CERT_REQUIRED` when `verify=True` and no custom context is present.
  - `None` when `verify=False`.

**Invariant:** A custom context's `verify_mode` is never overridden by the pool-key construction logic.

---

### Test interface

**Input:**

- An `HTTPAdapter` instance initialized with or without a custom `ssl.SSLContext`.

**Output:**

- Assertions on the pool's `cert_reqs` value.

**Invariant:** The test does not open a network socket.

---

## Risks

### R-1: urllib3 version compatibility

**Impact:** The pool-key construction logic and `resolve_cert_reqs` function may vary across urllib3 versions.

**Mitigation:** Verify the change against the urllib3 version pinned in `pyproject.toml`. If the pool-manager API or `cert_reqs` resolution changes, update the adapter logic to match.

**Status:** Open. The HLD assumes the current urllib3 API.

---

### R-2: Existing adapter tests break

**Impact:** The pool-key construction change may alter the pool-key format or pool-manager behavior in ways that break existing tests.

**Mitigation:** Run `python -m pytest tests/test_adapters.py` after the change. If tests fail, investigate whether the failure is due to a genuine regression or a test assumption that needs updating.

**Status:** Open. AC-3 requires all adapter tests to pass.

---

### R-3: Custom context detection is fragile

**Impact:** If the pool manager does not expose the `ssl_context` reference, or if the detection logic is incorrect, the fix may not work.

**Mitigation:** The pool manager already holds the `ssl_context` reference (passed to `init_poolmanager`). The detection logic must retrieve it from the pool manager before building the pool key. If the reference is not accessible, the fix cannot proceed.

**Status:** Open. The LLD must confirm the pool manager exposes the `ssl_context` reference.

---

## Architecture Decisions

### AD-1: Detect custom SSLContext by presence

**Binds:** The pool-key construction logic in `_urllib3_request_context` or `get_connection_with_tls_context`.

**Prevents:** Re-inventing a new detection mechanism or checking the context's `verify_mode` directly.

**Rule:** A custom `ssl_context` is present when the pool manager's `connection_pool_kw` or pool-manager instance holds a non-`None` `ssl_context` reference. The detection logic retrieves this reference and checks whether it is `None`.

**Rationale:** The pool manager already holds the `ssl_context` reference. Checking its presence is simpler and more robust than inspecting the context's `verify_mode` or comparing it to a default context.

**Status:** `[ADOPTED]`

---

### AD-2: Skip cert_reqs override when custom context is present

**Binds:** The `cert_reqs` assignment in `_urllib3_request_context`.

**Prevents:** Overriding a custom context's `verify_mode` with `CERT_REQUIRED`.

**Rule:** When a custom `ssl_context` is present (per AD-1), do not set `cert_reqs` in the pool-key construction. When no custom context is present and `verify=True`, set `cert_reqs` to `resolve_cert_reqs(None)` (which is `CERT_REQUIRED`). When `verify=False`, set `cert_reqs` to `None`.

**Rationale:** The caller's context already carries the intended `verify_mode`. Setting `cert_reqs` in the pool key forces urllib3 to override the context's `verify_mode`, which defeats the purpose of supplying a custom context. The default path (no custom context, `verify=True`) must still enforce certificate verification.

**Status:** `[ADOPTED]`

---

### AD-3: Preserve default verify=True behavior

**Binds:** The pool-key construction logic for requests without a custom context.

**Prevents:** Breaking the default certificate-verification behavior.

**Rule:** When no custom `ssl_context` is present and `verify=True`, the pool-key construction sets `cert_reqs` to `CERT_REQUIRED`. This is the existing behavior and must remain unchanged.

**Rationale:** The default path is well-tested and widely relied upon. The fix must not alter it.

**Status:** `[ADOPTED]`

---

## Data and API Notes

### Per-ticket schema/API notes

**Ticket:** Preserve caller-supplied SSLContext verify_mode in HTTPAdapter connection pools (PER-57)

- **API change:** None. The public API surface (`HTTPAdapter.init_poolmanager(ssl_context=...)`) remains unchanged.
- **Internal change:** The pool-key construction logic in `_urllib3_request_context` or `get_connection_with_tls_context` gains a conditional branch that detects a custom `ssl_context` and skips the `cert_reqs` override when present.
- **Data flow:**
  1. Caller passes `ssl_context=ctx` to `HTTPAdapter.init_poolmanager(...)`.
  2. Pool manager stores the context.
  3. On the next request, `get_connection_with_tls_context` retrieves the context from the pool manager.
  4. If the context is non-`None`, `_urllib3_request_context` does not set `cert_reqs`.
  5. If the context is `None` and `verify=True`, `_urllib3_request_context` sets `cert_reqs` to `CERT_REQUIRED`.
  6. urllib3 uses the pool key to select or create a connection pool. When `cert_reqs` is not set, the context's `verify_mode` is preserved.

---

## Deferred

- **CA-bundle paths:** Out of scope for this ticket.
- **Client-cert file checks:** Out of scope for this ticket.
- **`verify=False` behavior:** Out of scope for this ticket. The fix only addresses the `verify=True` + custom context case.
- **Hostname check preservation:** The ticket's expected behavior mentions hostname checks, but the AC does not test them. The LLD or tests may need to clarify whether hostname checks are controlled by the `ssl_context` or by other pool-key parameters.

---

## Diagrams

### Pool-key construction flow (before)

```mermaid
flowchart TD
    A[HTTPAdapter.init_poolmanager(ssl_context=ctx)] --> B[Pool manager stores ctx]
    C[Request with verify=True] --> D[get_connection_with_tls_context]
    D --> E[_urllib3_request_context]
    E --> F[cert_reqs = CERT_REQUIRED]
    F --> G[urllib3 overrides ctx.verify_mode]
    G --> H[Connection enforces CERT_REQUIRED]
```

### Pool-key construction flow (after)

```mermaid
flowchart TD
    A[HTTPAdapter.init_poolmanager(ssl_context=ctx)] --> B[Pool manager stores ctx]
    C[Request with verify=True] --> D[get_connection_with_tls_context]
    D --> E[Retrieve ctx from pool manager]
    E --> F{ctx is non-None?}
    F -->|Yes| G[Skip cert_reqs override]
    F -->|No| H[cert_reqs = CERT_REQUIRED]
    G --> I[urllib3 uses ctx.verify_mode]
    H --> J[urllib3 enforces CERT_REQUIRED]
```

---

## Open Questions

None at this time. The LLD must confirm the pool manager exposes the `ssl_context` reference and detail the conditional branches.

---

## Verification

- **AC-1:** Unit test initializes an `HTTPAdapter` with a custom `ssl.SSLContext` (e.g. `verify_mode=ssl.CERT_NONE`). Asserts the pool's `cert_reqs` does not force `CERT_REQUIRED`.
- **AC-2:** Unit test initializes an `HTTPAdapter` with no custom context. Asserts the pool's `cert_reqs` is `CERT_REQUIRED` when `verify=True`.
- **AC-3:** Run `python -m pytest tests/test_adapters.py` and confirm all tests pass.

---

**End of HLD**
