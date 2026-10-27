# Plan — PER-63 (v4)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard

Single-repo exception change: two raises updated for FileNotFoundError with filename attribute; designer must regenerate LLD with corrected test snippets per reviewer feedback.

## What changed

Designer reruns to fix test snippets: Test 1–3 now call cert_verify with all four arguments (conn, url, verify, cert) and do not open a TLS connection.

- Rerun: designer, test_architect
- Kept (results reused): analyst

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#5 `[hub#5:issue-6441178431]`: 1. On the test snippets (Test 1–3)

HTTPAdapter.cert_verify is cert_verify(self, conn, url, verify, cert). These samples call adapter.cert_verify(None, missing_cert_path, None), so the missing path is passed as url and cert is never set. Rewrite Test 1, Test 2, and Test 3 to call cert_verify with a stand-in connection, an https URL, verify=True, and cert set to the missing path or the (cert_path, missing_key_path) tuple. Assert exc.filename is that path. The tests must not open a TLS connection.
  - addressed by: designer rerun with instructions to correct Test 1, Test 2, and Test 3 signatures and assert exc.filename without opening a TLS connection

## Steps

1. [x] **analyst (inline_story)** — Ticket scope is concrete: change two specific raises (cert and key), preserve the third (CA-bundle), set filename attribute, and maintain OSError compatibility.
2. [x] **designer (lld)** — Caller-visible exception contract: must specify exact raise statements for missing cert and key paths, how filename is set, and that CA-bundle raise is unchanged.
   - focus: Rewrite Test 1, Test 2, and Test 3 to call cert_verify(conn, url, verify, cert) with a stand-in connection object, an https URL, verify=True, and cert set to the missing path or (cert_path, missing_key_path) tuple. Assert exc.filename is that path. Tests must not open a TLS connection.
3. [x] **test_architect (tests)** — Acceptance criteria specify multiple edge cases: missing cert, missing key, filename attribute, backward compatibility with OSError catch, and message format for existing tests.
4. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Fixed execution step.
6. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
7. [ ] **open_pr** _(auto)_ — Fixed execution step.
8. [ ] **pr_review** _(auto)_ — Fixed execution step.
9. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Ticket scope and acceptance criteria are unambiguous and purely technical.
- **architect** — No new components, cross-repo dependencies, or schema changes; the exception contract is handled by the designer.
- **scrum_master** — Single story in one repo; no breakdown needed.
- **critic** — Only one spec artifact (the story with tests); nothing to cross-check.
