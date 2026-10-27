# Plan — PER-63 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard

Replace two OSError raises with FileNotFoundError (preserving OSError as parent), set filename attribute, and verify backward compatibility; test_architect ensures edge cases and message format are covered.

## What changed

Analyst instructions updated to specify which two OSError raises to change (cert and key, not CA-bundle), use FileNotFoundError constructor with filename parameter, preserve str(exc) format, and state OSError subclass relationship.


## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#4 `[hub#4:issue-6440239992]`: Analyst scope is too small:
 
The analyst step says "add one unit test" and "replace OSError with FileNotFoundError." `cert_verify` has three `OSError` raises. Only the certificate-file raise and the key-file raise should change. The CA-bundle raise (`Could not find a suitable TLS CA certificate bundle`) stays `OSError`. `FileNotFoundError(message)` does not set `.filename`. The story needs the constructor that sets `filename` to the path that was passed in, keeps `FileNotFoundError` a subclass of `OSError`, and states what `str(exc)` should be, because `tests/test_requests.py::test_invalid_ssl_certificate_files` asserts the current message.
  - addressed by: Analyst rerun with instructions to change only cert and key raises, keep CA-bundle as OSError, set filename attribute via constructor, specify message format for test compatibility, and confirm FileNotFoundError remains OSError subclass.

## Steps

1. [ ] **analyst (inline_story)** _(rerun)_ — Ticket scope is concrete: change two specific raises (cert and key), preserve the third (CA-bundle), set filename attribute, and maintain OSError compatibility.
   - focus: Replace only the certificate-file and key-file OSError raises with FileNotFoundError. Keep the CA-bundle raise as OSError. Use the FileNotFoundError constructor that sets the filename attribute to the path passed in. Specify the exact str(exc) format for test_invalid_ssl_certificate_files compatibility. State that FileNotFoundError remains a subclass of OSError.
2. [ ] **test_architect (tests)** _(rerun)_ — Acceptance criteria specify multiple edge cases: missing cert, missing key, filename attribute, backward compatibility with OSError catch, and message format for existing tests.
3. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
4. [ ] **dev** _(auto)_ — Fixed execution step.
5. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
6. [ ] **open_pr** _(auto)_ — Fixed execution step.
7. [ ] **pr_review** _(auto)_ — Fixed execution step.
8. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Ticket scope and acceptance criteria are unambiguous and purely technical.
- **architect** — No API contract, schema, or cross-repo changes; the change is localized to exception handling in one function.
- **scrum_master** — Single story in one repo; no breakdown needed.
- **designer** — Implementation is straightforward: replace two OSError raises with FileNotFoundError(message, filename=path).
- **critic** — Only one spec artifact (the story with tests); nothing to cross-check.
